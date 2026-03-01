---
name: blockchain-escrow
description: Implement trustless escrow on EVM chains using OpenZeppelin patterns. Use when building products where multiple parties need to lock funds with automatic conditional release — commitments, marketplaces, betting, conditional payments.
---

# Blockchain Escrow

Extracted from: `OpenZeppelin/openzeppelin-contracts` — Escrow, ConditionalEscrow, RefundEscrow patterns.

## Core Mental Model

Blockchain escrow = funds locked in a smart contract that releases only when defined conditions are met, with no human able to override the outcome. This is the trustless primitive that makes multi-party financial agreements possible without a bank, lawyer, or platform as intermediary.

OpenZeppelin provides three escrow contracts as building blocks:
1. **Escrow** — basic deposit/withdraw per address
2. **ConditionalEscrow** — adds a condition that must be true to withdraw
3. **RefundEscrow** — three states: active (accepting deposits) → closed (winner can withdraw) → refunding (all deposits returned)

**RefundEscrow is the right base for most commitment/betting products.**

## OpenZeppelin RefundEscrow Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/escrow/RefundEscrow.sol";

// RefundEscrow state machine:
// Active    → deposits accepted, no withdrawals
// Closed    → beneficiary (winner) can withdraw
// Refunding → all depositors can get their money back

contract CommitmentEscrow {
    RefundEscrow private _escrow;
    
    constructor(address payable beneficiary) {
        _escrow = new RefundEscrow(beneficiary);
    }
    
    // Accept deposits from any participant
    function deposit(address refundee) external payable {
        _escrow.deposit{value: msg.value}(refundee);
    }
    
    // Creator wins — close escrow, beneficiary can withdraw
    function creatorWins() external {
        _escrow.close();       // moves to Closed state
        _escrow.beneficiaryWithdraw();  // sends to winner
    }
    
    // Creator loses or tie — everyone gets refunded
    function refundAll() external {
        _escrow.enableRefunds();  // moves to Refunding state
        // Each depositor calls withdrawRefund(depositorAddress)
    }
    
    function withdrawRefund(address payable refundee) external {
        _escrow.withdraw(refundee);
    }
}
```

## Full Pattern for Friend Circle Commitment Protocol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract FriendCircle is ReentrancyGuard {
    struct Commitment {
        address payable creator;
        uint256 creatorStake;
        uint256 deadline;
        uint256 votingDeadline;
        bool settled;
        
        // Bets placed against creator
        address[] bettors;
        mapping(address => uint256) betAmounts;
        uint256 totalBet;
        
        // Voting
        mapping(address => bool) hasVoted;
        mapping(address => bool) voteSuccess;  // true = creator succeeded
        uint256 votesFor;
        uint256 votesAgainst;
        
        // Pending withdrawals (pull pattern)
        mapping(address => uint256) pendingPayout;
    }

    mapping(uint256 => Commitment) public commitments;
    uint256 public nextId;
    uint256 public constant FEE_BPS = 400;  // 4%

    // Events
    event Created(uint256 indexed id, address creator, uint256 stake, uint256 deadline);
    event BetPlaced(uint256 indexed id, address bettor, uint256 amount);
    event Voted(uint256 indexed id, address voter, bool success);
    event Settled(uint256 indexed id, bool creatorWon, uint256 fee);
    event Withdrawn(address indexed to, uint256 amount);

    function create(uint256 deadline) external payable returns (uint256) {
        require(msg.value > 0, "Stake required");
        require(deadline > block.timestamp + 1 hours, "Deadline too soon");

        uint256 id = nextId++;
        Commitment storage c = commitments[id];
        c.creator = payable(msg.sender);
        c.creatorStake = msg.value;
        c.deadline = deadline;
        c.votingDeadline = deadline + 48 hours;

        emit Created(id, msg.sender, msg.value, deadline);
        return id;
    }

    function bet(uint256 id) external payable {
        Commitment storage c = commitments[id];
        require(!c.settled, "Settled");
        require(block.timestamp < c.deadline, "Deadline passed");
        require(msg.sender != c.creator, "Cannot bet against yourself");
        require(msg.value > 0, "Bet required");

        if (c.betAmounts[msg.sender] == 0) {
            c.bettors.push(msg.sender);
        }
        c.betAmounts[msg.sender] += msg.value;
        c.totalBet += msg.value;

        emit BetPlaced(id, msg.sender, msg.value);
    }

    function vote(uint256 id, bool success) external {
        Commitment storage c = commitments[id];
        require(!c.settled, "Settled");
        require(block.timestamp >= c.deadline, "Not yet");
        require(block.timestamp < c.votingDeadline, "Voting over");
        require(!c.hasVoted[msg.sender], "Already voted");
        // Voter must be creator or a bettor
        require(
            msg.sender == c.creator || c.betAmounts[msg.sender] > 0,
            "Not a participant"
        );

        c.hasVoted[msg.sender] = true;
        c.voteSuccess[msg.sender] = success;
        if (success) c.votesFor++; else c.votesAgainst++;

        emit Voted(id, msg.sender, success);
    }

    function settle(uint256 id) external nonReentrant {
        Commitment storage c = commitments[id];
        require(!c.settled, "Already settled");
        require(block.timestamp >= c.votingDeadline, "Voting still open");

        c.settled = true;

        uint256 fee = 0;

        if (c.votesFor == c.votesAgainst) {
            // TIE — everyone gets their money back
            c.pendingPayout[c.creator] += c.creatorStake;
            for (uint i = 0; i < c.bettors.length; i++) {
                address b = c.bettors[i];
                c.pendingPayout[b] += c.betAmounts[b];
            }
        } else if (c.votesFor > c.votesAgainst) {
            // CREATOR WINS — gets stake back + pot (minus fee)
            fee = c.totalBet * FEE_BPS / 10000;
            c.pendingPayout[c.creator] += c.creatorStake + c.totalBet - fee;
        } else {
            // CREATOR LOSES — stake distributed to winning bettors
            fee = c.creatorStake * FEE_BPS / 10000;
            uint256 distribute = c.creatorStake - fee;
            // Distribute proportional to bet size
            for (uint i = 0; i < c.bettors.length; i++) {
                address b = c.bettors[i];
                uint256 share = distribute * c.betAmounts[b] / c.totalBet;
                c.pendingPayout[b] += c.betAmounts[b] + share;
            }
        }

        emit Settled(id, c.votesFor > c.votesAgainst, fee);
    }

    // Pull pattern — each user claims their own payout
    function withdraw(uint256 id) external nonReentrant {
        Commitment storage c = commitments[id];
        uint256 amount = c.pendingPayout[msg.sender];
        require(amount > 0, "Nothing to withdraw");

        c.pendingPayout[msg.sender] = 0;
        payable(msg.sender).transfer(amount);

        emit Withdrawn(msg.sender, amount);
    }
}
```

