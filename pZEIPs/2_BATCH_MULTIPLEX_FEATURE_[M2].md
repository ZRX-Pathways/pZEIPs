## Summary
This is an update on the current progress on pZEIP-2, which is described [here](https://github.com/ZRX-Pathways/pZEIPs/pull/2/files).
The goal is to provide sufficient information about the progress made so far, the results obtained, and the learnings, while at the same time gathering community feedback in order to move pZEIP-2 to a formal ZEIP and its subsequent addition as a new protocol extension.
This document also serves to inform future contributors so they can grasp a clearer understanding of the idiosyncrasies of the 0x Exchange Proxy (EP), in particular with regards to native ETH transactions and arbitrary calldata inputs.

## Updates on specifications
The starting point of pZEIP-2 is to provide the 0x Exchange Proxy with a method to execute multiple swap transactions in a single call without imposing new transaction types. This way, the individual single-swap calls can be combined into an array of calls with the same format. The encoded array is then attached as `calldata` to the `batchMultiplex` call.
Another base rule of the implementation is to avoid incorporating swap logic in the new extension, therefore minimizing the risk of creating new attack vectors.
Slight modifications have been applied to the original specifications:
  - *enum* type `ErrorHandling` has been renamed `ErrorBehavior` for coherence with [pZEIP-1](https://github.com/ZRX-Pathways/pZEIPs/pull/1/files);
  - *enum* type `ErrorBehavior` has been implemented as a library in *src/features/libs/LibTypes.sol*.

Instead of publicly exposing three new methods, two of them were merged into a single one. Therefore, the new implementation exposes two methods.
For simplicity of debugging with the Typescript test suite, the two methods have been assigned different names, specifically:
  - `batchMultiplex` and `batchMultiplexOptionalParams` for the *BatchMultiplexFeature.sol* contract;
  - `batchMultiplexV2` and `batchMultiplexV2OptionalParams` for the *BatchMultiplexFeatureV2.sol* contract.
As Foundry tests have been implemented and it is expected that Foundry will become the default test framework for new features, the two methods could have the same name and still result in different method selectors.

### Implementations and the "simple" way
The new feature has been implemented in two different files: *BatchMultiplexFeature.sol* and *BatchMultiplexFeatureV2.sol*. The reason behind this is that whenever native ETH (or the native chain's token) is involved, the transaction needs special handling, as otherwise it will or may revert unexpectedly. A detailed explanation of the affected calls is described below.
A sample *BatchMultiplexValidator.sol* contract has been added to a new */src/examples* folder. Tests have been added to Typescript and Forge tests, but given Foundry's speed, it is advisable to use the latter only going forward.

The *BatchMultiplexFeatureV2.sol* operates as indicated in pZEIP-2, with the only limitation that it is restricted to ERC20, ERC721, and ERC1155 tokens. Native token (i.e., ETH for Ethereum and L2s, BNB for Bsc, MATIC for Polygon, etc.) transactions will be incorrectly forwarded and will revert. Only two limitations or requirements arise when using *BatchMultiplexFeatureV2*:
1) when a user wants to swap ETH for a batch of tokens, he will have to wrap it as a token first with a separate transaction and approve the 0x Exchange as a spender;
2) when a user wants to swap NFT-1 for NFT-2 in a single transaction, he will have to pre-approve the ERC20 token used to buy the NFT-2. The extra approval step could be removed by "baking" a new feature into the protocol, which would require implementing swap logic and is out of scope at the moment.

With these limitations in mind, the *BatchMultiplexFeatureV2* implements two methods for forwarding multiple calls to the 0x Exchange Proxy with the following characteristics:
  - no new types are imposed on the feature;
  - no swap logic is implemented;
  - no need to blacklist or whitelist methods;
  - optional `staticcall` before-trade validation in an external arbitrary contract;
  - optional behavior if one of the sub-calls reverts: `REVERT` `STOP` `CONTINUE`.

The feature's ABI has been updated as follows:
```
function batchMultiplexCallV2(bytes[] calldata data) external returns (bytes[] memory results);

function batchMultiplexOptionalParamsCallV2(
        bytes[] calldata data,
        LibTypes.ErrorBehavior errorBehavior
    ) external returns (bytes[] memory results);
```

One of the possibilities for allowing use of the *BatchMultiplexFeatureV2* would be to allow the 0x proxy to hold the tokens during the transaction, with the user wrapping as the first transaction and unwrapping as the last. However, in this case, the client would also have to encode unwrapping a token at the end of the transaction and sweeping the tokens left in the 0x Exchange Proxy contract.
A simpler solution would be to modify the interface of the ETH-related methods with an additional `unit256 amountTokenIn` parameter and avoid the use of `msg.value`. However, this would not completely solve the problem as, in some cases, the 0x Exchange Proxy will try to refund excess ETH. Technically, this could be solved by adding the extra boolean parameter `shouldRefundExcessEth`, and in this case, the batchMultiplex methods should assert that any ETH leftover is refunded. In order to prevent this edge scenario with future protocol upgrades, the *BatchMultiplexFeatureV2* methods are non-payable, while the *BatchMultiplexFeature* methods are payable and refund excess Eth at the end of the call.

### The more complex implementation
When native ETH is involved, it is not always possible to correctly forward the individual calls to the 0x Exchange Proxy. Our "simple" implementation, in fact, relies on using the low-level `delegatecall` call, which preserves `msg.sender` and `msg.value`. In the 0x Exchange V4, this means that ETH-related methods may or will revert in the event that multiple calls try to use native ETH.

As some methods require custom routing, we have to keep information about the individual methods in storage. In order to do this, we define three possible states for a selector: `Whitelisted`, `Blacklisted`, and `RequiresRouting`. We define this in *src/storage/LibBatchMultiplexStorage.sol* as a mapping of status by selector. The mapping is updated by calling `updateSelectorsStatus(UpdateSelectorStatus[] memory selectorsTuple)`, which is restricted to the 0x Exchange Proxy owner. When a non-whitelisted (i.e., either `Blacklisted` or `RequiresRouting`) method's state is updated, its selector is removed from the mapping, so that all methods but the ones that are either explicitly blacklisted or require special handling are whitelisted. This flow is efficient as it avoids having to whitelist all implemented methods, thus not requiring an update when new features are added to the proxy. The external method `getSelectorStatus` allows to investigate the status of a selector. Routing follows the same principles as the *MultiplexFeature* and uses some of its code with necessary modifications.
At the point of extension registration to the 0x Exchange Proxy, a few methods are registered as follows:
1. Blacklisted methods:
    - Meta transactions are not supported by default, MetaTransactionsV2 could be supported without need for routing:
      - `executeMetaTransactionV2`;
      - `batchExecuteMetaTransactionsV2`;
      - `executeMetaTransaction`;
      - `batchExecuteMetaTransactions`.
    - Batch NFT swaps:
      - `batchBuyERC721s`;
      - `batchBuyERC1155s`.
    - Remaining blacklisted methods:
      - `fillOtcOrderWithEth`;
      - `sellEthForTokenToUniswapV3`;
      - `sellToPancakeSwap`;
      - `sellToUniswap`;
      - `fillOrKillLimitOrder`;
      - `batchFillLimitOrders`.

2. Methods that require special routing and that will revert if not explicitly handled by the feature:
    - Multiplex payable methods that are not supported but could, if properly handled (they require passing individual values):
      - `multiplexBatchSellEthForToken`;
      - `multiplexMultiHopSellEthForToken`.
    - supported methods:
      - `sellToLiquidityProvider`;
      - `transformERC20`;
      - `buyERC721`;
      - `buyERC1155`;
      - `fillLimitOrder`.

Batch methods are blacklisted as they can be executed as an array of individual calls.
Some methods are explicitly restricted but could be supported. *MetaTransactionsFeatureV2* methods, for example, are non-payable and could be supported without the need for routing. However, meta transactions in general are blacklisted, as a batch of swaps can be executed by passing them as an array or `calldata` arguments to a single meta transaction.

We now dig deeper into why the remaining methods are blacklisted or routed through a custom path.
**Notice:** Some of the blacklisted methods can alternatively be executed "synthetically" by using a method that uses a custom route. This way, we can minimize the number of registered methods and develop a dedicated route only after it appears justified by the demand for that specific method.

- Blacklisted methods
  1. `fillOtcOrderWithEth`

      The method will wrap the attached value into WETH, and then it will [try to refund](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/OtcOrdersFeature.sol#L169C58-L169C77) the difference between `msg.value` and `takerTokenFilledAmount`, therefore preventing other calls from using the attached ETH. Technically, we could support the feature by wrapping `takerTokenFilledAmount`, routing the call to `_fillOtcOrder`, unwrapping and withdrawing excess ETH (if any). However, the decision of whether to implement custom routing for this method should be based on demand for the feature.

  2. `sellEthForTokenToUniswapV3`

      The method will try to wrap `msg.value` into WETH, which will revert if invoked multiple times. The method could be implemented by wrapping `sellAmount` and routing to `_sellTokenForTokenToUniswapV3`, but the feature can be executed by using the `sellToLiquidityProvider` method in the *LiquidityProviderFeature*.

  3. `sellToPancakeSwap`

      When trying to use this feature to swap tokens, but ETH has been attached to the call (i.e., for invoking other methods using ETH), the call [will revert](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/PancakeSwapFeature.sol#L190C31-L190C40). The call could be correctly executed with custom routing, but executing a `call` to `sellToPancakeSwap` would require adding some custom logic, and, as the *PancakeSwapFeature* is entirely written in assembly, it is easy prey to error. The method can be executed by routing to `sellToLiquidityProvider` instead.

  4. `sellToUniswap`

      Same as the previous point.

  5. `fillOrKillLimitOrder`

      The method will try to [refund the attached value](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/native_orders/NativeOrdersSettlement.sol#L204). Technically, it would be possible to route to `_fillLimitOrder`, but the decision to exclude was based on not identifying a clear demand for the feature.

- Methods that require routing
  1. `multiplexBatchSellEthForToken`

      This method will deposit attached ETH into WETH, which will lead to errors when other calls in the batch array try to use attached ETH. This method could be implemented by slightly modifying the interface of the *BatchMultiplexFeature.batchMultiplex(...params[])* by adding an additional param `uint256[] values` to *BatchMultiplexFeature.batchMultiplex(...params[], values[])*, wrapping passed `value` and routing to the `_multiplexBatchSell` internal method. This will not substantially change how the transaction is encoded at an API level, as the `value` parameter is attached to the transaction call, separate from the `calldata`.
  **Notice: _We seek feedback from the community on this specification, as this will also result in a different interface for the *BatchMultiplexFeature* and the *BatchMultiplexFeatureV2*. To be clear, this would not require an upgrade in the *MultiplexFeature*, from which much of the inspiration for this feature has come, and some of its code has been reused with proper adaptations where necessary._**

  2. `multiplexMultiHopSellEthForToken`

      Same as the previous point.

  3. `sellToLiquidityProvider`

      The call tries to deposit attached ETH into WETH, which will lead to errors when other calls looking to use attached ETH are included in the batch array. Therefore, whenever ETH is involved, we forward the call to the same method by overwriting `value` as `sellAmount` instead of the original call's `msg.value`.

  4. `transformERC20`

      Same issue as in the previous point. Whenever ETH is involved, we forward the call to `_transformERC20` by overwriting `value` as `args.inputTokenAmount` and `msg.sender` as `taker`.

  5. `buyERC721`

      When ETH is involved, the EP's [balance](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/nft_orders/ERC721OrdersFeature.sol#L111) will underflow (i.e., be corrected to zero by safe math) with multiple calls in the batch array accessing the attached ETH. This will lead to reverts when the underlying feature tries to refund excess ETH. In order for the call to execute correctly, we need to *delegatecall* `_buyERC721` and pass `address(this).balance` as the available balance parameter.

  6. `buyERC1155`

      Same as previous, but *delegatecall* to `_buyERC1155`.

  7. `fillLimitOrder`

      The method will try to [refund the protocol fee](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/native_orders/NativeOrdersSettlement.sol#L140C24-L140C55). While the protocol fee has been suspended, any unspent ETH is seen as an excess protocol fee, and its refund will lead to reverts when multiple calls in the batch array try to access the attached ETH. In order for the call to execute correctly, it is forwarded to `_fillLimitOrder` with a nil value and `msg.sender` as both the `sender` and the `taker`.


Some methods are flagged as blacklisted but could be implemented with a custom routing. The choice has been made to prefer minimize the number of custom features in order to reduce the error surface. Generally speaking, it is advisable to start by including the minimum number of supported methods based on the current demand for the features and subsequently add more as it becomes clearer what features should be included.

**Notice:** after the swaps are executed, but before the end of the transaction, we unwrap any WETH leftover and forward it to the `msg.sender`. As in our implementation no WETH is left in the 0x Exchange Proxy at the end of execution, this method is theoretically unnecessary and works as a safeguard measure that could be removed after further test assertions.

### Specificities of developing a multicall extension for the 0x Exchange Proxy
When forwarding an array of calls to the 0x Exchange Proxy, a developer must be aware that:
- `delegatecall` is a safe method to use, but will revert in some specific circumstances as the original call's `msg.value` is preserved;
- some publicly exposed internal methods have been developed that allow custom routing of calls that require handling. These methods allow forwarding the desired ETH;
- using `call` to forward a batch of swaps is generally unsafe, as:
  - the `msg.sender` is changed to the 0x Exchange Proxy itself, and therefore some external methods will be incorrectly executed;
  - when calling the publicly exposed internal methods, some parameters will be passed to the child, which would make the 0x Exchange Proxy insecure. In particular, in some cases, a `taker` can be passed, which is different from the `msg.sender`; in others, a `payer` can be passed, which is different from the `msg.sender`.
  While the first does not pose a security thread but "violates" the rule of the 0x protocol, the second poses a security thread as anyone would be able to move another user's funds (provided the user has an approval set to the 0x Exchange Proxy). Furthermore, the ABI of the publicly exposed internal methods is different from the ABI of the external methods that use them, thus requiring an amendment in the format at an API-provider level;
- a method cannot pass arbitrary data with `call` to self. Instead, inputs **MUST** be decoded according to the expected type and correctly routed with parameters validation;
- generally speaking, it is good practice to:
  - decode the encoded data according to their selector (the first four bytes of the `calldata`);
  - optionally revert with unsupported methods. This is helpful for debugging, as these transactions are blocked because they would otherwise fail;
  - correctly route calls, i.e.:
    - if a transaction requires special encoding:
      - if it is an NFT call, `delegatecall` to `self` with the correct input parameters;
      - if no ETH is involved, `delegatecall` to self;
      - `call` the publicly exposed internal methods (defined by the `onlySelf` modifier) with the correctly transformed input parameters;
  - `delegatecall` to self otherwise.

This will allow safe routing of the swaps to the internal methods, **AND** to maintain the 0x API order format.

## Notes
Calls to publicly exposed internal methods could skip pre-checks of the external methods, i.e., modifiers. Therefore, the batch multiplex methods **MUST** enforce those same assertions.
A `nonReentrant` modifier has been added to the *BatchMultiplexFeature*, as it skips the same high-level check when performing publicly exposed internal methods. The same modifier has not been implemented in the *BatchMultiplexFeatureV2*, as the calls get individually forwarded back to the 0x Exchange Proxy, where the individual methods implement a non-reentrant modifier where needed. In the more complex *BatchMultiplexFeature*, which allows using methods involving ETH, as the call is sometimes forwarded to an internal method, using a no-reentrancy lock is probably beneficial and necessary.
The upcoming Ethereum Cancun hardfork will activate [EIP1553](https://eips.ethereum.org/EIPS/eip-1153), which will make non-reentrancy modifiers way less gas-expensive (~15k gas per reentrancy lock). While this gas optimization could be helpful at this stage, it could be a better fit for a general upgrade of all the reentrancy locks of the 0x Exchange Proxy and is therefore considered out of scope.
For reference, the Ethereum Cancun upgrade is expected to go live on March 13, 2024, and Solidity v0.8.4 supports `tstore` and `tload` opcodes in inline assembly (the compiler will display a warning when using transient storage).
In order to validate a batch of swaps, the feature calls an arbitrary smart contract with a `staticcall`. In order to prevent denial of service attacks, it might be desirable to set a gas limit to the call, where the limit should be 2100 gas for accessing the external contract the first time, plus a stipend for executing validation.
For the purpose of maximum simplicity in the *BatchMultiplexFeatureV2*, the calldata validation in an external contract has been omitted. Therefore, the feature allows executing a batch of swaps and optionally defines `ErrorBehavior`.
A `onlyDelegateCall` modifier has been added to both implementations to prevent direct calls to the implementation contracts, but is not strictly necessary. Removing it would result in only marginal gas savings.
Both implementations are deemed safe in regards to new protocol features, as calls with non-mapped selectors will be forwarded to the EP with `delegatecall`.
Both implementations are deemed safe in the context of meta transactions (should the decision be made to support them in the new feature), as the 0x Exchange always validates the maker's signatures, i.e., the 0x Exchange does not use "trusted forwarders." If, however, a trusted forwarder were allowed to settle using the 0x Exchange and skip signature validation, the batch multiplex implementation should be amended to overwrite the `sender`. This issue is explained in [this post](https://blog.openzeppelin.com/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure) about ERC-2771 arbitrary address spoofing attacks.

## Conclusions
While pZEIP-2 was originally designed with the intent to put more emphasis on swaps of batches of NFTs and ERC20s, substantial work has been required on the native-ETH-related methods. While a simpler *BatchMultiplexFeatureV2* contract allows multiple token swaps as per the proposal's specs, the decision has been made to invest substantial time in debugging every single EP method for allowing full coverage of the 0x Exchange features (thus including native-ETH transactions). Two different implementations have been written so that the community can now compare the two proposed solutions, as they both offer advantages and disadvantages. The *BatchMultiplexFeatureV2* allows executing multiple swaps according to specs and has the advantage of being very simple, but excludes transactions using native ETH; should the underlying ETH-related methods be upgraded, the feature will require upgrading as its methods should now be made `payable`. The *BatchMultiplexFeature* is way more complex, is still not feature-complete (a few exchange methods are blacklisted), and requires an audit as it routes external calls to publicly exposed internal methods. In this context, the community should indicate whether the added complexity justifies the extra functionality.
