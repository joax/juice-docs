# projects

Contract: [`JBController`](broken-reference)​‌

Interface: [`IJBController`](../../../../interfaces/ijbcontroller.md)

**The Projects contract which mints ERC-721's that represent project ownership and transfers.**

## Definition

```solidity
/** 
  @notice 
  The Projects contract which mints ERC-721's that represent project ownership and transfers.
*/ 
IJBProjects public immutable override projects;
```

* The value cannot be changed.
* The resulting view function can be accessed externally by anyone.
* The resulting function overrides a function definition from the `IJBController` interface.