## Security Checklist for Escrow Contracts

- [ ] **ReentrancyGuard** on all ETH-sending functions
- [ ] **Pull over push** — never loop to pay multiple addresses
- [ ] **Checks-Effects-Interactions** — update state before external calls
- [ ] **Integer arithmetic** — 0.8+ handles overflow; use basis points for percentages
- [ ] **Access control** — who can call settle()? Only participants? Anyone after deadline?
- [ ] **Edge cases** — what if 0 bettors? What if creator is sole voter?
- [ ] **Event logging** — emit events for every state change (frontend + audit trail)
- [ ] **Get a Slither static analysis run** before mainnet: `slither src/`
- [ ] **Formal audit** before real money — minimum $5k, worth it if TVL > $50k

## Testing Template

```solidity
function test_FullCommitmentFlow() public {
    // Alice creates, Bob bets, both vote, settle, withdraw
    vm.prank(alice);
    uint256 id = contract.create{value: 0.02 ether}(block.timestamp + 7 days);

    vm.prank(bob);
    contract.bet{value: 0.01 ether}(id);

    vm.warp(block.timestamp + 7 days + 1);  // fast-forward to deadline

    vm.prank(alice);
    contract.vote(id, true);  // alice votes she succeeded
    vm.prank(bob);
    contract.vote(id, true);  // bob agrees

    vm.warp(block.timestamp + 48 hours + 1);  // past voting deadline
    contract.settle(id);

    uint256 balanceBefore = alice.balance;
    vm.prank(alice);
    contract.withdraw(id);
    assertGt(alice.balance, balanceBefore);
}
```

## Deployment on Polygon (Recommended for Low Fees)

```bash
# Add Polygon to foundry.toml
[rpc_endpoints]
polygon = "https://polygon-rpc.com"
mumbai = "https://rpc-mumbai.maticvigil.com"

# Deploy
forge script script/Deploy.s.sol \
  --rpc-url polygon \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify \
  --etherscan-api-key $POLYGONSCAN_KEY
```

## Key Gotchas

- `block.timestamp + 7 days` works but can be gamed by ~15 seconds by validators — fine for commitment apps, not fine for HFT
- Always use **basis points** for fees: `400 = 4%`, calculate as `amount * 400 / 10000`
- Mapping inside a struct: can't be returned or iterated directly — track keys in a separate array
- `address(this).balance` is the total ETH held by the contract — useful for sanity checks in tests
