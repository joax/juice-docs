# \_distributeToReservedTokenSplitsOf

{% tabs %}
{% tab title="Step by step" %}
**Distributed tokens to the splits according to the specified funding cycle configuration.**

# Definition

```solidity
function _distributeToReservedTokenSplitsOf(
  uint256 _projectId,
  JBFundingCycle memory _fundingCycle,
  uint256 _amount
) private returns (uint256 leftoverAmount) { ... }
```

* Arguments:
  * `_fundingCycle` is the [`JBFundingCycle`](../../../../data-structures/jbfundingcycle.md) to base the token distribution on.
  * `_amount` is the total amount of tokens to mint.
* The function is private to this contract.
* The function returns the leftover amount after all splits have been distributed.

# Body

1.  Save the passed in `_amount` as the `leftoverAmount` that will be returned. The subsequent routine will decrement the leftover amount as splits are settled.

    ```solidity
    // Set the leftover amount to the initial amount.
    leftoverAmount = _amount;
    ```
2.  Get a reference to reserved token splits for the current funding cycle configuration of the project.

    ```solidity
    // Get a reference to the project's reserved token splits.
    JBSplit[] memory _splits = splitsStore.splitsOf(
      _projectId,
      _fundingCycle.configuration,
      JBSplitsGroups.RESERVED_TOKENS
    );
    ```

    _External references:_

    * [`splitsOf`](../../../jbsplitsstore/read/splitsof.md)
3.  Loop through each split.

    ```solidity
    //Transfer between all splits.
    for (uint256 _i = 0; _i < _splits.length; _i++) { ... }
    ```
4.  Get a reference to the current split being iterated on.

    ```solidity
    // Get a reference to the split being iterated on.
    JBSplit memory _split = _splits[_i];
    ```
