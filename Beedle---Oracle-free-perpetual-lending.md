# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Medium Risk Findings
    - ### [M-01. Vulnerable to Reentrancy Attack](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 1
   - Low: 0


		
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








