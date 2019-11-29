---
kip: 1
title: Allow for easier withdrawals and re-using payouts
authors: Ramesh Nair, Jeff Lau
status: Superseded
created: 2018-11-11
---

## Simple Summary

**NOTE: This KIP has been superseded by [KIP-2](https://github.com/wearekickback/KIPs/blob/master/kips/kip-2.md)**

This change removes the cooling period (thus allowing users to withdraw any time) and minimizes the gas cost for user's to withdraw their payouts across multiple events. It also makes it possible to reuse a payout as a deposit for a new event without having to withdraw it first.

## Motivation

Our current setup requires the user to manually withdraw their ETH payout from an event contract once the event has been finalized, but within a set *cooling period* (usually 1 week) after the event ends, after which withdrawal is no longer possible and the remaining ETH can be taken by the event organizer. 

Although very few people have disagreed or complained about the above model, in practice
In practice we find that many users either a) are unaware they need to withdraw, b) forget to withdraw, c) are unable to withdraw within the cooling period, or d) would prefer auto-withdrawal. Withdrawal also costs gas, which cuts into the user's payout, plus they already had to pay gas when RSVP'ing for an event.

## Specification

We introduce 2 new contracts in addition to the existing event contract:

- `UserPot` - a single instance of this holds all ETH for all users across all events
- `Storage` - a single instance of this holds all data which keeps track of which users are attending which event and how much payout each user is owed

When a user RSVP's to an event this is the new flow:

1. User calls `register()` on the event's deployed contract with enough ETH
2. The event's contract forwards the ETH to the global `UserPot` contract
3. The `UserPot` adds the event address to the list of events the user is attending - this data is stored in the `Storage` contract
4. `UserPot` also calculates the amount of ETH withdrawable by the user (based on past event payouts owed to them) and records this number - also in the `Storage` contract

When a user wishes to withdraw their ETH:

1. The user send a transaction to `UserPot` asking to withdraw
2. `UserPot` calculates all the ETH owed to the user based on past attendance - data for this is obtained from the `Storage` contract
3. `UserPot` updates the `Storage` contract data for the user, moving events which have ended from the user's event list, and setting their withdrawable balance to 0
4. `UserPot` sends the total ETH back to the user

## Rationale

The above design was arrived at after considering various alternatives as well as other desirable features that users have asked for. Here are the key points:

