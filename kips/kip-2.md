---
kip: 2
title: Allow for easier withdrawals and re-using payouts (replaces KIP-1)
authors: Ramesh Nair
status: Draft
created: 2019-11-29
---

## Simple Summary

**Note: This KIP supersedes [KIP-1](https://github.com/wearekickback/KIPs/blob/master/kips/kip-1.md)**

This change removes the cooling period (thus allowing users to withdraw any time) and minimizes the gas cost for user's to withdraw their payouts across multiple events. It also makes it possible to reuse a payout as a deposit for a new event without having to withdraw it first.

## Motivation

Our current setup requires the user to manually withdraw their ETH payout from an event contract once the event has been finalized, but within a set *cooling period* (usually 1 week) after the event ends, after which withdrawal is no longer possible and the remaining ETH can be taken by the event organizer.

Although very few people have disagreed or complained about the above model, in practice
In practice we find that many users either a) are unaware they need to withdraw, b) forget to withdraw, c) are unable to withdraw within the cooling period, or d) would prefer auto-withdrawal. Withdrawal also costs gas, which cuts into the user's payout, plus they already had to pay gas when RSVP'ing for an event.

## Specification

We introduce 2 new contracts in addition to the existing event contract:

- `UserPot` - a single instance of this holds all ETH for all users across all events.
- `ACL` - a single instance of this holds the list of Kickback admins who can upgrade the `UserPot`.

When a user RSVP's to an event this is the new flow:

1. User calls `register()` on the event's deployed contract with enough ETH
2. The event's contract forwards the ETH to the global `UserPot` contract
2a. _In the case of ERC20 tokens the `UserPot` will transfer the requisite amount from the user's account, thus the user needs to have previously approved the `UserPot` contract_.
3. The `UserPot` adds the event address to the list of events the user is attending
4. `UserPot` also calculates the amount of ETH/tokens withdrawable by the user (based on past event payouts owed to them) and records this number

When a user wishes to withdraw their ETH:

1. The user send a transaction to `UserPot` asking to withdraw
2. `UserPot` calculates all the ETH/tokens owed to the user based on past attendance
3. `UserPot` updates the storage data for the user, moving events which have ended from the user's event list, and setting their withdrawable balance to 0
4. `UserPot` sends the total ETH/tokens back to the user

## Rationale

The above design was arrived at after considering various alternatives as well as other desirable features that users have asked for. Here are the key points:

