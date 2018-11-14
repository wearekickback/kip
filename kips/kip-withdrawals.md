---
kip: <to be assigned>
title: Allow for easier withdrawals and re-using payouts
author: @hiddentao
status: Draft
created: 2018-11-11
---

## Simple Summary


If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
A short (~200 word) description of the technical issue being addressed.

## Motivation
<!--The motivation is critical for KIPs. It should clearly explain why the existing specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->
The motivation is critical for KIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature.-->
The technical specification should describe the syntax and semantics of any new feature.

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

## Backwards Compatibility
<!--All KIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. KIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
All KIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. KIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Implementation

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

// don't want to pollute this KIP with utility contract definitions, so this will do for now...
import "https://github.com/runningbeta/rbac-solidity/raw/master/contracts/RBACWithAdmin.sol";

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




## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
