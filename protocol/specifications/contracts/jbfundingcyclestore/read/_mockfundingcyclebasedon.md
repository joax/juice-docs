# \_mockFundingCycleBasedOn

Contract:[`JBFundingCycleStore`](../)​

{% tabs %}
{% tab title="Step by step" %}
**A view of the funding cycle that would be created based on the provided one if the project doesn't approve a reconfiguration ahead of it starting.**

_Returns an empty funding cycle if there can't be a mock funding cycle based on the provided one._

# Definition

```solidity
function _mockFundingCycleBasedOn(JBFundingCycle memory _baseFundingCycle, bool _allowMidCycle)
  private
  view
  returns (JBFundingCycle memory) { ... }
```

* Arguments:
  * `_baseFundingCycle` is the [`JBFundingCycle`](../../../data-structures/jbfundingcycle.md) that the resulting funding cycle should follow.
  * `_allowMidCycle` is a flag indicating if the mocked funding cycle is allowed to already be mid cycle.
* The view function is private to this contract.
* The function does not alter state on the blockchain.
* The function returns a mock [`JBFundingCycle`](../../../data-structures/jbfundingcycle.md) of what the next funding cycle will be.

# Body

1.  A funding cycle with a `discountRate` of 1000000001 is a non-recurring funding cycle. An empty funding cycle should be returned if the base is non-recurring since there can't be subsequent cycles.

    ```solidity
    // Can't mock a non recurring funding cycle.
    if (_baseFundingCycle.discountRate == 1000000001) return _getStructFor(0, 0);
    ```

    _Internal references:_

    * [`_getStructFor`](\_getstructfor.md)
2.  Save a reference to the amount of seconds since right now that the returned funding cycle could have started at. There are a few possibilities.

    1. If the call to the function does not `_allowMidCycle`, the start date must be now or in the future. This is also the case if the base funding cycle doesn't have a duration because the next funding cycle can start immediately.
    2. If neither of these cases apply, moving back one full duration period of the `_baseFundingCycle` will find the most recent possible start time for the mock cycle to start.

    ```solidity
    // The distance of the current time to the start of the next possible funding cycle.
    // If the returned mock cycle must not yet have started, the start time of the mock must be in the future so no need to adjust backwards.
    // If the base funding cycle doesn't have a duration, no adjustment is necessary because the next cycle can start immediately.
    uint256 _timeFromImmediateStartMultiple = !_allowMidCycle || _baseFundingCycle.duration == 0
      ? 0
      : _baseFundingCycle.duration;
    ```

    _Internal references:_

    * [`_SECONDS_IN_DAY`](../properties/\_seconds\_in\_day.md)
3.  Find the correct start time for the mock funding cycle.

    ```solidity
    // Derive what the start time should be.
    uint256 _start = _deriveStartFrom(
      _baseFundingCycle,
      block.timestamp - _timeFromImmediateStartMultiple
    );
    ```

    _Internal references:_

    * [`_deriveStartFrom`](\_derivestartfrom.md)
4.  Find the correct number for the mock funding cycle.

    ```solidity
    // Derive what the number should be.
    uint256 _number = _deriveNumberFrom(_baseFundingCycle, _start);
    ```

    _Internal references:_

    * [`_deriveNumberFrom`](\_derivenumberfrom.md)
5.  Return a [`JBFundingCycle`](../../../data-structures/jbfundingcycle.md) with the aggregated configuration.

    ```solidity
    return
      JBFundingCycle(
        _number,
        _baseFundingCycle.configuration,
        _baseFundingCycle.basedOn,
        _start,
        _baseFundingCycle.duration,
        _deriveWeightFrom(_baseFundingCycle, _start),
        _baseFundingCycle.discountRate,
        _baseFundingCycle.ballot,
        _baseFundingCycle.metadata
      );
    ```

    _Internal references:_

    * [`_deriveWeightFrom`](\_deriveweightfrom.md)
{% endtab %}

{% tab title="Code" %}
```solidity
/** 
  @notice 
  A view of the funding cycle that would be created based on the provided one if the project doesn't make a reconfiguration.
 
  @dev
  Returns an empty funding cycle if there can't be a mock funding cycle based on the provided one.
  
  @param _baseFundingCycle The funding cycle that the resulting funding cycle should follow.
  @param _allowMidCycle A flag indicating if the mocked funding cycle is allowed to already be mid cycle.

  @return A mock of what the next funding cycle will be.
*/
function _mockFundingCycleBasedOn(JBFundingCycle memory _baseFundingCycle, bool _allowMidCycle)
  private
  view
  returns (JBFundingCycle memory)
{
  // Can't mock a non recurring funding cycle.
  if (_baseFundingCycle.discountRate == 1000000001) return _getStructFor(0, 0);
    
  // The distance of the current time to the start of the next possible funding cycle.
  // If the returned mock cycle must not yet have started, the start time of the mock must be in the future so no need to adjust backwards.
  // If the base funding cycle doesn't have a duration, no adjustment is necessary because the next cycle can start immediately.
  uint256 _timeFromImmediateStartMultiple = !_allowMidCycle || _baseFundingCycle.duration == 0
    ? 0
    : _baseFundingCycle.duration;
    
  // Derive what the start time should be.
  uint256 _start = _deriveStartFrom(
    _baseFundingCycle,
    block.timestamp - _timeFromImmediateStartMultiple
  );

  // Derive what the number should be.
  uint256 _number = _deriveNumberFrom(_baseFundingCycle, _start);

  return
    JBFundingCycle(
      _number,
      _baseFundingCycle.configuration,
      _baseFundingCycle.basedOn,
      _start,
      _baseFundingCycle.duration,
      _deriveWeightFrom(_baseFundingCycle, _start),
      _baseFundingCycle.discountRate,
      _baseFundingCycle.ballot,
      _baseFundingCycle.metadata
    );
}
```
{% endtab %}

{% tab title="Bug bounty" %}
| Category          | Description                                                                                                                            | Reward |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| **Optimization**  | Help make this operation more efficient.                                                                                               | 0.5ETH |
| **Low severity**  | Identify a vulnerability in this operation that could lead to an inconvenience for a user of the protocol or for a protocol developer. | 1ETH   |
| **High severity** | Identify a vulnerability in this operation that could lead to data corruption or loss of funds.                                        | 5+ETH  |
{% endtab %}
{% endtabs %}
