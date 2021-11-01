# reconfigureFundingCyclesOf

Contract: [`JBController`](broken-reference)​‌

Interface: [`IJBController`](../../../../interfaces/ijbcontroller.md)

{% tabs %}
{% tab title="Step by step" %}
**Configures the properties of the current funding cycle if the project hasn't distributed tokens yet, or sets the properties of the proposed funding cycle that will take effect once the current one expires if it is approved by the current funding cycle's ballot.**

_Only a project's owner or a designated operator can configure its funding cycles._

# Definition

```solidity
function reconfigureFundingCyclesOf(
  uint256 _projectId,
  JBFundingCycleData calldata _data,
  JBFundingCycleMetadata calldata _metadata,
  JBOverflowAllowance[] memory _overflowAllowances,
  JBSplit[] memory _payoutSplits,
  JBSplit[] memory _reservedTokenSplits
)
  external
  requirePermission(projects.ownerOf(_projectId), _projectId, JBOperations.RECONFIGURE)
  returns (uint256) { ... }
```

* Arguments:
  * `_projectId` is the ID of the project whose funding cycles are being reconfigured.
  * `_data` is a [`JBFundingCycleData`](../../../../data-structures/jbfundingcycledata.md) data structure that defines the project's funding cycle that will be queued. These properties will remain fixed for the duration of the funding cycle.
  * `_metadata` is a [`JBFundingCycleMetadata`](../../../../data-structures/jbfundingcyclemetadata.md) data structure specifying the controller specific params that a funding cycle can have.
  * `_overflowAllowances` is an array of [`JBOverflowAllowance`](../../../../data-structures/jboverflowallowance.md) data structures containing amounts, in wei (18 decimals), that a project can use from its own overflow on-demand for each payment terminal.
  * `_payoutSplits` is an array of [`JBSplit`](../../../../data-structures/jbsplit.md) data structures containing payout splits to set.
  * `_reservedTokenSplits` is an array of [`JBSplit`](../../../../data-structures/jbsplit.md) data structures containing reserved token splits to set.
* Through the [`requirePermission`](../../or-abstract/jboperatable/modifiers/requirepermission.md) modifier, the function is only accessible by the project's owner, or from an operator that has been given the `JBOperations.RECONFIGURE` permission by the project owner for the provided `_projectId`.
* The function returns the ID of the funding cycle that was configured.

# Body

1.  Make sure the reserved rate is a valid number out of 10000.

    ```solidity
    // The reserved project token rate must be less than or equal to 10000.
    require(_metadata.reservedRate <= 10000, '0x37: BAD_RESERVED_RATE');
    ```

2.  Make sure the redemption rate is a valid number out of 10000.

    ```solidity
    // The redemption rate must be between 0 and 10000.
    require(_metadata.redemptionRate <= 10000, '0x38: BAD_REDEMPTION_RATE');
    ```

3.  Make sure the ballot redemption rate is less than 100%.

    ```solidity
    // The ballot redemption rate must be less than or equal to 10000.
    require(_metadata.ballotRedemptionRate <= 10000, '0x39: BAD_BALLOT_REDEMPTION_RATE');
    ```

4.  Validate and pack the provided metadata into a `uint256`.

    ```solidity
    // Make sure the metadata is validated and packed into a uint256.
    uint256 _packedMetadata = JBFundingCycleMetadataResolver.packFundingCycleMetadata(
      _metadata
    );
    ```

    _Libraries used:_

    * [`JBFundingCycleMetadataResolver`](../../../../libraries/jbfundingcyclemetadataresolver.md)\
        `.packFundingCycleMetadata(...)`

5.  Configure the project's funding cycle, overflow allowances, and splits.

    ```solidity
    return
      _configure(
      _projectId,
      _data,
      _packedMetadata,
      _overflowAllowances,
      _payoutSplits,
      _reservedTokenSplits
    );
    ```

    _Internal references:_

    * [`_configure`](\_configure.md)
{% endtab %}

