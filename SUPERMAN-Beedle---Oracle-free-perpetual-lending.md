# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Vulnerable to Reentrancy attack](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Vulnerable to Reentrancy Attack](#M-01)
    - ### [M-02. unchecked-transfer](#M-02)
    - ### [M-03. unchecked-transfer](#M-03)
    - ### [M-04. arbitrary-send-erc20 (NO authorization)](#M-04)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 4
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Vulnerable to Reentrancy attack            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L402-L432

## Vulnerability Details
The vulnerability occurs because the `giveLoan` function performs an external call to `IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest)` before updating the state variables. If the external contract being called (i.e., `IERC20(loan.loanToken)`) triggers a reentrant call back into the `giveLoan` function or any other function in the contract before the state variables are updated.

## Impact
The impact of the reentrancy vulnerability in the `giveLoan` function could lead to potential loss of funds or even manipulation of state variables during loan processing due to an external contract calling back into the function before the state is updated, allowing malicious actors to exploit the contract.

## Tools Used
Slither

## Recommendations or Mitigation
To mitigate this vulnerability, the checks-effects-interactions pattern should be followed to ensure the state is updated before any external calls are made. 


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Vulnerable to Reentrancy Attack            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L642-L674

## Vulnerability Details
The issue is due to external calls to other (IERC20(loan.loanToken).transferFrom(...), IERC20(loan.loanToken).transfer(...), IERC20(loan.collateralToken).transferFrom(...), and IERC20(loan.collateralToken).transfer(...)) being made before updating the state variables in the contract (loans[loanId].debt, loans[loanId].collateral, loans[loanId].interestRate, loans[loanId].startTimestamp, loans[loanId].auctionStartTimestamp, and loans[loanId].auctionLength).


## Impact
The impact of this reentrancy vulnerability is that an attacker could call back into the refinance function before the state variables are fully updated, allowing the attacker to manipulate the loan and pool data and lead to the loss of funds or other unexpected behaviours.

## Tools Used
Slither

## Recommendations and Mitigation: To fix this vulnerability, Perform all necessary state updates first. After updating the state, perform external calls.

For example, the fix should look like this:
function refinance(uint256 loanId, uint256 poolId) external nonReentrant {
    Refinance[] memory refinances = refinanceRequests[loanId];

    // Perform all state updates first (Checks and Effects)
    for (uint256 i = 0; i < refinances.length; i++) {
        uint256 debt = refinances[i].debt;
        uint256 collateral = refinances[i].collateral;

        // Update loan debt
        loans[loanId].debt = debt;

        // Update loan collateral
        loans[loanId].collateral = collateral;

        // Update loan interest rate
        loans[loanId].interestRate = pools[poolId].interestRate;

        // Update loan start timestamp
        loans[loanId].startTimestamp = block.timestamp;

        // Update loan auction start timestamp
        loans[loanId].auctionStartTimestamp = type(uint256).max;

        // Update loan auction length
        loans[loanId].auctionLength = pools[poolId].auctionLength;

        // Update loan lender
        loans[loanId].lender = pools[poolId].lender;

        // Update pool balance
        pools[poolId].poolBalance -= debt;

        // Emit events for state changes
        emit Repaid(
            msg.sender,
            loans[loanId].lender,
            loanId,
            debt,
            collateral,
            loans[loanId].interestRate,
            loans[loanId].startTimestamp
        );

        emit Borrowed(
            msg.sender,
            pools[poolId].lender,
            loanId,
            debt,
            collateral,
            pools[poolId].interestRate,
            block.timestamp
        );

        emit Refinanced(loanId);
    }

    // Perform external calls after state updates (Interactions)
    for (uint256 i = 0; i < refinances.length; i++) {
        // Get loan info and validate the loan
        Loan memory loan = loans[loanId];
        if (msg.sender != loan.borrower) revert Unauthorized();

        // Get pool info and validate the new loan
        Pool memory pool = pools[poolId];
        if (pool.loanToken != loan.loanToken) revert TokenMismatch();
        if (pool.collateralToken != loan.collateralToken) revert TokenMismatch();
        if (pool.poolBalance < debt) revert LoanTooLarge();
        if (debt < pool.minLoanSize) revert LoanTooSmall();
        uint256 loanRatio = (debt * 10 ** 18) / collateral;
        if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();

        // Calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(loan);
        uint256 debtToPay = loan.debt + lenderInterest + protocolInterest;

        // Update the old lender's pool
        _updatePoolBalance(oldPoolId, pools[oldPoolId].poolBalance + loan.debt + lenderInterest);
        pools[oldPoolId].outstandingLoans -= loan.debt;

        // Deduct tokens from the new pool
        _updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
        pools[poolId].outstandingLoans += debt;

        if (debtToPay > debt) {
            // We owe more in debt so the borrower needs to give us more loan tokens
            // Transfer the loan tokens from the borrower to the contract
            IERC20(loan.loanToken).transferFrom(
                msg.sender,
                address(this),
                debtToPay - debt
            );
        } else if (debtToPay < debt) {
            // We have excess loan tokens so we give some back to the borrower
            // First, take the borrower fee
            uint256 fee = (borrowerFee * (debt - debtToPay)) / 10000;
            IERC20(loan.loanToken).transfer(feeReceiver, fee);
            // Transfer the loan tokens from the contract to the borrower
            IERC20(loan.loanToken).transfer(msg.sender, debt - debtToPay - fee);
        }

        // Transfer the protocol fee to governance
        IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);

        if (collateral > loan.collateral) {
            // Transfer the collateral tokens from the borrower to the contract
            IERC20(loan.collateralToken).transferFrom(
                msg.sender,
                address(this),
                collateral - loan.collateral
            );
        } else if (collateral < loan.collateral) {
            // Transfer the collateral tokens from the contract to the borrower
            IERC20(loan.collateralToken).transfer(
                msg.sender,
                loan.collateral - collateral
            );
        }
    }
}

