## Summary
This is an update on the current progress on pZEIP-2, which is described [here](https://github.com/ZRX-Pathways/pZEIPs/blob/main/pZEIPs/2_BATCH_MULTIPLEX_FEATURE_[M1].md).
The goal is to provide sufficient information about the progress so far, the results obtained, the learnings, while at the same time to gather community feedback in order to move pZEIP-2 to a formal ZEIP and its subsequent addition as a new protocol extension.
This document also serves to inform future contributors, so they can can grasp a clearer understanding of idiosyncrasies of the 0x Exchange proxy, in particular with regards to native ETH transactions and arbitrary calldata inputs.

## Specification
The starting point of pZEIP-2 is to provide a multicall method to the 0x Exchange proxy to execute multiple swap transactions in a single call without imposing extra types, thus enabling the calls to be encoded as an array of calldata with the same format as the individual calls. The encoded array is then attached as `calldata` to the `batchMultiplex` call.
Another base rule of the implementation is to avoid incorporating swap logic in the new extension, therefore minimizing the risk of creating new attack vectors.
Slight modifications have been applied to the original specifications:
- enum type `ErrorHandling` has been renamed `ErrorBehavior` for coherence with [pZEIP-1](https://github.com/ZRX-Pathways/pZEIPs/blob/main/pZEIPs/1_BUNDLE_SWAP_FEATURE.md).
- enum type `ErrorBehavior` has been implemented as a library in src/features/libs
/LibTypes.sol

### What has been implemented as use-cases
The new feature has been implemented in two different files: BatchMultiplexFeature.sol and BatchMultiplexFeatureV2.sol. The reason behind this is that whenever native ETH (or the native chain's token) is involved, the transaction needs special
handling as otherwise will/may revert unexpetedly. A detailed explanation of the affected calls is described below.
A sample BatchMultiplexValidator.sol contract has been added to a new /src/examples folder. Tests have been added to Typescript and Forge tests, but given Forge's speed it is advisable to using the latter only going forward.

The BatchMultiplexFeatureV2.sol operates as indicated in pZEIP-2 with the only limitation that it is restricted to ERC20, ERC721 and ERC1155 tokens. Native token (i.e. ETH for Ethereum and L2s, BNB for Bsc, MATIC for Polygon, ...) transactions
will be incorrectly forwarded and will revert. Only two limitations/requirements arise when using *BatchMultiplexFeatureV2*:
1) when a user wants to swap ETH for a batch of tokens, he will have to wrap it as a token first with a separate transaction and approve the 0x Exchange as spender
2) when a user wants to swap NFT-1 for NFT-2 in a single transaction, he will have to pre-approve the ERC20 token used to buy the NFT-2 as the transactions

With these limitations in mind, the *BatchMultiplexFeatureV2* implements two methods for forwarding multiple calls to the 0x Exchange proxy with the following characteristics:
- no new types imposed in the feature
- no swap logic implemented
- no need to blacklist or whitelist methods
- optional staticcall before-trade validation in an external arbitrary contract
- optional bevior if one of the sub-calls reverts: `REVERT` `STOP` `CONTINUE`.

The feature ABI has been updated as follows:
```
function batchMultiplexCallV2(bytes[] calldata data) external returns (bytes[] memory results);

function batchMultiplexOptionalParamsCallV2(
        bytes[] calldata data,
        LibTypes.ErrorBehavior errorBehavior
    ) external returns (bytes[] memory results);
```

One of the possibilities for allowing use of the *BatchMultiplexFeatureV2* would be to allow the 0x proxy to hold the tokens during the transaction, with the user wrapping as first transaction and unwrapping as last. However, in this case the client
would also have to encode unwrapping a token at the end of the transaction and sweeping the tokens left in the 0x Exchange Proxy contract.
A simpler solution would be to modify the interface of the ETH-related methods with an additional `unit256 amountTokenIn` parameter and avoiding use of msg.value. However, this would not completely solve the problem as in some cases the 0x Exchange proxy will try to refund excess ETH. Technically, this could be solved by one more boolean parameter `shouldRefundExcessEth`, and in this case the batchMultiplex methods should assert that any ETH leftover is refunded. In order to prevent this edge scenario with future protocol upgrades, the *BatchMultiplexFeatureV2* methods are non-payable, while the *BatchMultiplexFeature* methods are payable and refund excess Eth at the end of the call


Instead of publicly exposing 3 new methods, 2 of those got merged into a single one. Therefore, the new implementations
expose 2 methods.
For simplicity of debugging with the Typescript test suite, the two methods have been assigned different names, specifically:
`batchMultiplex` and `batchMultiplexOptionalParams` the BatchMultiplexFeature.sol contract; `batchMultiplexV2` and `batchMultiplexV2OptionalParams` for the BatchMultiplexFeatureV2.sol contract. As Foundry tests have been implemented and it is expected Foundry will become the default test framework for new features, the 2 methods could have the same name, and will still result in different method selectors.

### Any gotchas or edge cases that were unexpected and needed extra/special handling
When native ETH is involved, it is not possible to correclty forward the individual calls to the 0x Exchange proxy. Our implementation, in fact, relies on using the low-level `delegatecall` call, which preserves msg.sender and msg.value. In the 0x Exchange V4, ETH-related methods

The `multiplexMultiSellETH` and `ff` methods have not been implemented

TODO: in this section add specs of *BatchMultiplexFeature*, with description of additions to proxy storage, calls, delegatecalls, ... , when delegatecall is safe, and is used as fallback unless a method requires special routing.
blacklisted methods, whitelisted methods, call of publicly exposed internal methods, which MUST NOT be called arbitrarily.

As some methods require custom routing, we have to keep information about the individual methods in storage. In order to do this we define 3 possible states for a selector: `Whitelisted`, `Blacklisted` and `RequiresRouting`. We define this in *src/storage/LibBatchMultiplexStorage.sol* as a mapping of status by selector. The mapping is updated by calling `updateSelectorsStatus(UpdateSelectorStatus[] memory selectorsTuple)`, which is restricted to the 0x Exchange proxy owner. When a method is whitelisted, its selector is removed from the mapping, so that all methods but the ones that require special handling are whitelisted. This flow is efficient as it avoids having to whitelist all implemented methods, thus not requiring an update when new features are added to the proxy. An external method `getSelectorStatus` allows investigate the status of a selector. Routing follows the same principles as the *MultiplexFeature* and uses some of its code with necessary modifications.
At point of extension registration to the 0x Exchange proxy, a few methods are registered as follows:
  1. blacklisted methods
    - Meta transactions not supported by default; MetaTransactionsV2 could be supported without need for routing:
      - executeMetaTransactionV2
      - batchExecuteMetaTransactionsV2
      - executeMetaTransaction
      - batchExecuteMetaTransactions
    - Batch NFT swaps:
      - batchBuyERC721s
      - batchBuyERC1155s
    - remaining blacklisted methods:
      - fillOtcOrderWithEth
      - sellEthForTokenToUniswapV3
      - sellToPancakeSwap
      - sellToUniswap
      - fillOrKillLimitOrder
      - batchFillLimitOrders

  2. methods that require special routing and will revert if not explicitly handled by the feature.
    - Multiplex payable methods that are not supported but could if properly handled (requires passing individual values):
      - multiplexBatchSellEthForToken
      - multiplexMultiHopSellEthForToken
    - supported methods:
      - sellToLiquidityProvider
      - transformERC20
      - buyERC721
      - buyERC1155
      - fillLimitOrder

Batch methods are blacklisted as they can be executed as an array of individual calls.
Some methods are prevented access but could be supported. Meta transactions V2 methods, for example, are non-payable and could be supported without need for routing. However, meta transactions in general are blacklisted as a batch swap can be executed as a single meta transaction.

We now dig deeper into why the remaining methods are blacklisted or routed through a custom path. It is to be noted that the actions resulting from the execution of some of the blacklisted methods can be alternatively executed through the use of a methods that requires custom routing, therefore allowing a "synthetic" execution and not necessarily requiring custom support, unless proven by demand for a specific method.

- Blacklisted methods
  1. fillOtcOrderWithEth
    - The method will wrap attached value into WETH, and then it will [try to refund](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/OtcOrdersFeature.sol#L169C58-L169C77) the difference between `msg.value` and `takerTokenFilledAmount`, therefore preventing other calls from using attached ETH. Technically, we could support the feature by wrapping `takerTokenFilledAmount`, route the call to `_fillOtcOrder`, unwrap and withdraw excess ETH (if any). However, the decision of whether to implement custom routing for this method should be based on demand for the feature. The same reasoning goes for `fillTakerSignedOtcOrderForEth`. TODO: check why the latter was not included in the methods registration.

  2. sellEthForTokenToUniswapV3
    - The method will try to wrap `msg.value` into WETH, which will revert if invoked multiple times. The method could be implemented by wrapping `sellAmount` and routing to `_sellTokenForTokenToUniswapV3`, however the feature can be executed by use of the `sellToLiquidityProvider` method in the *LiquidityProviderFeature*.

  3. sellToPancakeSwap
    - When trying to use this feature to swap tokens, but ETH has been attached to the call (i.e. for invoking other methods using ETH), the call [will revert](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/PancakeSwapFeature.sol#L190C31-L190C40). The call could be correctly executed with custom routing, but executing a `call` to `sellToPancakeSwap` would require adding some custom logic and, as the *PancakeSwapFeature* is entirely written in assembly, it is easy prey to error. The method can be executed by routing to `sellToLiquidityProvider` instead.

  4. sellToUniswap
    Same as previous point.

  5. fillOrKillLimitOrder
    - The method will try to [refund attached value](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/native_orders/NativeOrdersSettlement.sol#L204). Technically it would be possible to route to `_fillLimitOrder`, but decision to exclude was based on not identifying clear demand for the feature.

  6. multiplexBatchSellEthForToken
    This method will deposit attached ETH into WETH, which will lead to errors when other calls in the batch array try to use attached ETH. This method could be implemented by slightly modifying the interface of the *BatchMultiplexFeature.batchMultiplex(...params[])* by adding an additional param `uint256[] values` to *BatchMultiplexFeature.batchMultiplex(...params[], values[])*, wrapping passed `value` and routing to the `_multiplexBatchSell` internal method. This will not substantially change how the transaction is encoded at an API level, as the `value` param is attached to the transaction call separate to the `calldata`. We seek feedback from the community on this specification, as this will also result in a different interface of the *BatchMultiplexFeature* and the *BatchMultiplexFeatureV2*. To be clear, this would not require an upgrade in the *MultiplexFeature*, from which much of the inspiration for this feature has come and some of its code been reused with proper adaptations, where necessary.

  7. multiplexMultiHopSellEthForToken
    Same as previous point.

- Methods that require routing.
  1. sellToLiquidityProvider
    The call tries to deposit attached ETH into WETH, which will lead to errors when other calls looking to use attached ETH are included in the batch array. Therefore, whenever ETH is involved, we forward the call to the same method by overwriting `value` as `sellAmount` instead of the original call's `msg.value`.

  2. transformERC20
    Same issue as in previous point. Whenever ETH is involved, we forward the call to `_transformERC20` by overwriting `value` as `args.inputTokenAmount` and `msg.sender` as `taker`.

  3. buyERC721
    When ETH in involved, [balance](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/nft_orders/ERC721OrdersFeature.sol#L111) will underflow with multiple calls in the batch array accessing attached ETH. This will result to reverts as the underlying feature will try to refund excess ETH. In order for the call to execute correctly, we need to *delegatecall* `_buyERC721` and pass `address(this).balance` as available balance parameter.

  4. buyERC1155
    Same as previous but *delegatecall* to `_buyERC1155`.

  5. fillLimitOrder
    The method will try to [refund the protocol fee](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/native_orders/NativeOrdersSettlement.sol#L140C24-L140C55). While the protocol fee has been suspended, any unspent ETH is seen as excess protocol fee and its refund will lead to reverts when multiple calls in the batch array try to access attached ETH. In order for the call to execute correctly, the call is forwarded to `_fillLimitOrder` with 0 value and `msg.sender` as both `sender` and `taker`.


Some methods are flagged as blacklisted, but could be implemented with a custom routing. The choice has been made to prefer minimizing the number of custom features in order to reduce the error surface. Generally speaking, it is advisable to start by including the minimum amount of supported methods based on the current demand for the features, and subsequently add more as it becomes clearer what features should be included.


### Any other learning/difficulties related to the code/protocol or testing
When forwarding an array of calls to the 0x Exchange proxy, a developer must be aware that:
- `delegatecall` is a safe method to use, but will revert in some specific circumstances, as the original call's `msg.value` is preserved
- some publicly exposed internal methods have been developed, which allow custom routing of calls that require handling. These methods allow forwarding desired ETH.
- using `call` to forward a batch of swaps in generally unsafe as:
  - the `msg.sender` is changed to the 0x Exchange proxy itself, and therefore some external methods will be incorrectly executed
  - when calling the publicly exposed internal methods, some parameters will be passed to the child, which would make the 0x Exchange proxy insecure. In particular, in some cases a `taker` can be passed which is different from the `msg.sender`, while in others a `payer` can be passed which is different from the msg.sender. While the first does not pose a security thred but "violates" the rule of the 0x protocol, the second poses a security thread as anyone would be able to move another user's funds (provided the user has an approval set to the 0x Exchange proxy). Furthermore, the ABI of the publicly exposed internal methods is different from the ABI of the external methods that use them, thus requiring an amendment in the format at an API-provider level.
- a method cannot pass arbitrary data with `call` to self. Instead, inputs MUST be decoded according to the expected type and correctly routed with parameters validation.
- generally speaking, it is good practice to:
  - decode the encoded data according to their selector (first four bytes of the calldata)
  - optionally revert on unsupported methods. This is not helpful for debugging, as these transactions are blocked because they would otherwise fail
  - correctly route calls, i.e.:
    - if a transaction requires special encoding:
      - if it is an NFT call, `delegatecall` to self with correct input parameters
      - if not ETH is involved, `delegatecall` to self
      - `call` the publicly exposed internal methods (defined by the `onlySelf` modifier) with the correctly transformed input parameters
  - `delegatecall` to self otherwise.

This will allow to safely route the swaps to the internal methods, AND to maintain the 0x API order format.

## Notes
Calls to publicly exposed internal methods could skip pre-checks of the external methods, i.e. modifiers. Therefore, the batch multiplex methods must enforce those same assertions.
A `nonReentrant` modifier has been added to the *BatchMultiplexFeature*, as it skips the same high-level check when performing publicly exposed internal methods. The same modifier has not been implemented in the BatchMultiplexFeatureV2, as the calls get individually forwarded back to the 0x Exchange proxy, where the individual methods implement a non-reentrant modifier where needed.
The more complex batch multiplex implementation, which allows using methods involving ETH, as the call is sometimes forwarded to an internal method, using a no-reentrancy lock is probably helpful.
The upcoming Ethereum Cancun hardfork will activate [EIP1553](https://eips.ethereum.org/EIPS/eip-1153), which will make non-reentrancy modifiers way less gas expensive (~15k gas per reentrancy lock). While this gas optimization could be helpful at this stage, it could better a better fit for a general upgrade of all the reentrancy locks of the 0x Exchange proxy, and is therefore considered out of scope.
For reference, the Ethereum Cancun upgrade is expected to go live on Mar 13th 2024 and Solidity v0.8.4 supports `tstore`
and `tload` opcodes in inline assembly (compiler will display a warning when using transient storage).
In order to validate a batch of swaps the feature calls an arbitrary smart contract with a `staticcall`. In order to prevent denial of service attacks, it might be desirable to set a gas limit to the call, where the limit should be 2100 gas for accessing the external contract the first time, plus a stipend for executing validation.
For the purpose of maximum simplicity in the BatchMultiplexFeatureV2, the calldata validation in an external contract has
been omitted. Therefore, the feature allows executing a batch of swaps and optionally define `ErrorBehavior`.
A `onlyDelegateCall` modifier has been added to both implementations to prevent direct calls to the implementations but is not stricly necessary. Removing it would result in only marginal gas savings.
Both implementations are deemed safe in regards to new protocol features, as calls with unknown selectors will be forwarded to self with `delegatecall`.
Both implementations are deemed safe in the context of meta transactions, as the 0x Exchange always validates the maker's signatures, i.e. the 0x Exchange does not use "trusted forwarders". If, However, a trusted forwarder were allowed to settle using the 0x Exchange and skip signatures validation, the batch multiplex implementation should be amended to overwrite the sender. This issue is explained in [this post](https://blog.openzeppelin.com/arbitrary-address-spoofing-vulnerability-erc2771context-multicall-public-disclosure) about ERC-2771 arbitrary address spoofing attacks.

## Conclusions
While pZEIP-2 was intended to put more emphasis on swaps of batches of NFTs and ERC20s, substantial work has been required on the ERC20-related methods. While a simpler *BatchMultiplexFeatureV2* contract allows tokens to tokens swaps as per the proposal's specs, the decision has been made to invest substantial time on debugging every single transaction for allowing full coverage of the 0x Exchange features (thus including native-ETH transactions). Two different implementations have been written, so that the community can now compare the two proposed solutions, as they both offer advantages and disadvantages. The *BatchMultiplexFeatureV2* allows executing multiple swaps according to specs and has the advantage of being very simple, but excludes transactions using native ETH; should the underlying ETH-related methods be upgraded, the feature will require upgrading as its methods should now be made `payable`. The *BatchMultiplexFeature* is way more complex, is still not feature-complete (a few exchange methods are blacklisted) and requires an audit, as it routes external calls to publicly-exposed internal methods. In this context, the community should indicate whether the added complexity justifies the extra functionality.
