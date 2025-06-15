# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01.  Incorrect Balance Update Due to Scaled Amount Mismatch in Stability Pool Deposits](#H-01)
    - ### [H-02.  Users Receive Fewer ZENO Tokens Than Expected in `buy` Function](#H-02)
    - ### [H-03. Multiple Boost Delegations Exceeding User Balance Allowed ](#H-03)
    - ### [H-04. Tokens Received During NFT Minting Become Permanently Stuck ](#H-04)
    - ### [H-05.  Double Scaling of Token Amounts in `transfer` and `transferFrom` Functions](#H-05)
- ## Medium Risk Findings
    - ### [M-01. `removeBoostDelegation` Restricts Removal to Delegatee, Preventing Expired Delegation Cleanup](#M-01)
    - ### [M-02. Users Can Stake Tokens Even When Contract Is Paused](#M-02)
    - ### [M-03. Users Can Repay Debt Even After Liquidation Grace Period Expires](#M-03)
    - ### [M-04. Inconsistent Scaled Balance Calculations in `transfer` and `transferFrom`  ](#M-04)
    - ### [M-05. `updateUserBoost` Function Resets `workingSupply` Incorrectly, Leading to Data Inconsistencies](#M-05)
    - ### [M-06.  Redundant Self-Transfer in `emergencyRevoke` Function  ](#M-06)
- ## Low Risk Findings
    - ### [L-01. Incorrect Fee Allocation in `initializeFeeTypes` Function](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 5
- Medium: 6
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01.  Incorrect Balance Update Due to Scaled Amount Mismatch in Stability Pool Deposits            



## Summary

The `deposit` function in the Stability Pool incorrectly updates user balances based on the requested `amount` instead of the actual transferred amount. Since `rToken` operates with scaled balances, the recorded deposit amount can differ from what is actually received.

## Vulnerability Details

The function `deposit` uses:

```solidity
rToken.safeTransferFrom(msg.sender, address(this), amount);
userDeposits[msg.sender] += amount;
```

However, `rToken` scales transfers internally:

```solidity
function _update(address from, address to, uint256 amount) internal override {
    uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
    super._update(from, to, scaledAmount);
}
```

This discrepancy means the user's recorded deposit might be inaccurate.

## Impact

Users may have incorrect deposit balances, which can lead to miscalculations in rewards, withdrawals, or overall pool accounting.

## Tools Used

Manual code review.

## Recommendations

* Use the actual transferred amount to update deposits:
  ```solidity
  uint256 beforeBalance = rToken.balanceOf(address(this));
  rToken.safeTransferFrom(msg.sender, address(this), amount);
  uint256 afterBalance = rToken.balanceOf(address(this));
  uint256 actualReceived = afterBalance - beforeBalance;
  userDeposits[msg.sender] += actualReceived;
  ```
* Ensure that reward calculations use the correct deposit amounts.

## <a id='H-02'></a>H-02.  Users Receive Fewer ZENO Tokens Than Expected in `buy` Function            



## Summary

The `buy` function charges users based on the auction price (`cost = price * amount`) but mints only the `amount` of ZENO tokens instead of the full `cost`. According to the documentation, the ratio should be 1:1, meaning users should receive tokens equal to their USDC deposit.

## Vulnerability Details

```solidity
function buy(uint256 amount) external whenActive {
    require(amount <= state.totalRemaining, "Not enough ZENO remaining");
    uint256 price = getPrice();
    uint256 cost = price * amount;
    require(usdc.transferFrom(msg.sender, businessAddress, cost), "Transfer failed");

    bidAmounts[msg.sender] += amount;
    state.totalRemaining -= amount;
    state.lastBidTime = block.timestamp;
    state.lastBidder = msg.sender;

    zeno.mint(msg.sender, amount); // @audit-issue should mint cost, not amount
    emit ZENOPurchased(msg.sender, amount, price);
}
```

* Users pay `cost` in USDC but receive only `amount` in ZENO.
* If `price` > 1, users end up with fewer ZENO tokens than their deposited USDC.
* This contradicts the stated 1:1 redemption ratio, misleading participants.

## Impact

* Users suffer financial loss by receiving less than the documented amount.

## Tools Used

Manual code review

## Recommendations

Update the minting logic to reflect a 1:1 ratio:

```solidity
zeno.mint(msg.sender, cost); // Mint the full cost in ZENO tokens
```

## <a id='H-03'></a>H-03. Multiple Boost Delegations Exceeding User Balance Allowed             



## Summary

