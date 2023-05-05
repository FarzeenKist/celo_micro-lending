# A Developer’s Guide on Building a Celo-Based Microlending Platform for Financial Inclusion

## Table of Contents

- [A Developer’s Guide on Building a Celo-Based Microlending Platform for Financial Inclusion](#a-developers-guide-on-building-a-celo-based-microlending-platform-for-financial-inclusion)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Setting up the Development Environment](#setting-up-the-development-environment)
    - [Step 1: Create a New Project](#step-1-create-a-new-project)
    - [Step 2: Install Hardhat](#step-2-install-hardhat)
    - [Step 3: Configure Hardhat](#step-3-configure-hardhat)
    - [Step 4: Create a Celo Wallet](#step-4-create-a-celo-wallet)
    - [Step 5: Fund Your Wallet With Testnet cUSD Tokens](#step-5-fund-your-wallet-with-testnet-cusd-tokens)
  - [Creating the Microlending Smart Contract](#creating-the-microlending-smart-contract)
    - [Step 1: Create New Solidity Files](#step-1-create-new-solidity-files)
    - [Step 2: Compile the Smart Contract](#step-2-compile-the-smart-contract)
    - [Step 3: Deploy the Smart Contract](#step-3-deploy-the-smart-contract)
    - [Step 4: Interact With the Smart Contract](#step-4-interact-with-the-smart-contract)
    - [Step 5: Testing](#step-5-testing)
      - [Setting up Our Testing Environment](#setting-up-our-testing-environment)
      - [Writing the Unit Tests](#writing-the-unit-tests)
  - [Conclusion](#conclusion)

## Introduction

Microlending has been acknowledged as an important strategy for poverty alleviation and financial inclusion. It gives low-income people access to loans that can help them start enterprises, pay for education or healthcare, and satisfy other basic requirements. Traditional financial institutions, on the other hand, have frequently disregarded low-income persons due to a lack of collateral and credit history. With the advent of blockchain technology, it has become possible to create microlending platforms that use smart contracts to eliminate the need for middlemen, cut costs, and expand loan access.

**Celo** is a blockchain platform that aims to increase financial inclusion by allowing people to transfer, receive, and store digital assets on their mobile devices. It offers a stablecoin called Celo Dollar (cUSD), which is pegged to the US dollar and may be used for microlending. In this tutorial, we will use smart contracts to create a Celo-based microlending platform.

## Prerequisites

To follow this tutorial, you will need the following:

- A basic understanding of blockchain technology and smart contracts
- Familiarity with Solidity programming language
- An understanding of the web3.js library
- Node.js & Npm installed on your machine
- A Celo wallet and some testnet cUSD

## Setting up the Development Environment

We will use the following tool to develop our microlending platform:

- Hardhat: A development framework that provides tools for compiling, testing, and deploying smart contracts.

We will now go through the steps to set up our development environment.

### Step 1: Create a New Project

Create a new directory for your project:

```bash
    mkdir microlending-platform
```

Navigate to the project directory you just created:

```bash
    cd microlending-platform
```

### Step 2: Install Hardhat

Open your terminal and run the following commands to install Hardhat:

```bash
    npm install hardhat --save-dev
```

Next, install the dependecies we will need in our Hardhat setup:

```bash
npm install --save-dev @nomiclabs/hardhat-ethers ethers @nomiclabs/hardhat-waffle ethereum-waffle chai
```

Initialize a new Hardhat Project:

```bash
    npx hardhat init
```

This will create a new project similar to this structure:

```
    ├── contracts
    │   └── Lock.sol
    ├── test
    └── hardhat-config.js
```

### Step 3: Configure Hardhat

Open the `hardhat-config.js` file and configure it as follows:

```javascript
require("@nomiclabs/hardhat-waffle");
// Your mnemomic/seed phrase
const MNEMONIC = "";

// You need to export an object to set up your config
// Go to https://hardhat.org/config/ to learn more

/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
	solidity: {
		version: "0.8.4",
		settings: {
			optimizer: {
				enabled: true,
				runs: 1000,
			},
		},
	},
	networks: {
		localhost: {
			url: "http://127.0.0.1:7545",
		},
		alfajores: {
			url: "https://alfajores-forno.celo-testnet.org",
			accounts: {
				mnemonic: MNEMONIC,
				path: "m/44'/52752'/0'/0",
			},
			chainId: 44787,
		},
	},
};

```

`MNEMONIC` should be replaced with your Celo testnet seed/mnemonic phrase. This file defines two networks: alfajores, which connects to the Celo Alfajores testnet, and localhost, which connects to a local hardhat instance.

### Step 4: Create a Celo Wallet

A Celo wallet is required to engage with the Celo network. You can take advantage of the wallet extension provided by Celo which can be downloaded and installed on your local machine **[here](https://celowallet.app/setup)**

### Step 5: Fund Your Wallet With Testnet cUSD Tokens

We need some testnet cUSD to test our microlending platform. You can request testnet cUSD from the Celo Faucet by visiting this **[link](https://celo.org/developers/faucet)**

After submitting your wallet address, you should receive testnet cUSD within a few minutes. You can check your balance in your celo wallet.

## Creating the Microlending Smart Contract

In this section, we'll build a smart contract that lets lenders deposit cUSD and borrowers seek loans. The smart contract will handle loan repayment, including interest, and will send payments to lenders.

### Step 1: Create New Solidity Files

Create a new file called `Microlending.sol` in the contracts directory and add the following code:

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.4;

interface IERC20Token {
    function transfer(address, uint256) external returns (bool);

    function approve(address, uint256) external returns (bool);

    function transferFrom(address, address, uint256) external returns (bool);

    function totalSupply() external view returns (uint256);

    function balanceOf(address) external view returns (uint256);

    function allowance(address, address) external view returns (uint256);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}

contract Microlending {
    struct Loan {
        address borrower;
        uint256 amount;
        uint256 interestRate;
        uint256 duration;
        uint256 endTime;
        bool repaid;
        bool approved;
    }

    Loan[] public loans;
    address internal cUsdTokenAddress =
        0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1;
    address public admin = msg.sender;

    event LoanRequested(
        address indexed borrower,
        uint256 amount,
        uint256 interestRate,
        uint256 duration
    );
    event LoanRepaid(uint256 loanId, uint256 amount);

    /**
     *@dev Validation checks are performed on the input data to ensure correct data is sent
     * @notice Allow users to request a loan
     * @param amount The requested amount for the loan
     * @param interestRate The interest rate of the loan
     * @param duration The duration of the loan
     */
    function requestLoan(
        uint256 amount,
        uint256 interestRate,
        uint256 duration
    ) external payable {
        require(amount >= 1000, "Amount must be at least 1000 wei");
        require(interestRate > 0, "Interest rate must be greater than zero");
        require(duration > 0, "Duration must be greater than zero");

        loans.push(
            Loan(msg.sender, amount, interestRate, duration, 0, false, false)
        );

        emit LoanRequested(msg.sender, amount, interestRate, duration);
    }

    /**
     * @dev Only admin can approve loans. The amount requested by the borrower is transferred.
     * @notice Allows the admin to approve a loan
     * @param loanId The index of the loan
     */
    function approveLoan(uint256 loanId) public payable {
        require(msg.sender == admin, "Unauthorized caller");
        Loan storage loan = loans[loanId];
        require(!loan.approved, "Loan is already approved");
        require(
            IERC20Token(cUsdTokenAddress).balanceOf(address(this)) >=
                loan.amount,
            "Insufficient balance to grant loan"
        );
        loan.approved = true;
        loan.endTime = block.timestamp + loan.duration;
        require(
            IERC20Token(cUsdTokenAddress).transfer(loan.borrower, loan.amount),
            "Withdrawal failed"
        );
    }

    /**
     * @dev Balance of borrower needs to be above or equal to the loan amount to be able to repay
     * @notice Allow users to repay their due loans
     * @param loanId the index of the loan
     */
    function repayLoan(uint256 loanId) external payable {
        Loan storage loan = loans[loanId];

        require(
            msg.sender == loan.borrower,
            "Only borrower can repay the loan"
        );
        require(!loan.repaid, "Loan already repaid");
        require(loan.approved, "Loan isn't approved");
        loan.repaid = true;
        uint256 amount = loan.amount +
            ((loan.amount * loan.interestRate) / 100);
        // penalty fee for late payment of loan
        // for testing purposes, 1 seconds is used
        if (block.timestamp > loan.endTime + 1 seconds) {
            amount += 0.5 ether;
        }
        require(
            IERC20Token(cUsdTokenAddress).transferFrom(
                msg.sender,
                address(this),
                amount
            ),
            "Failed to repay loan"
        );
        emit LoanRepaid(loanId, amount);
    }
}

```

Let's breakdown the contract line by line:

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.4;

interface IERC20Token {
    function transfer(address, uint256) external returns (bool);

    function approve(address, uint256) external returns (bool);

    function transferFrom(address, address, uint256) external returns (bool);

    function totalSupply() external view returns (uint256);

    function balanceOf(address) external view returns (uint256);

    function allowance(address, address) external view returns (uint256);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}

```

This is an interface that we will use to interact with the cUSD token.

```solidity
    contract Microlending {}
```

This declares the start of the contract.

```solidity
    struct Loan {
        address borrower;
        uint256 amount;
        uint256 interestRate;
        uint256 duration;
        uint256 endTime;
        bool repaid;
        bool approved;
    }
```

This defines a [struct](https://docs.soliditylang.org/en/v0.8.17/types.html#structs) named "Loan", which represents a loan that has been requested by a borrower.

```
    Loan[] public loans;
```

This declares a `public` collection of loans requested by borrowers. By making this array public, other contracts, and external applications will be able to read the data stored in it.


```solidity
    address internal cUsdTokenAddress =
        0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1;
    address public admin = msg.sender;
```

This defines two variables where the first one stores the address of the cUSD token on the Alfajores testnet and the second one stores the address of the deployer and gives the later admin privileges.


```solidity
        event LoanRequested(
        address indexed borrower,
        uint256 amount,
        uint256 interestRate,
        uint256 duration
    );
    event LoanRepaid(uint256 loanId, uint256 amount);
```

These are two events that the contract emits when a loan is requested or repaid. Events are a way for the contract to communicate with external applications and can be subscribed to by interested parties.

```solidity
    /**
     *@dev Validation checks are performed on the input data to ensure correct data is sent
     * @notice Allow users to request a loan
     * @param amount The requested amount for the loan
     * @param interestRate The interest rate of the loan
     * @param duration The duration of the loan
     */
    function requestLoan(
        uint256 amount,
        uint256 interestRate,
        uint256 duration
    ) external payable {
        require(amount >= 1000, "Amount must be at least 1000 wei");
        require(interestRate > 0, "Interest rate must be greater than zero");
        require(duration > 0, "Duration must be greater than zero");

        loans.push(
            Loan(msg.sender, amount, interestRate, duration, 0, false, false)
        );

        emit LoanRequested(msg.sender, amount, interestRate, duration);
    }
```
Borrowers can use this feature to request a loan. It takes the loan amount, interest rate, and duration as inputs and produces a new `Loan` struct with this information. The function then adds the new loan to the loans array and fires the `LoanRequested` event.

```solidity
    /**
     * @dev Only admin can approve loans. The amount requested by the borrower is transferred.
     * @notice Allows the admin to approve a loan
     * @param loanId The index of the loan
     */
    function approveLoan(uint256 loanId) public payable {
        require(msg.sender == admin, "Unauthorized caller");
        Loan storage loan = loans[loanId];
        require(!loan.approved, "Loan is already approved");
        require(
            IERC20Token(cUsdTokenAddress).balanceOf(address(this)) >=
                loan.amount,
            "Insufficient balance to grant loan"
        );
        loan.approved = true;
        loan.endTime = block.timestamp + loan.duration;
        require(
            IERC20Token(cUsdTokenAddress).transfer(loan.borrower, loan.amount),
            "Withdrawal failed"
        );
    }

```
The `approveLoan()` function allows the admin of the platform to approve pending loans. The function first checks if the sender is the admin, it then checks if the loan hasn't yet been approved and it finally ensures that the platform can afford to grant the loan. If nothing went wrong, the loan is approved, the deadline for the loan is set and the cUSD amount is sent to the borrower.


```solidity
    /**
     * @dev Balance of borrower needs to be above or equal to the loan amount to be able to repay
     * @notice Allow users to repay their due loans
     * @param loanId the index of the loan
     */
    function repayLoan(uint256 loanId) external payable {
        Loan storage loan = loans[loanId];

        require(
            msg.sender == loan.borrower,
            "Only borrower can repay the loan"
        );
        require(!loan.repaid, "Loan already repaid");
        require(loan.approved, "Loan isn't approved");
        loan.repaid = true;
        uint256 amount = loan.amount +
            ((loan.amount * loan.interestRate) / 100);
        // penalty fee for late payment of loan
        // for testing purposes, 1 seconds is used
        if (block.timestamp > loan.endTime + 1 seconds) {
            amount += 0.5 ether;
        }
        require(
            IERC20Token(cUsdTokenAddress).transferFrom(
                msg.sender,
                address(this),
                amount
            ),
            "Failed to repay loan"
        );
        emit LoanRepaid(loanId, amount);
    }
```

Borrowers can use this function to repay their loans. The `Loan` struct associated with the supplied `loanId` is first retrieved from the loans array. The function then validates that the sender is the borrower and that the loan has not been repaid. If these conditions are met, the function marks the loan as repaid, it then computes the total amount to be repaid (loan amount + interest), it also checks if the deadline to repay the loan has been exceeded and if **true** a late payment fee is added to `amount`. Finally, the payment is made to the platform and the `LoanRepaid` event is emitted.

### Step 2: Compile the Smart Contract

To compile the smart contract, run the following command in your terminal:

```bash
    npx hardhat compile
```

>**_Note_**: Don't forget to delete the Lock.sol file, as it would otherwise lead to an error when running some commands.

This command will generate the required artifacts in the `artifacts` directory and compile the smart contract.

### Step 3: Deploy the Smart Contract

We will now create our script to deploy the smart contract to the Alfajore testnet. Go to `scripts/deploy.js` and paste the following code:

```javascript
const hre = require("hardhat");

async function main() {
  const Microlending = await hre.ethers.getContractFactory("Microlending");
  const microlending = await Microlending.deploy();

  await microlending.deployed();

  console.log("Microlending deployed to:", microlending.address);
}

main();
```

This script uses the Hardhat framework to deploy the `Microlending` smart contract to the Celo testnet. To deploy the smart contract, run the following command in your terminal:

```bash
    npx hardhat run scripts/deploy.js --network alfajores
```

After the deployment is complete, you should see the address of the deployed smart contract in your terminal.

### Step 4: Interact With the Smart Contract

We'll add a new script called `interact.js` to the `scripts` directory to communicate with the smart contract. Code the script using the following:

```javascript
const hre = require("hardhat");

async function main() {
  const Microlending = await hre.ethers.getContractFactory("Microlending");
  const cUSDSmartContract = await hre.ethers.getContractAt("IERC20Token","0x874069Fa1Eb16D44d622F2e0Ca25eeA172369bC1");
  const microlending = Microlending.attach("CONTRACT_ADDRESS");

  // Deposit cUSD into the smart contract
  const depositAmount = await ethers.utils.parseUnits("1.2", "18");
  const transferTx = await cUSDSmartContract.transfer("CONTRACT_ADDRESS", depositAmount);
  await transferTx.wait();

  // Request a loan
  const loanAmount = await ethers.utils.parseUnits("1", "18"); // 1 cUSD
  const interestRate = 10; // 10%
  const duration = 86400; // 1 day
  const requestTx = await microlending.requestLoan(loanAmount, interestRate, duration);
  await requestTx.wait();
  const loanId = 0; // Assuming only one loan has been requested so far

  // Approve loan
  const approveLoanTx = await microlending.approveLoan(loanId);
  await approveLoanTx.wait();

  // Repay the loan
  const loan = await microlending.loans(loanId);
  
  const repayAmount = await loan.amount.add(
    loan.amount.mul(loan.interestRate).div(100)
  );
  const approveTx = await cUSDSmartContract.approve("CONTRACT_ADDRESS", repayAmount);
  await approveTx.wait();
  
  const repayTx = await microlending.repayLoan(loanId);
  await repayTx.wait();

}

main();


```

Replace `CONTRACT_ADDRESS` with the address of the deployed smart contract.

This script invests $1.2 into the smart contract, requests a loan of $1 with a 10% interest rate for one day, and repays the loan at $1.1.

To run the script, run the following command in your terminal:

```bash
npx hardhat run scripts/interact.js --network alfajores
```

After the script runs successfully, you should see the transactions in your Celo wallet.

### Step 5: Testing

#### Setting up Our Testing Environment

Before we can start writing our unit tests, we will need to do a few things. Firstly, create a new file in the `contracts` folder called `ERC20Token.sol` and paste the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract cUSDTest is ERC20, Ownable {
    constructor() ERC20("cUSDTest", "cUSD") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

This smart contract will essentially allow us to simulate an ERC-20 token that shares similar functionalities with the cUSD token.

Next, we will need to create a new function called `setCUsdTokenAddress()` that essentially allows us to set the `cUsdTokenAddress` variable to store the address of the deployed cUSDTest smart contract. Copy and paste the following code in the `Microlending.sol` file:

```solidity
    function setCUsdTokenAddress(address _token) public {
        require(msg.sender == admin);
        cUsdTokenAddress = _token;
    }

```


Lastly, we will need to install the `@openzeppelin/contracts` library as we will need it to compile our `ERC20Token.sol` smart contract. To install the `@openzeppelin/contracts` library use the following command:

```bash
npm i --save-dev @openzeppelin/contracts
```

>**_Note_**: This function is only required to properly test our Microlending smart contract using the test scrips.

#### Writing the Unit Tests

To test the smart contract, we will create a new `test` file in the test directory called `microlending.js`. Add the following code to the file:

```javascript
const { expect } = require("chai");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");
require("@nomiclabs/hardhat-ethers");

describe("Microlending", function () {

	const depositAmount = ethers.utils.parseUnits("100", "18");
	const loanAmount = ethers.utils.parseUnits("10", "18");
	const interestRate = 10; // 10%
	const duration = 86400; // 1 day
	const loanId = 0;
	const zero = ethers.utils.parseUnits("0");

  // Resets the state of the blockchain before every test. Similar to how the hook beforeEach works
  // Essentially redeploys the MicroLending and the cUSDTest smart contracts and mints cUSD to the accounts 
	async function resetFixture() {
		const Microlending = await ethers.getContractFactory("Microlending");
		const CUsdToken = await ethers.getContractFactory("cUSDTest");
		const [owner, addr1, addr2] = await ethers.getSigners();
		const microlending = await Microlending.deploy();
		const cUsdToken = await CUsdToken.deploy();
		await microlending.deployed();
		await cUsdToken.deployed();
		await microlending
			.connect(owner)
			.setCUsdTokenAddress(cUsdToken.address);
		await cUsdToken.connect(owner).mint(owner.address, depositAmount);
		await cUsdToken.connect(addr1).mint(addr1.address, depositAmount);
		await cUsdToken.connect(addr2).mint(addr2.address, depositAmount);
		return { owner, addr1, addr2, microlending, cUsdToken };
	}

  // Helper function that deposits cUSD tokens to the platform
	async function deposit(cUsdToken, microlending) {
		const transferTx = await cUsdToken.transfer(
			microlending.address,
			depositAmount
		);
		await transferTx.wait();
	}
  // Helper function that initiates the requestLoan() function to create a loan
	async function createLoan(microlending, addr) {
		const createTx = await microlending
			.connect(addr)
			.requestLoan(loanAmount, interestRate, duration);
		await createTx.wait();
		return createTx;
	}
  // Helper function that initiates the repayLoan() function to repay a loan
	async function repayLoanHelper(microlending, addr) {
		const repayLoanTx = await microlending.connect(addr).repayLoan(loanId);
		await repayLoanTx.wait();
    return repayLoanTx;
	}

// Helper function that initiates the approve() function to approve the platform to spend cUSD tokens
	async function approve(cUsdToken, addr, amount, microlending) {
		const approveTx = await cUsdToken
			.connect(addr)
			.approve(microlending.address, amount);
		await approveTx.wait();
	}
  // Helper function that initiates the loans() function to fetch a loan
	async function getLoan(microlending, loanId) {
		let loan = await microlending.loans(loanId);
		return loan;
	}
  // Helper function that fetches the cUSD balance of an address
	async function getAddressBalance(cUsdToken, addr) {
		let balance = await cUsdToken.balanceOf(addr);
		return balance;
	}

  
	describe("deposit", function () {
    // Test that cUSD tokens are successfully deposited into the platform
		it("should increase the platform's cUSD balance", async function () {
			const { microlending, cUsdToken } = await loadFixture(resetFixture);
			// Smart contract's balance should initially be zero
      expect(
				await getAddressBalance(cUsdToken, microlending.address)
			).to.equal(zero);
			await deposit(cUsdToken, microlending);
      // Smart contract's balance should now be 100 cUSD after the deposit() function has been called
			expect(
				await getAddressBalance(cUsdToken, microlending.address)
			).to.equal(depositAmount);
		});
	});

	describe("requestLoan", function () {
		it("should create a new loan", async function () {
			const { addr1, microlending } = await loadFixture(resetFixture);
			await createLoan(microlending, addr1);
			const loan = await getLoan(microlending, loanId);
			expect(loan.amount).to.equal(loanAmount);
			expect(loan.interestRate).to.equal(interestRate);
			expect(loan.duration).to.equal(duration);
			expect(loan.borrower).to.equal(addr1.address);
		});

		it("should transfer the loan amount to the borrower's cUSD balance", async function () {
			const { owner, addr1, microlending, cUsdToken } = await loadFixture(
				resetFixture
			);
			await deposit(cUsdToken, microlending);
			await createLoan(microlending, addr1);
      // cUSD balance of user "addr1" should initially be 100 cUSD
			expect(await getAddressBalance(cUsdToken, addr1.address)).to.equal(
				depositAmount
			);
      // cUSD balance of smart contract should be 100 cUSD
			expect(
				await getAddressBalance(cUsdToken, microlending.address)
			).to.equal(depositAmount);
			const approveTx = await microlending
				.connect(owner)
				.approveLoan(loanId);
			await approveTx.wait();

      // balanceAddr1 = 100 cUSD(initial balance) + 10 cUSD(loan amount) = 110 cUSD
			expect(await getAddressBalance(cUsdToken, addr1.address)).to.equal(
				depositAmount.add(loanAmount)
			);
      // balanceSmartContract = 100 cUSD(initial balance) - 10 cUSD(loan amount) = 90 cUSD
			expect(
				await getAddressBalance(cUsdToken, microlending.address)
			).to.equal(depositAmount.sub(loanAmount));
		});

		it("should emit a LoanRequested event", async function () {
			const { addr1, microlending } = await loadFixture(resetFixture);
			expect(await createLoan(microlending, addr1))
				.to.emit(microlending, "LoanRequested")
				.withArgs(addr1.address, loanAmount, interestRate, duration);
		});
	});

	describe("repayLoan", function () {
    // Helper functions that essentially creates/requests, approve and repay a loan
    async function repayLoan(cUsdToken, microlending, addr1, owner){
      await deposit(cUsdToken, microlending);
			await createLoan(microlending, addr1);
			const approveTx = await microlending
				.connect(owner)
				.approveLoan(loanId);

			await approveTx.wait();
			// Repay the loan
			let loan = await getLoan(microlending, loanId);

			const repayAmount = await loan.amount.add(
				loan.amount.mul(loan.interestRate).div(100)
			);

			await approve(cUsdToken, addr1, repayAmount, microlending);
			const repayTx = await repayLoanHelper(microlending, addr1);
      return repayTx;
    }
		it("should mark the loan as repaid and transfer the repaid amount to the platform's cUSD balance", async function () {
			const { owner, addr1, microlending, cUsdToken } = await loadFixture(
				resetFixture
			);
      // Loan is created, approved and repaid
      await repayLoan(cUsdToken, microlending, addr1, owner);
			loan = await microlending.loans(loanId);
      // calculated by the formula x = y(loan amount) * z(loan interest) / 100%
      // in current case: x = 10 cUSD(loan amount) * 10%(loan interest) / 100% = 1 cUSD 
      const repayAmount = await loan.amount.add(
				loan.amount.mul(loan.interestRate).div(100)
			);
			expect(loan.repaid).to.be.true;
			expect(
				await getAddressBalance(cUsdToken, microlending.address)
			).to.equal(depositAmount.sub(loanAmount).add(repayAmount));
		});
		it("should emit a LoanRepaid event", async function () {
			const { owner, addr1, microlending, cUsdToken } = await loadFixture(
				resetFixture
			);
			expect(await repayLoan(cUsdToken, microlending, addr1, owner))
				.to.emit(microlending, "LoanRepaid")
				.withArgs(0, owner.address, loanAmount);
		});
		it("should prevent non-borrowers from repaying loans", async function () {
			const { owner, addr1, microlending, cUsdToken } = await loadFixture(
				resetFixture
			);
			expect(
				await repayLoan(cUsdToken, microlending, addr1, owner)
			).to.be.revertedWith("Only borrower can repay loan");
		});

		it("should prevent repaying already repaid loans", async function () {
      
			const { owner, addr1, microlending, cUsdToken } = await loadFixture(
				resetFixture
			);
      
			await repayLoan(cUsdToken, microlending, addr1, owner);
			await expect(
				repayLoanHelper(microlending, addr1)
			).to.be.revertedWith("Loan already repaid");
		});
	});
});

```

This test file sets up the `Microlending` contract, creates three signers, and tests the deposit, requestLoan, and repayLoan functions.

To run the tests, run the following command in your terminal:

```bash
    npx hardhat test
```

You should see output similar to the following:

```bash
  Microlending
    deposit
      √ should increase the platform's cUSD balance (474ms)
    requestLoan
      √ should create a new loan
      √ should transfer the loan amount to the borrower's cUSD balance (93ms)
      √ should emit a LoanRequested event
    repayLoan
      √ should mark the loan as repaid and transfer the repaid amount to the platform's cUSD balance (113ms)
      √ should emit a LoanRepaid event (87ms)
      √ should prevent non-borrowers from repaying loans (88ms)
      √ should prevent repaying already repaid loans (143ms)


  8 passing (1s)
```

All tests should pass without any errors.

Congratulations! You have now created a rudimentary microlending platform on the Celo blockchain. You can use this as a starting point for developing more complex financial applications using Celo.

The next steps could include developing more sophisticated risk assessment and credit scoring algorithms, integrating with off-chain data sources, and developing a user interface to interact with the smart contract.

## Conclusion

In conclusion, developing a Celo-based microlending platform can be an excellent strategy to promote financial inclusion and provide credit to those in greatest need. We can establish a transparent, safe, and efficient lending platform that benefits both lenders and borrowers by leveraging the power of smart contracts and the Celo blockchain.

We covered the fundamentals of developing a microlending platform on Celo in this tutorial, including setting up a development environment, generating a smart contract, and testing our code. We also looked at the many functionalities of a microlending platform, such as seeking loans, repaying loans, depositing cash, and withdrawing funds.

You should now have a strong foundation for creating your Celo based microlending platform after following the instructions provided in this article. Of course, there's always more to learn and do better, but this tutorial ought to provide you with a good place to start as you continue on your development path.

When developing on any blockchain, keep in mind to always put security and best practices first, and don't be afraid to ask the Celo community for help and advice along the way. Wishing you well as you create your microlending platform!
