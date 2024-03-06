# pZEIP-1 - M2: Technical Details

Technical details on the 2 topics mentioned in [Technical Issues](../MILESTONE_2.md#technical-issues) of 
[pZEIP-1: Bundle Swap Feature (BSF) - M2](../MILESTONE_2.md).

## Issue 1: ERC20 vs native currency

### Topic

The proof of concept implementation has one entry point `bundleSwap()` which can handle ERC20 (ERC) and the native
currency (ETH) as in and output. For being able to support the native currency there is specific logic to handle the
difference between native and ERC20 tokens. Additional entry points that can either handle the native currency or just
ERC20 tokens could be implemented to optimize gas efficiency.

To handle both, ERC and ETH, the current implementation consists of different logic at points where the usage between
ERC and ETH makes a difference. For example:

```solidity
if (LibERC20Transformer.isTokenETH(swap.inputToken)) {
  (result.success, result.returnData) = address(this).call{value : swap.inputTokenAmount}(
    swap.data
  );
} else {
  (result.success, result.returnData) = address(this).call(
    swap.data
  );
}
```

Also receiving and returning needs different logic for ERC and ETH. All the time the bundle swap is just used for ERC
there is an overhead for the checks/logic mentioned above.

### Proposed Solution

Add additional entry point which can just handle ERC (not `payable`) to reduce gas consumption.

This additional entry point mitigates the issue of not being able to delegate calls from the BundleSwapFeature to sub
features. The reason why it is not possible in the first place is that one swap might use `sellTokenForTokenToUniswapV3`
which is not `payable`, however another swap might sell ETH. Therefore, ETH is sent with the tx and `delegatecall`
forwards
the ETH to all sub feature calls. In this case also to the not `payable` `sellTokenForTokenToUniswapV3` which leads to
an
error.
Lastly, this also mitigates the issue of arbitrary internal calls to the proxy. The issue is described in the section
below. Using `delegatecall` instead of `call` will resolve the issue since `delegatecall` will forward the external
address as
`msg.sender`. Call would change msg.sender to the proxies address which is an attack vector when combined with arbitrary
calldata.

## Issue 2: Arbitrtary internal calls to proxy

### Topic

During discussions from [SHA-2048](https://github.com/SHA-2048) and [gabririgo](https://github.com/gabririgo), related
to [pZEIP-2](../../2_BATCH_MULTIPLEX_FEATURE) (developed by gabririgo), it was figured out that internal arbitrary
calls back to the proxy can lead to harmful interactions since some internal functions are registered on the proxy and
could therefore be called by attackers to steal approved user funds.

For example the internal `_transformERC20()` function is registered on the proxy:

```solidity
_registerFeatureFunction(this._transformERC20.selector)
```

This is not an issue if it’s called by an external caller because there is a check if `msg.sender` equals the proxy
address. However, if a feature like the BSF, which is designed to get a list of swaps where each swap holds arbitrary
data to execute the subsequent swap, uses `call` to forward the arbitrary data to the proxy, the above-mentioned check
would pass, and it would be possible for an attacker to steal approved user funds.

### Potential Solutions

#### MultiplexFeature sanity checks

BSF forwards all calls to MultiplexFeature since this has already implemented sanity checks for sub-calls.

#### Implement own sanity checks

Implement sanity checks for the most important features directly in BSF and just support these. This would include:

* TransformERC20Feature
* NativeOrdersFeature
* OtcOrdersFeature
* UniswapFeature
* UniswapV3Feature

### Proposed Solution

Implement “MultiplexFeature sanity checks” since this bears less risk for the protocol as a whole because the
MultiplexFeature was already live for a long time and therefore battle tested.
