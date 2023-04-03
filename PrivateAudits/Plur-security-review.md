# Plur Systems Security Review by Rajat Beladiya



***review commit hash* - [fb5779be8435bd2d86b41d0c58e0a71e2dad7559](https://github.com/plursystems/v2/tree/fb5779be8435bd2d86b41d0c58e0a71e2dad7559)**

### Scope:
`Plurry.sol` <br />
`plurryGovernor.sol`

---

# Findings Summary

|Sequence |Issue|Instances|Severity|Status
|-|:-|:-:|:-|:-:|
| [M-1](#M-1) | Weak Sources of Randomness | 1 | Medium | Acknowledged
| [L-1](#L-1) |  `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()` | 1 | Low | Acknowledged
| [NC-1](#NC-1) | TODO Left in the code | 1 | Non Critical | Acknowledged
| [NC-2](#NC-2) | Functions not used internally could be marked external | 9 | Non Critical | Fixed
| [GAS-1](#GAS-1) | Use `selfbalance()` instead of `address(this).balance` | 1 | Gas | Acknowledged
| [GAS-2](#GAS-2) | Use calldata instead of memory for function arguments that do not get mutated | 9 | Gas | Fixed
| [GAS-3](#GAS-3) | Don't initialize variables with default value | 1 | Gas | Acknowledged
| [GAS-7](#GAS-7) | Use != 0 instead of > 0 for unsigned integer comparison | 2 | Gas | Acknowledged
<br />

# Medium Issues

## <a name="M-1"></a>[M-1] Weak Sources of Randomness

### description: 

```solidity
function initializePRNG(uint256 tokenId) internal view returns (LibPRNG.PRNG memory) {
        LibPRNG.PRNG memory prng;
        unchecked {
            // pseudo random
            prng.seed(uint256(uint160(msg.sender) + tx.gasprice + uint256(blockhash(block.number - 1)) + tokenId));
        }

        return prng;
    }
```

The given function initializes a pseudo-random number generator (PRNG) with a seed value. The main issue with the function is that it may not be as random as desired due to the choice of seed components. Here `msg.sender`, `tx.gasprice`, `blockhash(block.number - 1)` and `tokenId` can be predetermine.

### recommendation: 
A more secure approach to generating randomness in a smart contract is to use a verifiable random function (VRF) provided by Chainlink.

### locations: 
- [plurry.sol#L434-442](https://github.com/plursystems/v2/blob/fb5779be8435bd2d86b41d0c58e0a71e2dad7559/src/Plurry.sol#L434-L442)

# Low Issues

## <a name="L-1"></a>[L-1]  `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()`
Use `abi.encode()` instead which will pad items to 32 bytes, which will [prevent hash collisions](https://docs.soliditylang.org/en/v0.8.13/abi-spec.html#non-standard-packed-mode) (e.g. `abi.encodePacked(0x123,0x456)` => `0x123456` => `abi.encodePacked(0x1,0x23456)`, but `abi.encode(0x123,0x456)` => `0x0...1230...456`). "Unless there is a compelling reason, `abi.encode` should be preferred". If there is only one argument to `abi.encodePacked()` it can often be cast to `bytes()` or `bytes32()` [instead](https://ethereum.stackexchange.com/questions/30912/how-to-compare-strings-in-solidity#answer-82739).
If all arguments are strings and or bytes, `bytes.concat()` should be used instead

*Instances (1)*:
```solidity
File: src/Plurry.sol

283:             keccak256(abi.encodePacked(msg.sender, periodId, maxQuantity, nonce)).toEthSignedMessageHash().recover(

```


# Non Critical Issues


## <a name="NC-1"></a>[NC-1] TODO Left in the code
TODOs may signal that a feature is missing or not ready for audit, consider resolving the issue and removing the TODO comment

*Instances (1)*:
```solidity
File: src/Plurry.sol

724:     // TODO implement getHumanReadableTraits(uint64 traits) external pure returns (Traits memory)

```

## <a name="NC-2"></a>[NC-2] Functions not used internally could be marked external

*Instances (9)*:
```solidity
File: src/Plurry.sol

417:     function supportsInterface(bytes4 interfaceId) public view override(ERC721A, IERC721A) returns (bool) {

422:     function tokenURI(uint256 tokenId) public view override(ERC721A, IERC721A) returns (string memory) {

```

```solidity
File: src/plurryGovernor.sol

37:     function votingDelay() public view override (IGovernor, GovernorSettings) returns (uint256) {

41:     function votingPeriod() public view override (IGovernor, GovernorSettings) returns (uint256) {

45:     function quorum(uint256 blockNumber)

54:     function state(uint256 proposalId)

63:     function propose(

72:     function proposalThreshold() public view override (Governor, GovernorSettings) returns (uint256) {

90:     function supportsInterface(bytes4 interfaceId)

```


# Gas Optimizations

## <a name="GAS-1"></a>[GAS-1] Use `selfbalance()` instead of `address(this).balance`
Use assembly when getting a contract's balance of ETH.

You can use `selfbalance()` instead of `address(this).balance` when getting your contract's balance of ETH to save gas.
Additionally, you can use `balance(address)` instead of `address.balance()` when getting an external contract's balance of ETH.

*Saves 15 gas when checking internal balance, 6 for external*

*Instances (1)*:
```solidity
File: src/Plurry.sol

534:         SafeTransferLib.safeTransferETH(owner(), address(this).balance);

```

## <a name="GAS-2"></a>[GAS-2] Use calldata instead of memory for function arguments that do not get mutated
Mark data types as `calldata` instead of `memory` where possible. This makes it so that the data is not automatically loaded into memory. If the data passed into the function does not need to be changed (like updating values in an array), it can be passed in as `calldata`. The one exception to this is if the argument must later be passed into another function that takes an argument that specifies `memory` storage.

*Instances (9)*:
```solidity
File: src/Plurry.sol

410:     function setBaseURI(string memory newBaseURI) external {

```

```solidity
File: src/plurryGovernor.sol

64:         address[] memory targets,

65:         uint256[] memory values,

66:         bytes[] memory calldatas,

67:         string memory description

104:         address[] memory targets,

105:         uint256[] memory values,

106:         bytes[] memory calldatas,

```

## <a name="GAS-3"></a>[GAS-3] Don't initialize variables with default value

*Instances (1)*:
```solidity
File: src/Plurry.sol

484:         for (uint256 i = 0; i < length;) {

```

## <a name="GAS-4"></a>[GAS-4] Use != 0 instead of > 0 for unsigned integer comparison

*Instances (2)*:
```solidity
File: src/Plurry.sol

257:         if (refund > 0) {

324:         if (refund > 0) {

```

