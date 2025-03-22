// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract TokenizedBallotSystem is ERC20, ERC20Permit, ERC20Votes, AccessControl {
    // Token-related state variables
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    
    // Ballot-related state variables
    struct Proposal {
        bytes32 name;
        uint voteCount;
    }
    
    Proposal[] public proposals;
    uint256 public targetBlockNumber;
    mapping(address => uint256) public votePowerSpent;
    bool public ballotInitialized = false;
    
    // Events
    event BallotInitialized(uint256 targetBlockNumber);
    event ProposalAdded(bytes32 name, uint256 index);
    event VoteCast(address voter, uint256 proposalId, uint256 amount);
    
    // Token constructor
    constructor() ERC20("Governance Token", "GOV") ERC20Permit("Governance Token") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
    }
    
    // Token functions
    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
    
    // Required overrides from parent contracts
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Votes)
    {
        super._update(from, to, value);
    }

    function nonces(address owner)
        public
        view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
    
    // Ballot initialization function
    function initializeBallot(bytes32[] memory _proposalNames, uint256 _targetBlockNumber) external onlyRole(DEFAULT_ADMIN_ROLE) {
        require(!ballotInitialized, "Ballot already initialized");
        require(block.number > _targetBlockNumber, "Target block must be in the past");
        
        targetBlockNumber = _targetBlockNumber;
        
        for (uint i = 0; i < _proposalNames.length; i++) {
            proposals.push(Proposal({
                name: _proposalNames[i],
                voteCount: 0
            }));
            emit ProposalAdded(_proposalNames[i], i);
        }
        
        ballotInitialized = true;
        emit BallotInitialized(targetBlockNumber);
    }
    
    // Ballot voting function
    function vote(uint256 proposal, uint256 amount) external {
        require(ballotInitialized, "Ballot not initialized");
        require(proposal < proposals.length, "Invalid proposal");
        
        uint256 votingPower = getPastVotes(msg.sender, targetBlockNumber);
        
        uint256 spent = votePowerSpent[msg.sender];
        require(votingPower >= spent + amount, "Not enough voting power");
        
        votePowerSpent[msg.sender] = spent + amount;
        
        proposals[proposal].voteCount += amount;
        emit VoteCast(msg.sender, proposal, amount);
    }
    
    // Ballot query functions
    function winningProposal() public view returns (uint winningProposal_) {
        require(ballotInitialized, "Ballot not initialized");
        uint winningVoteCount = 0;
        for (uint p = 0; p < proposals.length; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal_ = p;
            }
        }
    }

    function winnerName() external view returns (bytes32 winnerName_) {
        require(ballotInitialized, "Ballot not initialized");
        winnerName_ = proposals[winningProposal()].name;
    }
    
    function getProposal(uint256 index) external view returns (bytes32 name, uint256 voteCount) {
        require(ballotInitialized, "Ballot not initialized");
        require(index < proposals.length, "Invalid proposal index");
        return (proposals[index].name, proposals[index].voteCount);
    }
    
    function getProposalsCount() external view returns (uint256) {
        require(ballotInitialized, "Ballot not initialized");
        return proposals.length;
    }
    
    function getRemainingVotingPower(address voter) external view returns (uint256) {
        require(ballotInitialized, "Ballot not initialized");
        uint256 votingPower = getPastVotes(voter, targetBlockNumber);
        return votingPower - votePowerSpent[voter];
    }
}


//Testing Framework:

describe("TokenizedBallotSystem", function () {
    describe("Token Functionality", function () {
        it("Should mint tokens to accounts and reflect the correct balance")
        it("Should allow delegation of voting power through self-delegation") 
        
        //TODO
        
    });
    
    describe("Ballot Initialization", function () {
        it("Should only allow admin to initialize the ballot with proposals")
        it("Should require the target block number to be in the past")
        
        //TODO
        
    });
    
    describe("Voting Process", function () {
        it("Should allow voting with token voting power based on holdings at target block")
        it("Should not allow voting beyond available voting power") 
        
        //TODO
        
    });
    
    describe("Results Calculation", function () {
        it("Should correctly determine the winning proposal based on vote count")
        it("Should return the correct name of the winning proposal") 
        
        //TODO
        
    });
    
    describe("Voting Power Management", function () {
        it("Should track remaining voting power correctly after partial votes")
        it("Should prevent double-spending of voting power across multiple votes")
        
        //TODO
        
    });
}