The `delegateBoost` function checks the user's veToken balance to validate the delegation amount but fails to consider previously active delegations. This oversight enables users to delegate the same veToken balance multiple times to different addresses, exceeding their actual voting power or boost capacity.

## Vulnerability Details

```solidity
function delegateBoost(
    address to,
    uint256 amount,
    uint256 duration
) external override nonReentrant {
    if (paused()) revert EmergencyPaused();
    if (to == address(0)) revert InvalidPool();
    if (to == msg.sender) revert CannotDelegateToSelf(); // Recommendation from [S-12]
    if (amount == 0) revert InvalidBoostAmount();
    if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION)
        revert InvalidDelegationDuration();

    uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
    if (userBalance < amount) revert InsufficientVeBalance();

    UserBoost storage delegation = userBoosts[msg.sender][to];
    if (delegation.amount > 0) revert BoostAlreadyDelegated();

    delegation.amount = amount;
    delegation.expiry = block.timestamp + duration;
    delegation.delegatedTo = to;
    delegation.lastUpdateTime = block.timestamp;

    emit BoostDelegated(msg.sender, to, amount, duration);
}
```

### Issue

* **Current check:** `userBalance < amount` ensures a single delegation does not exceed the user's veToken balance.
* **Missing check:** No consideration of existing active delegations from the same user.
* **Result:** A user with 100 veTokens can delegate 100 tokens to multiple addresses, surpassing their actual balance.

## Impact

* Users can amplify their influence or rewards beyond intended limits.
* Boost distribution integrity is compromised.
* Potential economic exploitation if boosts affect voting power, liquidity incentives, or rewards.

## Tools Used

Manual code review.

## Recommendations

Track total delegated amounts and ensure they do not exceed the user's balance:

```solidity
uint256 totalDelegated = getTotalDelegated(msg.sender);
if (userBalance < totalDelegated + amount) revert ExceedsAvailableDelegation();
```

Implement `getTotalDelegated` to sum all active delegations:

```solidity
function getTotalDelegated(address user) public view returns (uint256 total) {
    for (address delegate : delegates[user]) {
        UserBoost memory delegation = userBoosts[user][delegate];
        if (block.timestamp < delegation.expiry) {
            total += delegation.amount;
        }
    }
}
```

This ensures users cannot delegate more than their available veToken balance.

## <a id='H-04'></a>H-04. Tokens Received During NFT Minting Become Permanently Stuck             



## Summary

The `mint` function in `RAACNFT.sol` accepts ERC20 tokens as payment for NFTs but lacks a withdrawal mechanism. Since the contract is not upgradeable, the received funds become permanently locked, rendering them inaccessible.

## Vulnerability Details

```solidity
function mint(uint256 _tokenId, uint256 _amount) public override { 
    uint256 price = raac_hp.tokenToHousePrice(_tokenId); 
    if(price == 0) { revert RAACNFT__HousePrice(); }
    if(price > _amount) { revert RAACNFT__InsufficientFundsMint(); }

    token.safeTransferFrom(msg.sender, address(this), _amount); // @audit-issue: Tokens are accepted but cannot be withdrawn

    _safeMint(msg.sender, _tokenId);

    if (_amount > price) {
        uint256 refundAmount = _amount - price;
        token.safeTransfer(msg.sender, refundAmount);
    }

    emit NFTMinted(msg.sender, _tokenId, price);
}
```

### Issue

* The contract collects tokens but has no function to retrieve them.
* Being non-upgradeable prevents any future addition of a withdrawal function.
* Tokens remain locked indefinitely, causing potential fund loss.

## Impact

* Project owners lose access to user payments.

## Tools Used

Manual code review.

## Recommendations

Implement an owner-only withdrawal function:

```solidity
function withdraw(address to, uint256 amount) external onlyOwner {
    require(to != address(0), "Invalid address");
    require(token.balanceOf(address(this)) >= amount, "Insufficient funds");
    token.safeTransfer(to, amount);
}
```

Additionally, consider using a fee-collector contract to handle payments directly, reducing the risk of stuck funds.

## <a id='H-05'></a>H-05.  Double Scaling of Token Amounts in `transfer` and `transferFrom` Functions            



## Summary

The `RToken` contract scales token amounts in both the `transfer` and `transferFrom` functions and again within the internal `_update` function. This double scaling causes transferred amounts to be incorrectly calculated, allowing users to end up with more tokens than intended, potentially compromising the token’s economic integrity.

## Vulnerability Details

