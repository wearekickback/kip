---
kip: <to be assigned>
title: Allow for easier withdrawals and re-using payouts
author: @hiddentao
status: Draft
created: 2018-11-11
---

## Simple Summary

This change removes the cooling period (thus allowing user's to withdraw any time) and minimizes the gas cost for user's to withdraw their payouts across multiple events. It also makes it possible to reuse a payout as a deposit for a new event without having to withdraw it first.

## Motivation

Our current setup requires the user to manually withdraw their ETH payout from an event contract once the event has been finalized, but within a set *cooling period* (usually 1 week) after the event ends, after which withdrawal is no longer possible and the remaining ETH can be taken by the event organizer.

Although very few people have disagreed or complained about the above model, in practice
In practice we find that many users either a) are unaware they need to withdraw, b) forget to withdraw, c) are unable to withdraw within the cooling period, or d) would prefer auto-withdrawal. Withdrawal also costs gas, which cuts into the user's payout, plus they already had to pay gas when RSVP'ing for an event.

## Specification

We introduce 2 new contracts in addition to the existing event contract:

- `UserPot` - a single instance of this holds all ETH for all users across all events
- `Storage` - a single instance of this holds all data which keeps track of which users are attending which event and how much payout each user is owed

When a user RSVP's to an event this is the new flow:

1. User sends a transaction containing ETH to the event's deployed contract
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

- **Single user pot is necessary** - because our system redistributes ETH between users (i.e. if you don't show up your ETH gets _given_ to other users) it's necessary to have all ETH for a given event stored in one place, as it would be too expensive gas-wise and time-wise to send ETH amongst the participants of an event. This is why we can't have a `UserPot` instance per user.

- **Reuse of deposit** - because we have a single user pot instance, we can easily reuse the payout from an old event as a deposit for a new event. In the sample implementation below, this is actually what happens when a user RSVP's. Thus, users do not need to withdraw their ETH and send it back in in order to attend future events.

- **Easier overview of user balance** - because all ETH is in one place we can easily give the user an on-chain overview of their total deposits and payouts on Kickback.

- **Upgrade-able contracts** - separating the payment processing logic from data storage and from event management makes it easier for us to deploy updates to the various contracts. The data storage contract is kept as simple as possible and as a pure re-usable data store, albeit with role-based access control to ensure its security.

## Risks and and trade-offs

Although gas costs for RSVP'ing have gone up, I think this is justifiable given the improvements in overall user experience, plus the additional flexibility we gain over our current model.

The events emitted by the event contract will all stay the same - meaning our backend doesn't need to change enormously for this. However we will no longer show when a user has withdrawn from an event - there's no need to since withdrawals across events are now batched.

Storing all ETH into a single contract may seem risky, but we can negate this with good testing and auditing of the contract.

## Implementation

Note:

- `RBACWithAdmin` - see https://github.com/runningbeta/rbac-solidity/raw/master/contracts/RBACWithAdmin.sol

**Data storage**

This contract is only for storing data, thus allowing us to easily upgrade the `UserPot` contract below.

The interface:

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

This contracts hold all the ETH committed to events. Users can reuse payouts as deposits for new events, or withdraw all of their payouts all together in one go.

The interface:

```solidity
interface UserPotInterface {
  function deposit(address _event, address _user, uint256 _deposit) external payable;
}

import "./StorageInterface.sol";
import "./EventInterface.sol";
import "./ERC165.sol";
import "RBACWithAdmin.sol";

contract UserPot is RBACWithAdmin, ERC165, UserPotInterface {
    bytes32 public constant STORAGE_KEY_EVENTS = keccak256("events");
    bytes32 public constant STORAGE_KEY_LEFTOVER = keccak256("leftover");

  StorageInterface dataStore = StorageInterface(0x4920ebe161687f4a2180a698171ff5bfb2fbac65);

uint256 INTERFACE_ID;

constructor () {
   INTERFACE_ID = this.deposit.selector ^ this.withdraw.selector ^ this.calculatePayout.selector ^ this.calculateDeposit.selector;
}

  function deposit(address _user) external payable {
    EventInterface event = EventInterface(msg.sender)
    uint256 _deposit = event.getDeposit(_user)
    uint256 bal = calculatePayout(_user);
    bal += msg.value;
    require(bal >= _deposit, 'you need to pay more to register for event');
    updateUserData(_user, msg.sender, bal - _deposit);
  }

    function withdraw() public {
        uint256 bal = calculatePayout(msg.sender);
        msg.sender.transfer(bal);
        updateUserData(msg.sender, address(0), 0);
    }

    function updateUserData(address _user, address _newEvent, uint256 _newBalance) internal {
        address[] memory events = dataStore.getAddresses(_user, STORAGE_KEY_EVENTS);
        address[] memory newEvents = new address[](events.length + 1);
        uint256 newEventsLength = 0;
        uint256 len = events.length;
        for (uint256 i = 0; i < events.length; i += 1) {
            EventInterface e = EventInterface(events[i]);
            // only remember still-active events
            if (!e.hasEnded()) {
                newEvents[newEventsLength] = events[i];
                newEventsLength++;
            }
        }
        if (_newEvent != address(0)) {
                newEvents[newEventsLength] = events[i];
                newEventsLength++;
        }
       // we should add a convenience method to StorageInterface which combines these two calls into one!
      // TODO: passing the array through doesn't quite work yet
        dataStore.setAddresses(_user, STORAGE_KEY_EVENTS, events, newEventsLength);
        dataStore.setUint(_user, STORAGE_KEY_LEFTOVER, _newBalance);
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
            bal += e.getDeposit(_user);
        }
    }
    return bal;
  }

    function supportsInterface(bytes4 interfaceID) external view returns (bool) {
        return
          interfaceID == this.supportsInterface.selector || // ERC165
          interfaceID == INTERFACE_ID;
    }

  function destroy(address _destination) public onlyAdmin {
      ERC165 i = ERC165(_destination);
      require(i.supportsInterface(INTERFACE_ID));
      selfdestruct(_destination);
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
        userPot.deposit.value(msg.value)(address(this), msg.sender, deposit);

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