5.  Get a reference to the amount of tokens to distribute to the current split. This amount is the total amount multiplied by the percentage of the split, which is a number out of 10000000.

    ```solidity
    // The amount to send towards the split. JBSplit percents are out of 10000000.
    uint256 _tokenCount = PRBMath.mulDiv(_amount, _split.percent, 10000000);
    ```

    _Libraries used:_

    * [`PRBMath`](https://github.com/hifi-finance/prb-math/blob/main/contracts/PRBMath.sol)
      * `.mulDiv`
6.  If there are tokens to mint for the given split, do so. If the split has an `allocator` specified, the tokens should go to that address. Otherwise if the split has a `projectId` specified, the tokens should be directed to the project's owner. Otherwise, the tokens should be directed at the `beneficiary` address of the split. Afterwards, if there's an `allocator` specified, let it know that tokens have been sent. Reduce the leftover amount by the tokens that were sent to the split.

    ```solidity
    // Mints tokens for the split if needed.
    if (_tokenCount > 0) {
      tokenStore.mintFor(
        // If an allocator is set in the splits, set it as the beneficiary. Otherwise if a projectId is set in the split, set the project's owner as the beneficiary. Otherwise use the split's beneficiary.
        _split.allocator != IJBSplitAllocator(address(0))
          ? address(_split.allocator)
          : _split.projectId != 0
          ? projects.ownerOf(_split.projectId)
          : _split.beneficiary,
        _projectId,
        _tokenCount,
        _split.preferClaimed
      );

      // If there's an allocator set, trigger its `allocate` function.
      if (_split.allocator != IJBSplitAllocator(address(0)))
        _split.allocator.allocate(
          _tokenCount,
          JBSplitsGroups.RESERVED_TOKENS,
          _projectId,
          _split.projectId,
          _split.beneficiary,
          _split.preferClaimed
        );

      // Subtract from the amount to be sent to the beneficiary.
      leftoverAmount = leftoverAmount - _tokenCount;
    }
    ```

    _External references:_

    * [`mintFor`](../../../jbtokenstore/write/mintfor.md)
7.  Emit a `DistributeToReservedTokenSplit` event for the split being iterated on with the relevant parameters.

    ```solidity
    emit DistributeToReservedTokenSplit(
      _fundingCycle.configuration,
      _fundingCycle.number,
      _projectId,
      _split,
      _tokenCount,
      msg.sender
    );
    ```

    _Event references:_

    * [`DistributeToReservedTokenSplit`](../events/distributetoreservedtokensplit.md)
{% endtab %}

{% tab title="Only code" %}
```solidity
/**
  @notice
  Distributed tokens to the splits according to the specified funding cycle configuration.

  @param _projectId The ID of the project for which reserved token splits are being distributed.
  @param _fundingCycle The funding cycle to base the token distribution on.
  @param _amount The total amount of tokens to mint.

  @return leftoverAmount If the splits percents dont add up to 100%, the leftover amount is returned.
*/
function _distributeToReservedTokenSplitsOf(
  uint256 _projectId,
  JBFundingCycle memory _fundingCycle,
  uint256 _amount
) private returns (uint256 leftoverAmount) {
  // Set the leftover amount to the initial amount.
  leftoverAmount = _amount;

  // Get a reference to the project's reserved token splits.
  JBSplit[] memory _splits = splitsStore.splitsOf(
    _projectId,
    _fundingCycle.configured,
    JBSplitsGroups.RESERVED_TOKENS
  );

  //Transfer between all splits.
  for (uint256 _i = 0; _i < _splits.length; _i++) {
    // Get a reference to the split being iterated on.
    JBSplit memory _split = _splits[_i];

    // The amount to send towards the split. JBSplit percents are out of 10000000.
    uint256 _tokenCount = PRBMath.mulDiv(_amount, _split.percent, 10000000);

    // Mints tokens for the split if needed.
    if (_tokenCount > 0) {
      tokenStore.mintFor(
        // If an allocator is set in the splits, set it as the beneficiary. Otherwise if a projectId is set in the split, set the project's owner as the beneficiary. Otherwise use the split's beneficiary.
        _split.allocator != IJBSplitAllocator(address(0))
          ? address(_split.allocator)
          : _split.projectId != 0
          ? projects.ownerOf(_split.projectId)
          : _split.beneficiary, 
        _projectId,
        _tokenCount,
        _split.preferClaimed
      );

      // If there's an allocator set, trigger its `allocate` function.
      if (_split.allocator != IJBSplitAllocator(address(0)))
        _split.allocator.allocate(
          _tokenCount,
          JBSplitsGroups.RESERVED_TOKENS,
          _projectId,
          _split.projectId,
          _split.beneficiary,
          _split.preferClaimed
        );

      // Subtract from the amount to be sent to the beneficiary.
      leftoverAmount = leftoverAmount - _tokenCount;
    }

    emit DistributeToReservedTokenSplit(
      _fundingCycle.configuration,
      _fundingCycle.number,
      _projectId,
      _split,
      _tokenCount,
      msg.sender
    );
  }
}
```
{% endtab %}

{% tab title="Events" %}
| Name                                                                                | Data                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**`DistributeToReservedTokenSplit`**](../events/distributetoreservedtokensplit.md) | <ul><li><code>uint256 indexed fundingCycleId</code></li><li><code>uint256 indexed projectId</code></li><li><a href="../../../../data-structures/jbsplit.md"><code>JBSplit</code></a><code>split</code></li><li><code>uint256 tokenCount</code></li><li><code>address caller</code></li></ul> |
{% endtab %}

{% tab title="Bug bounty" %}
| Category          | Description                                                                                                                            | Reward |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| **Optimization**  | Help make this operation more efficient.                                                                                               | 0.5ETH |
| **Low severity**  | Identify a vulnerability in this operation that could lead to an inconvenience for a user of the protocol or for a protocol developer. | 1ETH   |
| **High severity** | Identify a vulnerability in this operation that could lead to data corruption or loss of funds.                                        | 5+ETH  |
{% endtab %}
{% endtabs %}