* **`transfer`** **function:**\
  Scales the amount using `getNormalizedIncome()`.
* **`transferFrom`** **function:**\
  Scales the amount using `_liquidityIndex`.
* **`_update`** **function:**\
  Scales the amount again regardless of prior scaling in the transfer functions.

### Code Example

```solidity
function transfer(address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
    uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome()); // First scaling
    return super.transfer(recipient, scaledAmount);
}

function _update(address from, address to, uint256 amount) internal override {
    uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome()); // Second scaling
    super._update(from, to, scaledAmount);
}
```

### Issue

* **First scaling**: Applied in `transfer` and `transferFrom`.
* **Second scaling**: Reapplied in `_update`, leading to the amount being scaled twice.
* Double scaling leads to token holders receiving disproportionately higher or lower token amounts depending on scaling factors.

## Impact

* Users can exploit this to inflate their token balances, draining protocol reserves.

## Tools Used

Manual code review.

## Recommendations

Ensure scaling is applied only once during transfers:

### Solution 1: Remove Scaling from `transfer` and `transferFrom`

Let `_update` handle all scaling to maintain consistency:

```solidity
function transfer(address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
    return super.transfer(recipient, amount); // No pre-scaling
}
```

### Solution 2: Remove Scaling from `_update`

If scaling should be handled in transfer functions, bypass scaling in `_update`:

```solidity
function _update(address from, address to, uint256 amount) internal override {
    super._update(from, to, amount); // No additional scaling
}
```

By adopting one consistent scaling approach, the protocol prevents unintended inflation or deflation of token balances.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `removeBoostDelegation` Restricts Removal to Delegatee, Preventing Expired Delegation Cleanup            



## Summary

The `removeBoostDelegation` function restricts delegation removal to the delegatee (`msg.sender`), which can become problematic if the delegatee refuses to remove themselves. As a result, expired delegations may persist indefinitely, causing state bloat and incorrect boost calculations.

## Vulnerability Details

Currently, the function checks:

```solidity
if (delegation.delegatedTo != msg.sender) revert DelegationNotFound();
```

This ensures that **only the delegatee** can remove the delegation. However, if the delegatee is uncooperative or malicious, the expired delegation remains in storage even after `delegation.expiry` has passed.

## Impact

* **Incorrect Reward Calculations:** Boost metrics may be skewed, potentially affecting fair reward distributions.
* **Loss of Control:** Users lose control over their delegations if the delegatee is unresponsive.

## Tools Used

Manual code review and state flow analysis.

## Recommendations

Allow anyone to remove an **expired delegation**, ensuring proper cleanup and consistent state.

### Suggested Fix

Replace:

```solidity
if (delegation.delegatedTo != msg.sender) revert DelegationNotFound();
```

With:

```solidity
if (delegation.expiry > block.timestamp) revert InvalidDelegationDuration();
```

This change:

* Permits anyone to remove delegations **only after expiry**, preventing abuse.
* Ensures timely cleanup of expired delegations.
* Restores user autonomy over their delegations.

## <a id='M-02'></a>M-02. Users Can Stake Tokens Even When Contract Is Paused            



## Summary

The `stake` function in `BaseGauge` does not enforce a pause check, allowing users to continue staking even when the contract is paused. This contradicts the expected behavior of a paused contract, where staking operations should be temporarily halted.

## Vulnerability Details

The function currently lacks a `whenNotPaused` modifier or a manual check for the paused state:

```solidity
function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
    if (amount == 0) revert InvalidAmount();
    _totalSupply += amount;
    _balances[msg.sender] += amount;
    stakingToken.safeTransferFrom(msg.sender, address(this), amount);
    emit Staked(msg.sender, amount);
}
```

## Impact

* **Users can unknowingly stake in a paused contract, leading to unexpected behavior.**
* **Potential security risks if pausing was meant to prevent further interactions due to an emergency situation.**
* **Violation of expected contract behavior, reducing trust and reliability.**

## Tools Used

Manual code review.

## Recommendations

Enforce a pause check using either:

1. Adding the `whenNotPaused` modifier:
   ```solidity
   function stake(uint256 amount) external nonReentrant whenNotPaused updateReward(msg.sender) {
   ```
2. Explicitly reverting if paused:
   ```solidity
   if (paused()) revert ContractPaused();
   ```

## <a id='M-03'></a>M-03. Users Can Repay Debt Even After Liquidation Grace Period Expires            



## Summary