- **Singleton user pot is necessary** - because our system redistributes ETH/tokens between users (i.e. if you don't show up your ETH/tokens get _given_ to other users) it's necessary to have all ETH/tokens for a given event stored in one place, as it would be too expensive gas-wise and time-wise to send ETH/tokens amongst the participants of an event. This is why we can't have a `UserPot` instance per user.

- **Reuse of deposit** - because we have a single user pot instance, we can easily reuse the payout from an old event as a deposit for a new event. In the sample implementation below, this is actually what happens when a user RSVP's. Thus, users do not need to withdraw their ETH/tokens and send it back in in order to attend future events.

- **Easier overview of user balance** - because all ETH/tokens are in one place we can easily give the user an on-chain overview of their total deposits and payouts on Kickback.

- **Upgrade-able contracts** - by making the user pot upgradeable (with admin approval) we allow for flexibility in future and for bug fixing capability.

- **Configurable event incentives** - one of the nice things about this architecture is that the `UserPot` is only responsible for summing up deposits and payouts - actual calculation of what the payout should be is done by the `Event` contract. If we wished to test out new strategies (e.g. sweepstakes) or try using a single `Event` instance for multiple events we could easily do so.

- **Potential revenue opportunity** - with the rise of tools and projects such as rDAI and the DAI savings rate there is increasing opportunity of earning interest from staked assets. By changing the structure to hold our customers money it will increase the overall revenue potential when we integrate with such features in future.

## Risks and and trade-offs

Although gas costs for RSVP'ing have gone up, I think this is justifiable given the improvements in overall user experience, plus the additional flexibility we gain over our current model.

The events emitted by the event contract will all stay the same - meaning our backend doesn't need to change enormously for this. However we will no longer show when a user has withdrawn from an event - there's no need to since withdrawals across events are now batched.

Storing all ETH/tokens into a single contract may seem risky, but we can negate this with good testing and auditing of the contract, and by managing admin access using multisig accounts with op-sec supervision.

## Implementation

**EternalStorage**

The base class which specifies an extendable storage system that allows for easy upgrades.

```solidity
/**
 * Base storage class.
 */
contract EternalStorage {
  // scalars
  mapping(string => address) dataAddress;
  mapping(string => string) dataString;
  mapping(string => bytes32) dataBytes32;
  mapping(string => int256) dataInt256;
  mapping(string => uint256) dataUint256;
  mapping(string => bool) dataBool;
  // arrays
  mapping(string => address[]) dataManyAddresses;
  mapping(string => bytes32[]) dataManyBytes32s;
  mapping(string => int256[]) dataManyInt256;
  mapping(string => uint256[]) dataManyUint256;
  mapping(string => bool[]) dataManyBool;
}
```

**Proxy**

Base class for all upgradeable contracts. Upgradeability is based on the [OpenZepellin Proxy + EternalStorage](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_eternal_storage) pattern.

```solidity
/**
 * Upgradeable contracts base class
 */
contract Proxy is EternalStorage {
  /**
  * @dev This event will be emitted every time the implementation gets upgraded
  * @param implementation representing the address of the upgraded implementation
  */
  event Upgraded(address indexed implementation);

  /**
   * Constructor.
   */
  constructor (address _implementation) public {
    require(_implementation != address(0), 'implementation must be valid');
    dataAddress["implementation"] = _implementation;
  }

  /**
  * @dev Get the address of the implementation where every call will be delegated.
  * @return address of the implementation to which it will be delegated
  */
  function getImplementation() public view returns (address) {
    return dataAddress["implementation"];
  }

  /**
   * @dev Point to a new implementation.
   * This is internal so that descendants can control access to this in custom ways.
   */
  function setImplementation(address _implementation) internal {
    require(_implementation != address(0), 'implementation must be valid');
    require(_implementation != dataAddress["implementation"], 'already this implementation');

    dataAddress["implementation"] = _implementation;

    emit Upgraded(_implementation);
  }

  /**
  * @dev Fallback function allowing to perform a delegatecall to the given implementation.
  * This function will return whatever the implementation call returns
  */
  function () payable external {
    address _impl = getImplementation();
    require(_impl != address(0), 'implementation not set');

    assembly {
      let ptr := mload(0x40)
      calldatacopy(ptr, 0, calldatasize)
      let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
      let size := returndatasize
      returndatacopy(ptr, 0, size)
      switch result
      case 0 { revert(ptr, size) }
      default { return(ptr, size) }
    }
  }
}
```

**IACL, ACL, AccessControl**

Contracts for handling access control.

At present the `ACL` stores the list of system admins and provides mechanisms for updating that list. The `AccessControl` class is how other contracts will make use of the ACL. Admins can add and remove other admins, but there will always be atleast 1 admin in the system.

The ACL address is set once during construction but this could easily be changed to using an ENS lookup if we want to make the ACL itself replaceable in future.

_Note: I just copied the ACL code from an existing project. There could be a simpler solution for admin management if this seems too complex._

```solidity
/**
 * Interface for ACL.
 */
interface IACL {
  // admin
  function isAdmin(address _addr) external view returns (bool);
  function proposeNewAdmin(address _addr) external;
  function cancelNewAdminProposal(address _addr) external;
  function acceptAdminRole() external;
  function removeAdmin(address _addr) external;
}

/**
 * Access-control list
 */
contract ACL is IACL {
  mapping (address => bool) public admins;
  mapping (address => bool) public pendingAdmins;
  uint256 public numAdmins;

  event AdminProposed(address indexed addr);
  event AdminProposalCancelled(address indexed addr);
  event AdminProposalAccepted(address indexed addr);
  event AdminRemoved(address indexed addr);

  modifier assertIsAdmin () {
    require(admins[msg.sender], 'unauthorized - must be admin');
    _;
  }

  constructor () public {
    admins[msg.sender] = true;
    numAdmins = 1;
  }

  function isAdmin(address _addr) public view returns (bool) {
    return admins[_addr];
  }

  function proposeNewAdmin(address _addr) public assertIsAdmin {
    require(!admins[_addr], 'already an admin');
    require(!pendingAdmins[_addr], 'already proposed as an admin');
    pendingAdmins[_addr] = true;
    emit AdminProposed(_addr);
  }

  function cancelNewAdminProposal(address _addr) public assertIsAdmin {
    require(pendingAdmins[_addr], 'not proposed as an admin');
    pendingAdmins[_addr] = false;
    emit AdminProposalCancelled(_addr);
  }

  function acceptAdminRole() public {
    require(pendingAdmins[msg.sender], 'not proposed as an admin');
    pendingAdmins[msg.sender] = false;
    admins[msg.sender] = true;
    numAdmins++;
    emit AdminProposalAccepted(msg.sender);
  }

  function removeAdmin(address _addr) public assertIsAdmin {
    require(1 < numAdmins, 'cannot remove last admin');
    require(_addr != msg.sender, 'cannot remove oneself');
    require(admins[_addr], 'not an admin');
    admins[_addr] = false;
    numAdmins--;
    emit AdminRemoved(_addr);
  }
}

/**
 * Access control helper base class.
 */
contract AccessControl is EternalStorage {
  modifier assertIsAdmin () {
    require(acl().isAdmin(msg.sender), 'must be admin');
    _;
  }

  constructor (address _acl) public {
    dataAddress["acl"] = _acl;
  }

  function acl () internal view returns (IACL) {
    return IACL(dataAddress["acl"]);
  }
}
```

**UserPot, IUserPot, UserPotImpl**

The user pot.

Note that `UserPot` doesn't implement `IUserPot` itself since it's going to be a proxy contract which delegates all calls to a `UserPotImpl` instance. Also note that `upgrade()` can only be called by an administrator.

```solidity
/**
 * Interface for ERC20
 */
interface IERC20 {
  function balanceOf(address account) external view returns (uint256);
  function transfer(address recipient, uint256 amount) external returns (bool);
  function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
}


/**
 * Interface for User pot
 */
interface IUserPot {
  function deposit(address _event, address _user, uint256 _deposit) external payable;
}


/**
 * User pot instance
 */
contract UserPot is AccessControl, Proxy {
  constructor (address _acl, address _userPotImpl) AccessControl(_acl) Proxy(_userPotImpl) public {}

  function upgrade (address _implementation) public assertIsAdmin {
    setImplementation(_implementation);
  }
}

/**
 * User pot implementation
 */
contract UserPotImpl is EternalStorage, AccessControl, IUserPot {
  function deposit(address _event, address _user) public payable {
    IEvent _event = IEvent(msg.sender);
    address token = _event.getToken();
    uint256 deposit = _event.getDeposit(_user);
    uint256 bal = calculatePayout(_user, _token);

    if (token != address(0)) {
      if (bal < deposit) {
        IERC20 t = IERC20(token);
        require(t.balanceOf(_user) + bal >= deposit, 'you need to have more tokens to register for event');
        t.transferFrom(_user, address(this), deposit - bal);
        bal = 0;
      } else {
        bal = bal - deposit;
      }
    } else {
      bal += msg.value;
      require(bal >= deposit, 'you need to pay more to register for event');
      bal = bal - deposit;
    }

    _updateUserData(_user, msg.sender, token, bal);
  }

  function withdraw(address _token) public {
    uint256 bal = calculatePayout(msg.sender, _token);

    _updateUserData(msg.sender, address(0), _token, 0);

    if (_token != address(0)) {
      IERC20(_token).transfer(msg.sender, bal);
    } else {
      msg.sender.transfer(bal);
    }
  }

  function calculatePayout(address _user, address _token) public view returns (uint256) {
    string memory key = string(abi.encodePacked(_user, _token));
    uint256 bal = dataUint256[key];
    address[] storage events = dataManyAddresses[key];
    for (uint256 i = 0; i < events.length; i += 1) {
        IEvent e = IEvent(events[i]);
        if (e.hasEnded()) {
            bal += e.getPayout(_user);
        }
    }
    return bal;
  }

  function calculateDeposit(address _user, address _token) public view returns (uint256) {
    string memory key = string(abi.encodePacked(_user, _token));
    uint256 bal = 0;
    address[] storage events = dataManyAddresses[key];
    for (uint256 i = 0; i < events.length; i += 1) {
        IEvent e = IEvent(events[i]);
        if (!e.hasEnded()) {
            bal += e.getDeposited(_user);
        }
    }
    return bal;
  }

  /* Update the data associated with this user in the data storage contract.
   *
   * Note: for each user we constantly keep track of the list of non-ended events they have registered to attend as well as
   * the their current ETH balance, based on their previous contract payouts.
   *
   * @param _user Address of the user to update.
   * @param _newEvent The address of the new event they've registered for. If 0 then they haven't registered for a new event.
   * @param _token The token being dealt with. If null address then ETH is the token.
   * @param _newBalance The user's new ETH leftover balance.
   */
  function _updateUserData(address _user, address _newEvent, address _token, uint256 _newBalance) internal {
    string memory key = string(abi.encodePacked(_user, _token));

    dataUint256[key] = _newBalance;

    address[] storage events = dataManyAddresses[key];
    uint256 numEvents = events.length;

    for (uint256 i = 0; i < numEvents; i += 1) {
      IEvent e = IEvent(events[i]);
      // remove ended events
      if (e.hasEnded()) {
        // overwrite with last item if possible
        if (i < numEvents - 1) {
          events[i] = events[numEvents - 1];
        }
        // remove last item
        events[numEvents - 1] = address(0);
        numEvents--;
      }
    }

    events.length = numEvents;

    if (_newEvent != address(0)) {
      events[numEvents] = _newEvent;
      events.length++;
    }
  }
}
```

**IEvent, Event**

The event contracts (i.e. the existing `Conference` contract). Just the rough minimum requirements are given here.

```solidity
/**
 * Interface for events
 */
interface IEvent {
  function hasEnded() external view returns (bool);
  function getToken() external view returns (address);
  function getPayout(address _addr) external view returns (uint256);
  function getDeposit(address _addr) external view returns (uint256);
}



contract Event is IEvent {
  // ... existing fields
  IUserPot public userPot;

  constructor (
    address _userPot,
    string _name,
    uint256 _deposit,
    uint256 _limitOfParticipants,
    address _owner
  ) public {
    userPot = IUserPot(_userPot);
    // ...
  }

  function register() public payable onlyActive{
    require(registered < limitOfParticipants, 'participant limit reached');
    require(!isRegistered(msg.sender), 'already registered');

    // send to user pot
    userPot.deposit.value(msg.value)(msg.sender);

    registered++;
    participantsIndex[registered] = msg.sender;
    participants[msg.sender] = Participant(registered, msg.sender);

    emit RegisterEvent(msg.sender, registered);
  }

  function getToken() public returns (address) {
    return address(0); // ETH
  }

  function getPayout(address _user) public view returns (uint256) {
    if (!ended || !isAttended(_user)) {
        return 0;
    }
    return payoutAmount;
  }

  function getDeposit(address _user) public view returns (uint256) {
    return deposit;
  }

  function hasEnded() public view returns (bool) {
    return ended;
  }

  // .... rest of the code almost same as the existing Conference contract
}
```


## Future considerations

The following topics has been discussed with respect to this KIP.

### Automatic withdrawal

Event though this change eliminates the need to call `withdraw()` per event, the user still has to withdraw manually at some point if they want to have their ETH back.

Some users expressed that they would prefer that their ETH/tokens get sent back automatically. This could be achieved in future by having the `withdraw()` function be called on behalf of a user by a third-party. But realistically speaking, users are probably better off not _needing_ to withdraw anymore, and only withdrawing later on when they actually _want_ to.

### Support for different payout strategies

There is some interest in different payout strategies, e.g. using sweepstake and for different logic when handling a paid event. This KIP will enable us to cater for such variations as it detaches event-related logic from the money management side of things.

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


