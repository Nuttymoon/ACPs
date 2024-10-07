| ACP          | 99                                                                                   |
| :----------- | :----------------------------------------------------------------------------------- |
| Title        | Subnet Manager Solidity Standard                                                     |
| Author(s)    | Gauthier Leonard ([@Nuttymoon](https://github.com/Nuttymoon))                        |
| Status       | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/98)) |
| Track        | Best Practices                                                                       |
| Dependencies | [ACP-77](../77-reinventing-subnets/README.md)                                        |

## Abstract

Define (i) a reference interface for a minimal Subnet Manager Solidity smart contract to be deployed on any Avalanche EVM chain, as well as (ii) a modular architecture to easily plug in custom “security modules” on top of this contract (e.g. to implement a PoS Subnet).

This ACP relies on concepts introduced in [ACP-77 (Reinventing Subnets)](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets). It depends on it to be marked as `Implementable`.

## Motivation

[ACP-77 (Reinventing Subnets)](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets) opens the door to managing a Subnet validator set (stored on the P-Chain) from any chain on the Avalanche Network. The P-Chain allows a Subnet to specify a "Subnet manager". This `(blockchainID, address)` pair is responsible for sending Warp messages contained within `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` on the P-Chain. This enables an on-chain program to add, modify the weight of, and remove validators.