The `repay` function allows users to repay their debt even when the liquidation grace period has passed and liquidation is pending finalization. This undermines the liquidation process, enabling users to bypass penalties and potentially exploit the system.

## Vulnerability Details

Currently, the `repay` function lacks a check to prevent repayments after liquidation has been initiated and the grace period has expired:

```solidity
function repay(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
    _repay(amount, msg.sender); // @audit-issue can repay debt even when liquidation has been initiated
}
```

### Problematic Scenario

* When a user becomes eligible for liquidation, a grace period begins.
* If the user fails to repay during this grace period, liquidation should proceed.
* **Issue:** Users can still call `repay` after the grace period but before `finalizeLiquidation` is triggered, negating the liquidation process.

## Impact

* **Bypassing Liquidation Penalties:** Users avoid penalties by repaying after grace period expiration.
* **Economic Exploit:** Borrowers can exploit this loophole to maintain risky positions without facing consequences.
* **Integrity Risk:** Undermines the protocol’s liquidation process and fairness.

## Tools Used

Manual code review.

## Recommendations

Add a validation check in the `repay` function to prevent repayments after the liquidation grace period expires:

```solidity
if (isUnderLiquidation[msg.sender] && block.timestamp > liquidationStartTime[msg.sender] + liquidationGracePeriod) {
    revert LiquidationGracePeriodExpired();
}
```

## <a id='M-04'></a>M-04. Inconsistent Scaled Balance Calculations in `transfer` and `transferFrom`              



## Summary

The `RToken` contract’s `transfer` and `transferFrom` functions use different scaling methods to calculate the transfer amounts. This inconsistency allows users to exploit whichever method results in a more favorable token amount, potentially causing discrepancies in token balances and unexpected economic behavior.

## Vulnerability Details

* **`transfer`** **function:**

  ```solidity
  function transfer(address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
      uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
      return super.transfer(recipient, scaledAmount);
  }
  ```

* **`transferFrom`** **function:**
  ```solidity
  function transferFrom(address sender, address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
      uint256 scaledAmount = amount.rayDiv(_liquidityIndex); // @audit-issue using transferFrom will return a different value than transfer()
      return super.transferFrom(sender, recipient, scaledAmount);
  }
  ```

### Issue

* The `transfer` function uses `ILendingPool(_reservePool).getNormalizedIncome()` for scaling.
* The `transferFrom` function uses `_liquidityIndex` for scaling.
* These two values may differ, resulting in transferred amounts that are inconsistent depending on which function is used.

## Impact

* Users can exploit the discrepancy to receive more tokens or pay fewer tokens.
* Potential loss of protocol funds or economic imbalances.

## Tools Used

Manual code review.

## Recommendations

Use a unified scaling method for both functions to maintain consistency:

* **Option 1:** Standardize both to use `getNormalizedIncome()`:

  ```solidity
  uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
  ```

* **Option 2:** Standardize both to use `_liquidityIndex` if that’s the intended approach:
  ```solidity
  uint256 scaledAmount = amount.rayDiv(_liquidityIndex);
  ```

This ensures that `transfer` and `transferFrom` provide consistent and predictable results.

## <a id='M-05'></a>M-05. `updateUserBoost` Function Resets `workingSupply` Incorrectly, Leading to Data Inconsistencies            



## Summary

The `updateUserBoost` function directly sets the `workingSupply` to the newly calculated boost value, which can lead to inconsistencies between `workingSupply` and `totalBoost`. When a user interacts with a supported pool for the first time, the `newBoost` can return a high default value (e.g., `10000`), causing the `workingSupply` to be incorrectly inflated.

## Vulnerability Details

When the function is called under these conditions:

* The user has never interacted with the specified pool (`userBoost.amount` defaults to `0`).
* The pool is already supported and has an existing `poolBoost` state.

The `newBoost` calculation can return a high value (e.g., `10000`) due to the default veToken calculation logic. This value is then directly assigned to `poolBoost.workingSupply` without considering the existing `poolBoost.totalBoost`.

### Problematic Code Segment

```solidity
poolBoost.workingSupply = newBoost; // @audit-issue workingSupply may be set to an inflated value (e.g., 10000)
```

### Example Scenario

1. **Initial State:**

   * `poolBoost.totalBoost = 10_000_000`
   * `poolBoost.workingSupply = 10_000_000`