- **Singleton user pot is necessary** - because our system redistributes ETH between users (i.e. if you don't show up your ETH gets _given_ to other users) it's necessary to have all ETH for a given event stored in one place, as it would be too expensive gas-wise and time-wise to send ETH amongst the participants of an event. This is why we can't have a `UserPot` instance per user.

- **Reuse of deposit** - because we have a single user pot instance, we can easily reuse the payout from an old event as a deposit for a new event. In the sample implementation below, this is actually what happens when a user RSVP's. Thus, users do not need to withdraw their ETH and send it back in in order to attend future events.

- **Easier overview of user balance** - because all ETH is in one place we can easily give the user an on-chain overview of their total deposits and payouts on Kickback.

- **Upgrade-able contracts** - separating the payment processing logic from data storage and from event management makes it easier for us to deploy updates to the various contracts. The data storage contract is kept as simple as possible and as a pure re-usable data store, albeit with role-based access control to ensure its security. We use the `UpgradeableInterface` interface in concert with [`ERC165`](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/introspection/ERC165.sol) to enable upgrade-able contracts (see below).

- **Configurable event incentives** - one of the nice things about this architecture is that the `UserPot` is only responsible for summing up deposits and payouts - actual calculation of what the payout should be is done by the `Event` contract. If we wished to test out new strategies (e.g. sweepstakes) or try using a single `Event` instance for multiple events we could easily do so.

## Risks and and trade-offs

Although gas costs for RSVP'ing have gone up, I think this is justifiable given the improvements in overall user experience, plus the additional flexibility we gain over our current model.

The events emitted by the event contract will all stay the same - meaning our backend doesn't need to change enormously for this. However we will no longer show when a user has withdrawn from an event - there's no need to since withdrawals across events are now batched.

Storing all ETH into a single contract may seem risky, but we can negate this with good testing and auditing of the contract, and by managing admin access using multisig accounts with op-sec supervision.

## Implementation

Note:

- `RBACWithAdmin` - see https://github.com/runningbeta/rbac-solidity/raw/master/contracts/RBACWithAdmin.sol
- `ERC165` - see https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/introspection/ERC165.sol

**Data storage**

This contract is only for storing data, thus allowing us to easily upgrade the `UserPot` contract below.

```solidity
pragma solidity ^0.4.25;

interface StorageInterface {
  function getAddresses(address _addr, bytes32 _key) external view returns (address[]);
  function setAddresses(address _addr, bytes32 _key, address[] _value, uint256 _len) external;

  function getString(address _addr, bytes32 _key) external view returns (string);
  function setString(address _addr, bytes32 _key, string _value) external;

  function getBytes32(address _addr, bytes32 _key) external view returns (bytes32);
  function setBytes32(address _addr, bytes32 _key, bytes32 _value) external;

  function getUint(address _addr, bytes32 _key) external view returns (uint);
  function setUint(address _addr, bytes32 _key, uint _value) external;
}

import "./RBACWithAdmin.sol";

contract Storage is RBACWithAdmin, StorageInterface {
  bytes32 public constant ROLE_WRITER = keccak256("writer");

  struct Data {
    mapping(bytes32 => address[]) addresses;
    mapping(bytes32 => string) strings;
    mapping(bytes32 => bytes32) bytes32s;
    mapping(bytes32 => uint) uints;
  }

  mapping(address => Data) data;

  modifier isAuthorized (address _addr) {
    require(hasRole(msg.sender, ROLE_ADMIN) || hasRole(msg.sender, ROLE_WRITER));
    _;
  }

  function getAddresses(address _addr, bytes32 _key) external view returns (address[]) {
    return data[_addr].addresses[_key];
  }

  function getString(address _addr, bytes32 _key) external view returns (string) {
    return data[_addr].strings[_key];
  }

  function getUint(address _addr, bytes32 _key) external view returns (uint) {
    return data[_addr].uints[_key];
  }

  function getBytes32(address _addr, bytes32 _key) external view returns (bytes32) {
    return data[_addr].bytes32s[_key];
  }

  function setBytes32(address _addr, bytes32 _key, bytes32 _value) external isAuthorized(_addr) {
    data[_addr].bytes32s[_key] = _value;
  }

  function setString(address _addr, bytes32 _key, string _value) external isAuthorized(_addr) {
    data[_addr].strings[_key] = _value;
  }

  function setUint(address _addr, bytes32 _key, uint _value) external isAuthorized(_addr) {
    data[_addr].uints[_key] = _value;
  }

  function setAddresses(address _addr, bytes32 _key, address[] _value, uint256 _len) external isAuthorized(_addr) {
    data[_addr].addresses[_key] = _value;
    data[_addr].addresses[_key].length = _len;
  }
}
```

**User pot**

This contracts hold all the ETH committed to events. Users can reuse payouts as deposits for new events, or withdraw all of their payouts all together in one go. It also has allows for itself to be upgraded, by sending the total balance to a newly deployed user pot instance.

NOTE: We should calculate the gas cost of the various methods (especially `updateUserData`) and then work out a reasonable limit to max. no of events a user can be regstered to attend at any one time based on that. That limit will then need to be enforced app-side as well.

```solidity
pragma solidity ^0.4.25;

interface UpgradeableInterface {
  function upgrade(address _newContract) external;
}

interface UserPotInterface {
  function deposit(address _event, address _user, uint256 _deposit) external payable;
  function withdraw() external;
  function calculatePayout(address _user) external view returns (uint256);
  function calculateDeposit(address _user) external view returns (uint256);
}

import "./Storage.sol";
import "./ERC165.sol";
import "./RBACWithAdmin.sol";
import "./EventInterface.sol";

contract UserPot is RBACWithAdmin, UserPotInterface, ERC165, UpgradeableInterface {
  bytes32 public constant STORAGE_KEY_EVENTS = keccak256("events");
  bytes32 public constant STORAGE_KEY_LEFTOVER = keccak256("leftover");
  
  bytes4 INTERFACE_ID = bytes4(keccak256('UserPotInterface'));
  
  StorageInterface dataStore;

  constructor (address _dataStore) public {
    _registerInterface(INTERFACE_ID);
    dataStore = StorageInterface(_dataStore);
   }

  function deposit(address _user) external payable {
    EventInterface _event = EventInterface(msg.sender);
    uint256 _deposit = _event.getDeposit(_user);
    uint256 bal = calculatePayout(_user);
    bal += msg.value;
    require(bal >= _deposit, 'you need to pay more to register for event');
    _updateUserData(_user, msg.sender, bal - _deposit);      
  }

  function withdraw() external {
    uint256 bal = calculatePayout(msg.sender);
    msg.sender.transfer(bal);
    _updateUserData(msg.sender, address(0), 0);
  }
  
  /* Update the data associated with this user in the data storage contract.
   * 
   * Note: for each user we constantly keep track of the list of non-ended events they have registered to attend as well as 
   * the their current ETH balance, based on their previous contract payouts.
   * 
   * @param _user Address of the user to update.
   * @param _newEvent The address of the new event they've registered for. If 0 then they haven't registered for a new event.
   * @param _newBalance The user's new ETH leftover balance.
  function _updateUserData(address _user, address _newEvent, uint256 _newBalance) internal {
    address[] memory events = dataStore.getAddresses(_user, STORAGE_KEY_EVENTS);
    
    // can only really do fixed-size arrays in memory, so we limit to 10. But we should calculate the gas cost of
    // the various methods (especially this one) and then work out a reasonable limit based on that. That limit will
    // then need to be enforced app-side as well
    address[] memory newEvents = new address[](10);
    
    uint256 newEventsLen = 0;

    for (uint256 i = 0; i < events.length; i += 1) {
      EventInterface e = EventInterface(events[i]);
      // remove ended events
      if (!e.hasEnded()) {
        newEvents[newEventsLen] = events[i];
        newEventsLen++;
      }
    }
    if (_newEvent != address(0)) {
      newEvents[newEventsLen] = _newEvent;
      newEventsLen++;
    }
    dataStore.setUint(_user, STORAGE_KEY_LEFTOVER, _newBalance);
    dataStore.setAddresses(_user, STORAGE_KEY_EVENTS, newEvents, newEventsLen);
  }
  
  function calculatePayout(address _user) public view returns (uint256) {
    uint256 bal = dataStore.getUint(_user, STORAGE_KEY_LEFTOVER);
    address[] memory events = dataStore.getAddresses(_user, STORAGE_KEY_EVENTS);
    for (uint256 i = 0; i < events.length; i += 1) {
        EventInterface e = EventInterface(events[i]);
        if (e.hasEnded()) {
            bal += e.getPayout(_user);
        }
    }
    return bal;
  }

  function calculateDeposit(address _user) public view returns (uint256) {
    uint256 bal = 0;
    address[] memory events = dataStore.getAddresses(_user, STORAGE_KEY_EVENTS);
    for (uint256 i = 0; i < events.length; i += 1) {
        EventInterface e = EventInterface(events[i]);
        if (!e.hasEnded()) {
            bal += e.getDeposited(_user);
        }
    }
    return bal;
  }
  
  function upgrade(address _newContract) external {
    ERC165 i = ERC165(_newContract);
    require(i.supportsInterface(INTERFACE_ID), 'new contract has different interface');
    selfdestruct(_newContract);
  }
}
```

**Event**

This is deployed for each event, and links back to the user pot.

```solidity
interface EventInterface {
  function hasEnded() external view returns (bool);
  function getPayout(address _addr) external view returns (uint256);
  function getDeposit(address _addr) external view returns (uint256);
}

import "./RBACWithAdmin.sol";
import "./UserPotInterface.sol";

contract Event is RBACWithAdmin, EventInterface {
  UserPotInterface userPot = UserPotInterface(0x610033b6dd5a08004e46f2097ca09b693d744118);

  constructor (
    string _name,
    uint256 _deposit,
    uint256 _limitOfParticipants,
    address _owner
  ) public {
    if (_owner != address(0)) {
        addRole (owner, ROLE_ADMIN);
    }

    // ... reset of code same as existing Conference constructor
  }

  function register() external payable onlyActive{
    require(registered < limitOfParticipants, 'participant limit reached');
    require(!isRegistered(msg.sender), 'already registered');

    // send to user pot
    userPot.deposit.value(msg.value)(msg.sender);

    registered++;
    participantsIndex[registered] = msg.sender;
    participants[msg.sender] = Participant(registered, msg.sender);

    emit RegisterEvent(msg.sender, registered);
  }


  function getPayout(address _user) external view returns (uint256) {
    if (!ended || !isAttended(_user)) {
        return 0;
    }
    return payoutAmount;
  }

  function getDeposit(address _user) external view returns (uint256) {
    return deposit;
  }

  function hasEnded() external view returns (bool) {
    return ended;
  }

  // .... rest of the code almost same as the existing Conference contract
}
```

## Future considerations

The following topics has been discussed with respect to this KIP.

### Automatic withdrawal

Event though this change eliminates the need to call `withdraw()` per event, the user still has to withdraw manually at some point if they want to have their ETH back.

Some users expressed that they would prefer that their ETH gets sent back automatically. This could be achieved in future by having the `withdraw()` function be called on behalf of a user by a third-party.

### Support for different payout strategies

There is some interest in different payout stratgies, e.g. using sweepstake and for different logic when handling a paid event. This KIP will enable us to cater for such variations as it detaches event-related logic from the money management side of things.

### ERC20 token support

If we decide to support tokens, we will need to upgrade the `UserPot` contract to support tokens.


## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