By following the "Checks-Effects-Interactions" pattern, you ensure that the state is updated correctly before any external calls are made, making the contract safer against reentrancy vulnerabilities



## <a id='M-02'></a>M-02. unchecked-transfer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L317C1-L327

## Vulnerability Details
In both cases, the contract calls the transferFrom function to transfer tokens from the msg.sender (the borrower) to another address (address(this) or feeReceiver). However, the code does not check the return values of these transfer functions.

## Impact
The impact of ignoring the return value of `transferFrom` functions is that the contract may not correctly account for token transfers, leading to consequences like the loss of tokens or incomplete transaction reversals on transfer failures, potentially resulting in financial loss

## Tools Used
Slither

## Recommendations or Mitigation
The mitigation for this issue is to handle the return values of the transferFrom functions using a require statement to check the success of the transfers and react appropriately if the transfer fails.

## <a id='M-03'></a>M-03. unchecked-transfer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L267

## Vulnerability Details
The contract transfers fees amount of tokens to the 'feeReceiver' address without checking the return value of the transfer function

## Impact
If the transfer function fails due to reasons like invalid transactions or insufficient balance, it can lead to unintended consequences. For example, the fees amount may not be correctly collected, affecting the revenue for the protocol.

## Tools Used
Slither

## Recommendations or mitigation: To mitigate this issue, first reduce contract size, you can consider using error codes or constants instead of string literals in revert statements. Error codes or constants are stored more efficiently in the contract's bytecode, leading to smaller contract sizes. Then handle the return value (or error codes) of the transfer function. Example shown below:
    require(
    IERC20(loan.loanToken).transfer(feeReceiver, fees),
    "Token transfer to feeReceiver failed"
);


## <a id='M-04'></a>M-04. arbitrary-send-erc20 (NO authorization)            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L152-L155

## Vulnerability Details
The 'transferFrom' function is being called without checking whether the p.lender address has approved the Lender contract to spend tokens on its behalf

## Impact
The vulnerability allows unauthorized token transfers from the p.lender address, leading to potential loss of funds for the user and exposing them to financial risks if exploited.
## Tools Used
Slither
## mitigation: To mitigate this issue, first reduce contract size; you can consider using error codes or constants instead of string literals in revert statements. Error codes or constants are stored more efficiently in the contract's bytecode, leading to smaller contract sizes. Then either use revert statement or error codes to ensure that the 'Lender' contract is authorised to transfer tokens on behalf of the p.lender address. This prevents unauthorized token transfers and also enhances the security of a contract.

Could look something like this; if (p.poolBalance > currentBalance) {
        // Ensure that the Lender contract is allowed to transfer tokens from p.lender
        uint256 transferAmount = p.poolBalance - currentBalance;
        require(
            IERC20(p.loanToken).allowance(p.lender, address(this)) >= transferAmount,
            "Lender not authorized to transfer tokens"
        );

OR

if (p.poolBalance > currentBalance) {
        // Ensure that the Lender contract is allowed to transfer tokens from p.lender
        uint256 transferAmount = p.poolBalance - currentBalance;
        require(
            IERC20(p.loanToken).allowance(p.lender, address(this)) >= transferAmount,
            ERR_NOT_AUTHORIZED
        );







