const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("TokenizedBallotSystem", function () {
  let tokenizedBallot;
  let owner;
  let addr1;
  let addr2;
  let addrs;
  let blockNumber;

  // Convert string to bytes32
  function stringToBytes32(str) {
    return ethers.utils.formatBytes32String(str);
  }

  beforeEach(async function () {
    // Deploy the contract before each test
    const TokenizedBallotSystem = await ethers.getContractFactory("TokenizedBallotSystem");
    [owner, addr1, addr2, ...addrs] = await ethers.getSigners();
    tokenizedBallot = await TokenizedBallotSystem.deploy();
    await tokenizedBallot.deployed();
    
    // Current block will be our target block for voting
    blockNumber = await ethers.provider.getBlockNumber();
  });

  describe("Token Functionality", function () {
    it("Should mint tokens to accounts and reflect the correct balance", async function () {
      // Mint 100 tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Check the balance of addr1
      const balance = await tokenizedBallot.balanceOf(addr1.address);
      expect(balance).to.equal(ethers.utils.parseEther("100"));
    });

    it("Should allow delegation of voting power through self-delegation", async function () {
      // Mint 100 tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Move one block forward
      await ethers.provider.send("evm_mine", []);
      
      // Get current votes
      const votes = await tokenizedBallot.getVotes(addr1.address);
      expect(votes).to.equal(ethers.utils.parseEther("100"));
    });
  });

  describe("Ballot Initialization", function () {
    it("Should only allow admin to initialize the ballot with proposals", async function () {
      // Prepare proposal names
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2"),
        stringToBytes32("Proposal 3")
      ];
      
      // Move forward a block to make sure targetBlockNumber is in the past
      await ethers.provider.send("evm_mine", []);
      
      // Admin initializes the ballot
      await tokenizedBallot.initializeBallot(proposalNames, blockNumber);
      
      // Check if ballot is initialized
      const initialized = await tokenizedBallot.ballotInitialized();
      expect(initialized).to.equal(true);
      
      // Try to initialize again, should revert
      await expect(
        tokenizedBallot.initializeBallot(proposalNames, blockNumber)
      ).to.be.revertedWith("Ballot already initialized");
      
      // Non-admin tries to initialize, should revert
      await expect(
        tokenizedBallot.connect(addr1).initializeBallot(proposalNames, blockNumber)
      ).to.be.reverted;
    });

    it("Should require the target block number to be in the past", async function () {
      // Prepare proposal names
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2")
      ];
      
      // Try to initialize with future block
      const futureBlock = await ethers.provider.getBlockNumber() + 10;
      
      // Should revert when target block is in the future
      await expect(
        tokenizedBallot.initializeBallot(proposalNames, futureBlock)
      ).to.be.revertedWith("Target block must be in the past");
    });
  });

  describe("Voting Process", function () {
    it("Should allow voting with token voting power based on holdings at target block", async function () {
      // Mint tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // Vote for proposal 0 with 50 tokens
      await tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("50"));
      
      // Check vote count
      const proposal = await tokenizedBallot.getProposal(0);
      expect(proposal.voteCount).to.equal(ethers.utils.parseEther("50"));
    });

    it("Should not allow voting beyond available voting power", async function () {
      // Mint tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // Try to vote with more tokens than available
      await expect(
        tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("150"))
      ).to.be.revertedWith("Not enough voting power");
    });
  });

  describe("Results Calculation", function () {
    it("Should correctly determine the winning proposal based on vote count", async function () {
      // Mint tokens to voters
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      await tokenizedBallot.mint(addr2.address, ethers.utils.parseEther("50"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      await tokenizedBallot.connect(addr2).delegate(addr2.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2"),
        stringToBytes32("Proposal 3")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // addr1 votes for proposal 0 with 60 tokens
      await tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("60"));
      
      // addr2 votes for proposal 1 with 50 tokens
      await tokenizedBallot.connect(addr2).vote(1, ethers.utils.parseEther("50"));
      
      // addr1 votes for proposal 2 with 40 tokens
      await tokenizedBallot.connect(addr1).vote(2, ethers.utils.parseEther("40"));
      
      // Check winning proposal
      const winningProposal = await tokenizedBallot.winningProposal();
      expect(winningProposal).to.equal(0); // Proposal 0 has 60 votes, the most
    });

    it("Should return the correct name of the winning proposal", async function () {
      // Mint tokens to voter
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalName = "Winner Proposal";
      const proposalNames = [
        stringToBytes32(proposalName),
        stringToBytes32("Loser Proposal")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // Vote for proposal 0
      await tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("100"));
      
      // Check winner name
      const winnerName = await tokenizedBallot.winnerName();
      expect(ethers.utils.parseBytes32String(winnerName)).to.equal(proposalName);
    });
  });

  describe("Voting Power Management", function () {
    it("Should track remaining voting power correctly after partial votes", async function () {
      // Mint tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // Vote for proposal 0 with 30 tokens
      await tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("30"));
      
      // Check remaining voting power
      let remainingPower = await tokenizedBallot.getRemainingVotingPower(addr1.address);
      expect(remainingPower).to.equal(ethers.utils.parseEther("70"));
      
      // Vote for proposal 1 with 20 tokens
      await tokenizedBallot.connect(addr1).vote(1, ethers.utils.parseEther("20"));
      
      // Check remaining voting power again
      remainingPower = await tokenizedBallot.getRemainingVotingPower(addr1.address);
      expect(remainingPower).to.equal(ethers.utils.parseEther("50"));
    });

    it("Should prevent double-spending of voting power across multiple votes", async function () {
      // Mint tokens to addr1
      await tokenizedBallot.mint(addr1.address, ethers.utils.parseEther("100"));
      
      // Self-delegate to activate voting power
      await tokenizedBallot.connect(addr1).delegate(addr1.address);
      
      // Wait for a block
      await ethers.provider.send("evm_mine", []);
      
      // Save the current block as our snapshot
      const targetBlock = await ethers.provider.getBlockNumber();
      
      // Move forward a block
      await ethers.provider.send("evm_mine", []);
      
      // Initialize ballot with past block number
      const proposalNames = [
        stringToBytes32("Proposal 1"),
        stringToBytes32("Proposal 2")
      ];
      await tokenizedBallot.initializeBallot(proposalNames, targetBlock);
      
      // Vote for proposal 0 with 80 tokens
      await tokenizedBallot.connect(addr1).vote(0, ethers.utils.parseEther("80"));
      
      // Try to vote for proposal 1 with 30 tokens (only 20 remain)
      await expect(
        tokenizedBallot.connect(addr1).vote(1, ethers.utils.parseEther("30"))
      ).to.be.revertedWith("Not enough voting power");
      
      // Vote with remaining 20 tokens should work
      await tokenizedBallot.connect(addr1).vote(1, ethers.utils.parseEther("20"));
      
      // Check remaining voting power is now 0
      const remainingPower = await tokenizedBallot.getRemainingVotingPower(addr1.address);
      expect(remainingPower).to.equal(ethers.utils.parseEther("0"));
    });
  });
});