On each validator set change, the P-Chain is willing to sign an `AddressedCall` to notify any onchain program tracking the validator set. Onchain programs must be able to interpret this message, so they can trigger the appropriate action. The 2 kinds of `AddressedCall`s [defined in ACP-77](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets#proof-of-subnet-validator-set-change) are `SubnetValidatorRegistrationMessage` and `SubnetValidatorWeightUpdateMessage`.

Given these assumptions and the fact that most of the active blockchains on Avalanche mainnet are EVM-based, we propose defining a reference implementation for a Solidity smart contract that can:

1. Hold relevant information about the current Subnet validator set, as well as historical information
2. Send validator set updates to the P-Chain by generating `AdressedCall`s defined in ACP-77
3. Correctly update the validator set by interpreting notification messages received from the P-Chain
4. Be easily extended with custom “security modules” to implement any security model (e.g. PoS). Those modules have to implement the `IACP99SecurityModule` interface and will be called by the `ACP99Manager` contract upon validator set updates.

Having an audited and open-source reference implementation freely available will contribute to lowering the cost of launching Subnets on Avalanche.

Once deployed, the `ACP99Manager` contract will be used as the `Address` in the [`ConvertSubnetTx`](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets#convertsubnettx).

## Specification

**Note:** The naming convention followed for the interfaces and contracts are inspired from the way [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/5.x/) are named after ERC standards, using `ACP` instead of `ERC`.

### IACP99Manager

Here is the proposed interface for the `IACP99Manager` contract:

```solidity
interface IACP99Manager {
    /// @notice Subnet validation status
    enum ValidationStatus {
        Registering,
        Active,
        Updating,
        Removing,
        Completed,
        Expired
    }

    /**
     * @notice Subnet validation
     * @param status The validation status
     * @param nodeID The NodeID of the validator
     * @param startTime The start time of the validation
     * @param endTime The end time of the validation
     * @param periods The list of validation periods.
     * The index is the nonce associated with the weight update.
     * @param activeSeconds The time during which the validator was active during this validation
     * @param uptimeSeconds The uptime of the validator for this validation
     */
    struct Validation {
        ValidationStatus status;
        bytes32 nodeID;
        uint64 startTime;
        uint64 endTime;
        ValidationPeriod[] periods;
        uint64 activeSeconds;
        uint64 uptimeSeconds;
    }

    /**
     * @notice Subnet validation period
     * @param weight The weight of the validator during the period
     * @param startTime The start time of the validation period
     * @param endTime The end time of the validation period (only ≠ 0 when the period is over)
     */
    struct ValidationPeriod {
        uint64 weight;
        uint64 startTime;
        uint64 endTime;
    }

    /// @notice Emitted when the security module address is set
    event SetSecurityModule(address indexed securityModule);
    /// @notice Emitted when an initial validator is registered
    event RegisterInitialValidator(
        bytes32 indexed nodeID, bytes32 indexed validationID, uint64 weight
    );
    /// @notice Emitted when a validator registration to the Subnet is initiated
    event InitiateValidatorRegistration(
        bytes32 indexed nodeID,
        bytes32 indexed validationID,
        bytes32 registrationMessageID,
        uint64 weight,
        uint64 registrationExpiry
    );
    /// @notice Emitted when a validator registration to the Subnet is completed
    event CompleteValidatorRegistration(
        bytes32 indexed nodeID, bytes32 indexed validationID, uint64 weight
    );
    /// @notice Emitted when a validator weight update is initiated
    event InitiateValidatorWeightUpdate(
        bytes32 indexed nodeID,
        bytes32 indexed validationID,
        bytes32 weightUpdateMessageID,
        uint64 weight
    );
    /// @notice Emitted when a validator weight update is completed
    event CompleteValidatorWeightUpdate(
        bytes32 indexed nodeID, bytes32 indexed validationID, uint64 nonce, uint64 weight
    );

    error ACP99Manager__ValidatorSetAlreadyInitialized();
    error ACP99Manager__InvalidSubnetConversionID(
        bytes32 subnetConversionID, bytes32 messageSubnetConversionID
    );
    error ACP99Manager__InvalidManagerBlockchainID(
        bytes32 managerBlockchainID, bytes32 conversionBlockchainID
    );
    error ACP99Manager__InvalidManagerAddress(address managerAddress, address conversionAddress);
    error ACP99Manager__ZeroAddressSecurityModule();
    error ACP99Manager__OnlySecurityModule(address sender, address securityModule);
    error ACP99Manager__InvalidExpiry(uint64 expiry, uint256 timestamp);
    error ACP99Manager__ZeroNodeID();
    error ACP99Manager__NodeAlreadyValidator(bytes32 nodeID);
    error ACP99Manager__InvalidSignatureLength(uint256 length);
    error ACP99Manager__InvalidValidationID(bytes32 validationID);
    error ACP99Manager__InvalidWarpMessage();
    error ACP99Manager__InvalidSourceChainID(bytes32 sourceChainID);
    error ACP99Manager__InvalidOriginSenderAddress(address originSenderAddress);
    error ACP99Manager__InvalidRegistration();
    error ACP99Manager__NodeIDNotActiveValidator(bytes32 nodeID);
    error ACP99Manager__InvalidUptimeValidationID(bytes32 validationID);
    error ACP99Manager__InvalidSetSubnetValidatorWeightNonce(uint64 nonce, uint64 currentNonce);

    /// @notice Get the ID of the Subnet tied to this manager
    function subnetID() external view returns (bytes32);

    /// @notice Get the address of the security module attached to this manager
    function getSecurityModule() external view returns (address);

    /// @notice Get the validation details for a given validation ID
    function getValidation(bytes32 validationID) external view returns (Validation memory);

    /// @notice Get a Subnet validator's active validation ID
    function getValidatorActiveValidation(bytes32 nodeID) external view returns (bytes32);

    /// @notice Get the current Subnet validator set (list of NodeIDs)
    function getActiveValidatorSet() external view returns (bytes32[] memory);

    /// @notice Get the total weight of the current Subnet validator set
    function subnetTotalWeight() external view returns (uint64);

    /// @notice Get the list of message IDs associated with a validator of the Subnet
    function getValidatorValidations(bytes32 nodeID) external view returns (bytes32[] memory);

    /**
     * @notice Set the address of the security module attached to this manager
     * @param securityModule_ The address of the security module
     */
    function setSecurityModule(address securityModule_) external;

    /**
     * @notice Verifies and sets the initial validator set for the chain through a P-Chain
     * SubnetConversionMessage.
     * @param subnetConversionData The subnet conversion message data used to recompute and verify against the subnetConversionID.
     * @param messsageIndex The index that contains the SubnetConversionMessage Warp message containing the subnetConversionID to be verified against the provided {subnetConversionData}
     */
    function initializeValidatorSet(
        ValidatorMessages.SubnetConversionData calldata subnetConversionData,
        uint32 messsageIndex
    ) external;

    /**
     * @notice Initiate a validator registration by issuing a RegisterSubnetValidatorTx Warp message
     * @param nodeID The ID of the node to add to the Subnet
     * @param weight The weight of the node on the Subnet
     * @param expiry The time after which this message is invalid
     * @param signature The Ed25519 signature of [subnetID]+[nodeID]+[blsPublicKey]+[weight]+[timestamp]
     */
    function initiateValidatorRegistration(
        bytes32 nodeID,
        uint64 weight,
        uint64 expiry,
        bytes memory signature
    ) external returns (bytes32);

    /**
     * @notice Resubmits a validator registration message to be sent to P-Chain.
     * Only necessary if the original message can't be delivered due to validator churn.
     * @param validationID The validationID attached to the registration message
     */
    function resendValidatorRegistrationMessage(bytes32 validationID) external returns (bytes32);

    /**
     * @notice Completes the validator registration process by returning an acknowledgement of the registration of a
     * validationID from the P-Chain.
     * @param messageIndex The index of the Warp message to be received providing the acknowledgement.
     */
    function completeValidatorRegistration(uint32 messageIndex) external;

    /**
     * @notice Initiate a validator weight update by issuing a SetSubnetValidatorWeightTx Warp message.
     * If the weight is 0, this initiates the removal of the validator from the Subnet. An uptime proof can be
     * included. This proof might be required to claim validator rewards (handled by the security module).
     * @param nodeID The ID of the node to modify
     * @param weight The new weight of the node on the Subnet
     * @param includesUptimeProof Whether the uptime proof is included in the message
     * @param messageIndex The index of the Warp message containing the uptime proof
     */
    function initiateValidatorWeightUpdate(
        bytes32 nodeID,
        uint64 weight,
        bool includesUptimeProof,
        uint32 messageIndex
    ) external;

    /**
     * @notice Completes the validator weight update process by returning an acknowledgement of the weight update of a
     * validationID from the P-Chain.
     * @param messageIndex The index of the Warp message to be received providing the acknowledgement.
     */
    function completeValidatorWeightUpdate(uint32 messageIndex) external;
}
```

#### About `Validation`s

A `Validation` represents the continuous time frame during which a node is part of the validator set and can be composed of multiple periods. A new period starts every time the validator weight changes during the same validation.

Each `Validation` is identified by its `validationID` which is the SHA256 of the Payload of the `AddressedCall` in the `RegisterSubnetValidatorTx` adding the validator to the Subnet's validator set, as defined in ACP-77.

#### About `ExitValidatorSetTx`

This transaction allows a validator to exit a Subnet validator set directly from the P-Chain to prevent the Subnet from locking validators into participating in consensus indefinitely. Therefore it is not meant to be called in the `Manager` contract.

### IACP99SecurityModule

Here is the proposed interface for the `IACP99SecurityModule` contract:

```solidity
interface IACP99SecurityModule {
    /**
     * @notice Information about a validator registration
     * @param nodeID The NodeID of the validator node
     * @param validationID The ValidationID of the validation
     * @param weight The initial weight assigned to the validator
     * @param startTime The timestamp when the validation started
     */
    struct ValidatiorRegistrationInfo {
        bytes32 nodeID;
        bytes32 validationID;
        uint64 weight;
        uint64 startTime;
    }

    /**
     * @notice Information about a validator's uptime
     * @param activeSeconds The total number of seconds the validator was active
     * @param uptimeSeconds The total number of seconds the validator was online
     * @param averageWeight The average weight of the validator during the period
     */
    struct ValidatorUptimeInfo {
        uint64 activeSeconds;
        uint64 uptimeSeconds;
        uint64 averageWeight;
    }

    /**
     * @notice Information about a change in a validator's weight
     * @param nodeID The NodeID of the validator node
     * @param validationID The ValidationID of the validation
     * @param nonce A sequential number to order weight changes
     * @param newWeight The new weight assigned to the validator
     * @param uptime The uptime information for the validator
     */
    struct ValidatorWeightChangeInfo {
        bytes32 nodeID;
        bytes32 validationID;
        uint64 nonce;
        uint64 newWeight;
        ValidatorUptimeInfo uptimeInfo;
    }

    error ACP99SecurityModule__ZeroAddressManager();
    error ACP99SecurityModule__OnlyManager(address sender, address manager);

    /// @notice Get the address of the ACP99Manager contract secured by this module
    function getManagerAddress() external view returns (address);

    /**
     * @notice Handle a validator registration
     * @param validatorInfo The information about the validator
     */
    function handleValidatorRegistration(ValidatiorRegistrationInfo memory validatorInfo)
        external;

    /**
     * @notice Handle a validator weight change
     * @param weightChangeInfo The information about the validator weight change
     */
    function handleValidatorWeightChange(ValidatorWeightChangeInfo memory weightChangeInfo)
        external;
}
```

### Reference Architecture

Each `ACP99Manager` contract will be associated with one "security module" that must implement the `IACP99SecurityModule` interface and is the only contract allowed to call the `ACP99Manager` functions related to validator set changes (`initiateValidatorRegistration`, and `initiateValidatorWeightUpdate`). Everytime a validator is added/removed or a weight change is initiated, the `ACP99Manager` will in turn call the corresponding function of the "security module" (`handleValidatorRegistration` or `handleValidatorWeightChange`). We recommend that the "security module" reference an immutable `ACP99Manager` contract address for security reasons.

It is up to the "security module" to decide what action to take when a validator is added/removed or a weight change is confirmed by the P-Chain. Such actions could be starting the withdrawal period and allocating rewards in a PoS Subnet.

```mermaid
---
title: ACP-99 Architecture with a PoA Security Module
---
graph LR
  Safe(Safe multisig)
  SecurityModule(ACP99PoAModule)
  Manager(ACP99Manager)
  P(P-Chain)

  subgraph "Manager chain"
    Safe
    SecurityModule
    Manager
  end
  Safe -.->|Own| SecurityModule
  Safe -.->|Own| Manager
  SecurityModule <-.->|Reference| Manager
  Safe -->|addValidator| SecurityModule
  SecurityModule -->|initiateValidatorRegistration| Manager
  Manager -->|sendWarpMessage| P
  P -->|completeValidatorRegistration| Manager
  Manager -->|handleValidatorRegistration| SecurityModule
```

"Security modules" could implement PoS, Liquid PoS, etc. The specification of such smart contracts is out of the scope of this ACP.

## Backwards Compatibility

The `IACP99Manager` and `IACP99SecurityModule` interfaces are only a reference interface, they don’t have any impact on the current behavior of the Avalanche protocol.

## Reference Implementation

A work in progress reference implementation is available in the [Suzaku Contracts Library](https://github.com/suzaku-network/suzaku-contracts-library/blob/acp-99-implementation/README.md#acp99-contracts-library) repository. It will be updated until this ACP is considered `Implementable` based on the outcome of the discussion.

## Security Considerations

The audit process of the `ValidatorSetManager` contract is of the utmost importance for the future of the Avalanche ecosystem as most Subnets would rely upon it to secure their Subnet.

## Open Questions

### Is there an interest to keep historical information about the validator set on the manager chain?

The functions `getValidation` and `getValidatorValidations` would allow to retrieve historical information about the validator set. The information will anyway be available on the P-Chain, so we could envision only keeping track of the active validations in the `Manager` contract. This would avoid storing historical data on the manager chain and make the implementation of the `Manager` more straightforward.

### How could we name “Security Modules”?

I don’t really like this name but cannot come up with anything else.

## Acknowledgments

Special thanks to @leopaul36, @aaronbuchwald, @dhrubabasu and @minghinmatthewlam for their reviews of previous versions of this ACP!

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).