{% tab title="Code" %}
```solidity
/**
  @notice
  Configures the properties of the current funding cycle if the project hasn't distributed tokens yet, or sets the properties of the proposed funding cycle that will take effect once the current one expires if it is approved by the current funding cycle's ballot.

  @dev
  Only a project's owner or a designated operator can configure its funding cycles.

  @param _projectId The ID of the project whose funding cycles are being reconfigured.
  @param _data The funding cycle configuration data. These properties will remain fixed for the duration of the funding cycle.
    @dev _data.target The amount that the project wants to payout during a funding cycle. Sent as a wad (18 decimals).
    @dev _data.currency The currency of the `target`. Send 0 for ETH or 1 for USD.
    @dev _data.duration The duration of the funding cycle for which the `target` amount is needed. Measured in days. Send 0 for cycles that are reconfigurable at any time.
    @dev _data.discountRate A number from 0-10000 indicating how valuable a contribution to this funding cycle is compared to previous funding cycles.
      If it's 0, each funding cycle will have equal weight.
      If the number is 9000, a contribution to the next funding cycle will only give you 10% of tickets given to a contribution of the same amoutn during the current funding cycle.
      If the number is 10001, an non-recurring funding cycle will get made.
    @dev _data.ballot The ballot contract that will be used to approve subsequent reconfigurations. Must adhere to the IFundingCycleBallot interface.
  @param _metadata A struct specifying the controller specific params that a funding cycle can have.
    @dev _metadata.reservedRate A number from 0-10000 (0-100%) indicating the percentage of each contribution's newly minted tokens that will be reserved for the token splits.
    @dev _metadata.redemptionRate The rate from 0-10000 (0-100%) that tunes the bonding curve according to which a project's tokens can be redeemed for overflow.
      The bonding curve formula is https://www.desmos.com/calculator/sp9ru6zbpk
      where x is _count, o is _currentOverflow, s is _totalSupply, and r is _redemptionRate.
    @dev _metadata.ballotRedemptionRate The redemption rate to apply when there is an active ballot.
    @dev _metadata.pausePay Whether or not the pay functionality should be paused during this cycle.
    @dev _metadata.pauseWithdrawals Whether or not the withdraw functionality should be paused during this cycle.
    @dev _metadata.pauseRedeem Whether or not the redeem functionality should be paused during this cycle.
    @dev _metadata.pauseMint Whether or not the mint functionality should be paused during this cycle.
    @dev _metadata.pauseBurn Whether or not the burn functionality should be paused during this cycle.
    @dev _metadata.allowTerminalMigration Whether or not the terminal migration functionality should be paused during this cycle.
    @dev _metadata.allowControllerMigration Whether or not the controller migration functionality should be paused during this cycle.
    @dev _metadata.holdFees Whether or not fees should be held to be processed at a later time during this cycle.
    @dev _metadata.useDataSourceForPay Whether or not the data source should be used when processing a payment.
    @dev _metadata.useDataSourceForRedeem Whether or not the data source should be used when processing a redemption.
    @dev _metadata.dataSource A contract that exposes data that can be used within pay and redeem transactions. Must adhere to IJBFundingCycleDataSource.
  @param _overflowAllowances An array contraining amounts, in wei (18 decimals), that a project can use from its own overflow on-demand for each payment terminal.
  @param _payoutSplits An array of payout splits to set.
  @param _reservedTokenSplits An array of reserved token splits to set.

  @return The ID of the funding cycle that was successfully configured.
*/
function reconfigureFundingCyclesOf(
  uint256 _projectId,
  JBFundingCycleData calldata _data,
  JBFundingCycleMetadata calldata _metadata,
  JBOverflowAllowance[] memory _overflowAllowances,
  JBSplit[] memory _payoutSplits,
  JBSplit[] memory _reservedTokenSplits
)
  external
  requirePermission(projects.ownerOf(_projectId), _projectId, JBOperations.RECONFIGURE)
  returns (uint256)
{
  // The reserved project token rate must be less than or equal to 10000.
  require(_metadata.reservedRate <= 10000, '0x37: BAD_RESERVED_RATE');

  // The redemption rate must be between 0 and 10000.
  require(_metadata.redemptionRate <= 10000, '0x38: BAD_REDEMPTION_RATE');

  // The ballot redemption rate must be less than or equal to 10000.
  require(_metadata.ballotRedemptionRate <= 10000, '0x39: BAD_BALLOT_REDEMPTION_RATE');

  // Make sure the metadata is validated and packed into a uint256.
  uint256 _packedMetadata = JBFundingCycleMetadataResolver.packFundingCycleMetadata(
    _metadata
  );

  return
    _configure(
      _projectId,
      _data,
      _packedMetadata,
      _overflowAllowances,
      _payoutSplits,
      _reservedTokenSplits
    );
}
```
{% endtab %}

{% tab title="Errors" %}
| String                                 | Description                                                |
| -------------------------------------- | ---------------------------------------------------------- |
| **`0x51: BAD_RESERVED_RATE`**          | Thrown if the reserved rate is greater than 100%.          |
| **`0x52: BAD_REDEMPTION_RATE`**        | Thrown if the redemption rate is greater than 100%.        |
| **`0x53: BAD_BALLOT_REDEMPTION_RATE`** | Thrown if the ballot redemption rate is greater than 100%. |
{% endtab %}

{% tab title="Events" %}
| Name                                                                                | Data                                                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**`DistributeReservedTokens`**](../events/distributereservedtokens.md)             | <ul><li><code>uint256 indexed fundingCycleId</code></li><li><code>uint256 indexed projectId</code></li><li><code>address indexed beneficiary</code></li><li><code>uint256 count</code></li><li><code>uint256 projectOwnerTokenCount</code></li><li><code>string memo</code></li><li><code>address caller</code></li></ul> |
| [**`DistributeToReservedTokenSplit`**](../events/distributetoreservedtokensplit.md) | <ul><li><code>uint256 indexed fundingCycleId</code></li><li><code>uint256 indexed projectId</code></li><li><a href="../../../data-structures/jbsplit.md"><code>JBSplit</code></a><code>split</code></li><li><code>uint256 tokenCount</code></li><li><code>address caller</code></li></ul>                                 |
| [**`SetOverflowAllowance`**](../events/setoverflowallowance.md)                     | <ul><li><code>uint256 indexed projectId</code></li><li><code>uint256 indexed configuration</code></li><li><a href="../../../data-structures/jboverflowallowance.md"><code>JBOverflowAllowance</code></a><code>allowance</code></li><li><code>address caller</code></li></ul>                                              |
{% endtab %}

{% tab title="Bug bounty" %}
| Category          | Description                                                                                                                            | Reward |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| **Optimization**  | Help make this operation more efficient.                                                                                               | 0.5ETH |
| **Low severity**  | Identify a vulnerability in this operation that could lead to an inconvenience for a user of the protocol or for a protocol developer. | 1ETH   |
| **High severity** | Identify a vulnerability in this operation that could lead to data corruption or loss of funds.                                        | 5+ETH  |
{% endtab %}
{% endtabs %}