2. **First-Time User Interaction:**
   * `oldBoost = 0` (user has no prior interaction)
   * `newBoost = 10000` (from `_calculateBoost`)
   * `poolBoost.totalBoost` updates to `10_010_000` correctly.
   * **However:**
     ```solidity
     poolBoost.workingSupply = newBoost; // Now set to 10,000 instead of maintaining the totalBoost-related proportion.
     ```
   * Result: `workingSupply` becomes **10,000**, while `totalBoost` is **10,010,000**, leading to data inconsistency.

## Impact

* **Data Inconsistency:** `workingSupply` becomes lower than `totalBoost`, potentially affecting calculations relying on accurate supply metrics.
* **Reward Distribution Errors:** Misalignment between `workingSupply` and `totalBoost` can distort pool-based calculations, affecting user rewards and overall system integrity.
* **Potential Manipulation:** Malicious actors may exploit this to alter the working supply for personal gain.

## Tools Used

Manual code review and logical analysis of state variable updates.

## Recommendations

Update the assignment of `workingSupply` to reflect accumulated boosts rather than resetting it:

### Suggested Fix

Instead of:

```solidity
poolBoost.workingSupply = newBoost;
```

Use:

```solidity
poolBoost.workingSupply = poolBoost.workingSupply + (newBoost - oldBoost);
```

This ensures that `workingSupply` is incremented or decremented relative to the previous boost values, maintaining consistency with `totalBoost`.

Alternatively, if `workingSupply` should mirror `totalBoost`:

```solidity
poolBoost.workingSupply = poolBoost.totalBoost;
```

This ensures both metrics remain aligned and avoids potential data inconsistencies.

## <a id='M-06'></a>M-06.  Redundant Self-Transfer in `emergencyRevoke` Function              



## Summary

The `emergencyRevoke` function in `RaacReleaseOchestrator.sol` performs an unnecessary self-transfer of `unreleasedAmount` tokens to the contract’s own address. Since the tokens are already held by the contract, this operation is redundant and serves no purpose, potentially confusing future maintainers.

## Vulnerability Details

### Code Snippet

```solidity
if (unreleasedAmount > 0) {
    raacToken.transfer(address(this), unreleasedAmount); // @audit-info Redundant self-transfer
    emit EmergencyWithdraw(beneficiary, unreleasedAmount);
}
```

## Impact

* redundancy of transfer.

## Recommendations

Remove the redundant transfer call:

```solidity
if (unreleasedAmount > 0) {
    emit EmergencyWithdraw(beneficiary, unreleasedAmount); // No need for transfer
}
```

This simplifies the function.


# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect Fee Allocation in `initializeFeeTypes` Function            



## Summary

The `initializeFeeTypes` function in the `FeeCollector` contract sets fee allocations in basis points (bps). However, both the Buy/Sell Swap Tax and NFT Royalty Fees exceed the intended total of 2%, violating the requirement that fee allocations must sum to 2% (200 bps).

## Vulnerability Details

The contract uses a `10000` bps system, where `100 bps = 1%`. Both fee configurations incorrectly sum to `2000 bps = 20%` instead of the required `200 bps = 2%`.

### Buy/Sell Swap Tax (Incorrect)

```solidity
feeTypes[6] = FeeType({
    veRAACShare: 500,     // 5%
    burnShare: 500,       // 5%
    repairShare: 1000,    // 10%
    treasuryShare: 0
}); // Total: 2000 bps = 20% (should be 200 bps = 2%)
```

### NFT Royalty Fees (Incorrect)

```solidity
feeTypes[7] = FeeType({
    veRAACShare: 500,     // 5%
    burnShare: 0,
    repairShare: 1000,    // 10%
    treasuryShare: 500    // 5%
}); // Total: 2000 bps = 20% (should be 200 bps = 2%)
```

## Impact

* **Overcharging Users:** Users are charged 20% instead of the intended 2%.
* **Misallocation of Funds:** Stakeholders receive incorrect fund distributions.
* **Protocol Compliance Risks:** Failing to meet stated fee parameters can harm trust and regulatory adherence.

## Tools Used

Manual code review and bps-to-percentage calculations.

## Recommendations

Update the fee values to ensure they total `200 bps = 2%`:

```solidity
feeTypes[6] = FeeType({
    veRAACShare: 50,      // 0.5%
    burnShare: 50,        // 0.5%
    repairShare: 100,     // 1.0%
    treasuryShare: 0
});

feeTypes[7] = FeeType({
    veRAACShare: 50,      // 0.5%
    burnShare: 0,
    repairShare: 100,     // 1.0%
    treasuryShare: 50     // 0.5%
});
```

This ensures compliance with the 2% total fee requirement.



