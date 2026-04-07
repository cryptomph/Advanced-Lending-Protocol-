# Advanced-Lending-Protocol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/token/ERC20/IERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/access/Ownable.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/security/ReentrancyGuard.sol";

contract BaseLendingProtocol is Ownable, ReentrancyGuard {
    IERC20 public borrowToken;          // Token users can borrow (e.g. USDC on Base)

    uint256 public collateralFactor = 150;   // 150% collateralization ratio (over-collateralized)
    uint256 public liquidationThreshold = 120; // Liquidate if ratio <= 120%
    uint256 public borrowInterestRate = 5;   // 5% annual interest (simplified)

    struct Loan {
        uint256 collateralAmount;   // ETH deposited as collateral
        uint256 borrowedAmount;     // Amount of borrowToken borrowed
        uint256 borrowTime;         // Timestamp when loan was taken
    }

    mapping(address => Loan) public loans;

    event CollateralDeposited(address indexed user, uint256 amount);
    event LoanBorrowed(address indexed user, uint256 borrowAmount);
    event LoanRepaid(address indexed user, uint256 repayAmount);
    event Liquidated(address indexed user, address liquidator, uint256 collateralSeized);
    event InterestRateUpdated(uint256 newRate);

    error InsufficientCollateral();
    error LoanNotActive();
    error Undercollateralized();
    error NoLoanToLiquidate();

    constructor(address _borrowToken) Ownable(msg.sender) {
        borrowToken = IERC20(_borrowToken);
    }

    // Deposit ETH as collateral
    function depositCollateral() external payable nonReentrant {
        require(msg.value > 0, "Must send ETH");
        loans[msg.sender].collateralAmount += msg.value;
        emit CollateralDeposited(msg.sender, msg.value);
    }

    // Borrow tokens against collateral
    function borrow(uint256 borrowAmount) external nonReentrant {
        Loan storage loan = loans[msg.sender];
        require(loan.collateralAmount > 0, "No collateral");

        uint256 maxBorrow = (loan.collateralAmount * 1e18 * 100) / collateralFactor; // Simplified price assumption (1 ETH = 1 unit)
        require(borrowAmount <= maxBorrow - loan.borrowedAmount, "Exceeds max borrow");

        loan.borrowedAmount += borrowAmount;
        loan.borrowTime = block.timestamp;

        borrowToken.transfer(msg.sender, borrowAmount);
        emit LoanBorrowed(msg.sender, borrowAmount);
    }

    // Repay borrowed tokens + accrued interest
    function repay(uint256 repayAmount) external nonReentrant {
        Loan storage loan = loans[msg.sender];
        if (loan.borrowedAmount == 0) revert LoanNotActive();

        uint256 interest = calculateInterest(msg.sender);
        uint256 totalDue = loan.borrowedAmount + interest;

        require(repayAmount <= totalDue, "Repay too much");

        borrowToken.transferFrom(msg.sender, address(this), repayAmount);

        if (repayAmount >= totalDue) {
            // Fully repaid
            payable(msg.sender).transfer(loan.collateralAmount); // Return all collateral
            delete loans[msg.sender];
        } else {
            loan.borrowedAmount = totalDue - repayAmount;
            loan.borrowTime = block.timestamp; // Reset interest clock
        }

        emit LoanRepaid(msg.sender, repayAmount);
    }

    // Calculate accrued interest (simplified yearly, pro-rated by time)
    function calculateInterest(address user) public view returns (uint256) {
        Loan memory loan = loans[user];
        if (loan.borrowedAmount == 0) return 0;

        uint256 timeElapsed = block.timestamp - loan.borrowTime;
        uint256 interest = (loan.borrowedAmount * borrowInterestRate * timeElapsed) / (365 days * 100);
        return interest;
    }

    // Get current collateral value in borrow token units (assuming 1 ETH = 2000 borrow units for demo)
    function getCollateralValue(address user) public pure returns (uint256) {
        // In production, integrate Chainlink price feed for real ETH price
        return loans[user].collateralAmount * 2000; // Demo price: 1 ETH = $2000
    }

    // Liquidate undercollateralized loan
    function liquidate(address borrower) external nonReentrant {
        Loan storage loan = loans[borrower];
        if (loan.borrowedAmount == 0) revert NoLoanToLiquidate();

        uint256 collateralValue = getCollateralValue(borrower);
        uint256 totalDebt = loan.borrowedAmount + calculateInterest(borrower);

        if (collateralValue * 100 < totalDebt * liquidationThreshold) {
            // Seize collateral
            uint256 collateralToSeize = loan.collateralAmount;
            delete loans[borrower];

            payable(msg.sender).transfer(collateralToSeize); // Liquidator gets collateral
            emit Liquidated(borrower, msg.sender, collateralToSeize);
        } else {
            revert Undercollateralized();
        }
    }

    // Admin functions
    function setInterestRate(uint256 newRate) external onlyOwner {
        require(newRate <= 20, "Rate too high");
        borrowInterestRate = newRate;
        emit InterestRateUpdated(newRate);
    }

    function withdrawToken(address token, uint256 amount) external onlyOwner {
        IERC20(token).transfer(owner(), amount);
    }

    // Allow contract to receive ETH
    receive() external payable {}
}
