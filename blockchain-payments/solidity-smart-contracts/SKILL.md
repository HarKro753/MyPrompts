---
name: solidity-smart-contracts
description: Write, deploy, and test Solidity smart contracts on Ethereum and EVM-compatible chains. Use when building contracts for escrow, payments, voting, or token mechanics. Covers OpenZeppelin patterns, Foundry testing, and Layer 2 deployment.
---

# Solidity Smart Contracts

Extracted from: `OpenZeppelin/openzeppelin-contracts` (27k⭐), `foundry-rs/foundry` (10k⭐), `transmissions11/solmate` (4.3k⭐), `scaffold-eth/scaffold-eth-2` (2k⭐)

## Core Mental Model

A smart contract is immutable code that holds and moves funds according to rules nobody can change. The moment it's deployed, not even the author can override it. This makes security the first concern — every function that moves money is an attack surface.

The OpenZeppelin pattern: **inherit from audited base contracts rather than writing from scratch.** Their contracts represent years of security audits and bug fixes. Starting from scratch for a token, escrow, or access control system is almost always wrong.

## Project Setup (Foundry — industry standard)

```bash
# Install Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# New project
forge init my-project
cd my-project

# Install OpenZeppelin
forge install OpenZeppelin/openzeppelin-contracts

# Project structure
src/          # Contract source files
test/         # Forge tests (Solidity)
script/       # Deployment scripts
lib/          # Dependencies (OpenZeppelin etc.)
```

## Escrow Contract Pattern (for Friend Circle Commitment Protocol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract CommitmentEscrow is ReentrancyGuard {
    enum Status { Active, Voting, Settled }

    struct Commitment {
        address creator;
        uint256 stake;
        uint256 pot;           // sum of bets against
        uint256 deadline;
        Status status;
        uint256 votesFor;
        uint256 votesAgainst;
        mapping(address => uint256) betAgainst;  // amount bet against
        mapping(address => bool) hasVoted;
    }

    mapping(uint256 => Commitment) public commitments;
    uint256 public nextId;
    uint256 public feeBps = 400; // 4% in basis points

    event CommitmentCreated(uint256 id, address creator, uint256 stake, uint256 deadline);
    event BetPlaced(uint256 id, address bettor, uint256 amount);
    event Voted(uint256 id, address voter, bool vote);
    event Settled(uint256 id, bool success, uint256 payout);

    function createCommitment(uint256 deadline) external payable {
        require(msg.value > 0, "Must stake ETH");
        require(deadline > block.timestamp, "Deadline must be future");

        uint256 id = nextId++;
        Commitment storage c = commitments[id];
        c.creator = msg.sender;
        c.stake = msg.value;
        c.deadline = deadline;
        c.status = Status.Active;

        emit CommitmentCreated(id, msg.sender, msg.value, deadline);
    }

    function betAgainst(uint256 id) external payable {
        Commitment storage c = commitments[id];
        require(c.status == Status.Active, "Not active");
        require(block.timestamp < c.deadline, "Deadline passed");
        require(msg.value > 0, "Must bet ETH");
        require(msg.sender != c.creator, "Cannot bet against yourself");

        c.betAgainst[msg.sender] += msg.value;
        c.pot += msg.value;

        emit BetPlaced(id, msg.sender, msg.value);
    }

    function vote(uint256 id, bool success) external {
        Commitment storage c = commitments[id];
        require(block.timestamp >= c.deadline, "Voting not open yet");
        require(c.status == Status.Active, "Not in voting phase");
        require(!c.hasVoted[msg.sender], "Already voted");
        // Only group members who staked can vote — add membership check here

        c.hasVoted[msg.sender] = true;
        if (success) c.votesFor++; else c.votesAgainst++;

        emit Voted(id, msg.sender, success);
    }

    function settle(uint256 id) external nonReentrant {
        Commitment storage c = commitments[id];
        require(c.status == Status.Active, "Already settled");
        require(block.timestamp >= c.deadline + 48 hours, "Voting period not over");

        c.status = Status.Settled;

        if (c.votesFor == c.votesAgainst) {
            // Tie — refund creator
            payable(c.creator).transfer(c.stake);
            // Refund all bettors (implement loop or pull pattern)
            emit Settled(id, false, c.stake);
        } else if (c.votesFor > c.votesAgainst) {
            // Creator wins
            uint256 fee = c.pot * feeBps / 10000;
            uint256 payout = c.stake + c.pot - fee;
            payable(c.creator).transfer(payout);
            emit Settled(id, true, payout);
        } else {
            // Creator loses — distribute pot to winning bettors
            // Use pull pattern for payouts (see below)
            emit Settled(id, false, 0);
        }
    }
}
```

## Critical Security Patterns

### ReentrancyGuard — always use on functions that send ETH
```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
// Inherit and use nonReentrant modifier on any function sending ETH
```

### Pull over Push for payouts
Never loop over addresses to pay out — gas limits can cause transactions to fail and funds get stuck. Instead, let each recipient withdraw:
```solidity
mapping(address => uint256) public pendingWithdrawals;

function claimPayout(uint256 id) external nonReentrant {
    uint256 amount = pendingWithdrawals[msg.sender];
    require(amount > 0, "Nothing to claim");
    pendingWithdrawals[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

### Checks-Effects-Interactions pattern
Always: check conditions → update state → interact with external contracts/send ETH. Never send ETH before updating state.

## Testing with Foundry

```solidity
// test/CommitmentEscrow.t.sol
pragma solidity ^0.8.20;
import "forge-std/Test.sol";
import "../src/CommitmentEscrow.sol";

contract CommitmentEscrowTest is Test {
    CommitmentEscrow public escrow;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        escrow = new CommitmentEscrow();
        vm.deal(alice, 1 ether);
        vm.deal(bob, 1 ether);
    }

    function test_CreateCommitment() public {
        vm.prank(alice);
        uint256 deadline = block.timestamp + 7 days;
        escrow.createCommitment{value: 0.02 ether}(deadline);
        // assert state
    }

    function test_TieRefundsStake() public {
        // test 50/50 tie returns stake
    }
}
```

```bash
forge test                    # run all tests
forge test -vvv               # verbose (shows logs)
forge test --gas-report       # gas usage per function
forge coverage                # coverage report
```

## Deployment

```bash
# Deploy to local anvil testnet
anvil  # starts local chain

# Deploy script
forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast

# Deploy to Polygon Mumbai testnet
forge script script/Deploy.s.sol \
  --rpc-url https://rpc-mumbai.maticvigil.com \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify
```

## Choosing a Chain for Low-Cost Apps

For Friend Circle Commitment Protocol (small stakes, high tx volume):
- **Polygon PoS** — cheapest, fastest, most mature L2. Gas ~$0.001. Best for MVP.
- **Arbitrum One** — Ethereum L2, slightly higher fees than Polygon but stronger security guarantees.
- **Base** — Coinbase's L2, growing fast, good UX tools.
- Avoid Ethereum mainnet for small stakes — gas fees would eat the entire stake.

## Frontend Integration (wagmi + viem)

```typescript
import { useWriteContract, useReadContract } from 'wagmi'
import { parseEther } from 'viem'

// Write to contract
const { writeContract } = useWriteContract()
writeContract({
  address: CONTRACT_ADDRESS,
  abi: ESCROW_ABI,
  functionName: 'createCommitment',
  args: [deadline],
  value: parseEther('0.02'),  // stake
})

// Read from contract
const { data } = useReadContract({
  address: CONTRACT_ADDRESS,
  abi: ESCROW_ABI,
  functionName: 'commitments',
  args: [commitmentId],
})
```

## OpenZeppelin Contracts to Know

- `ReentrancyGuard` — prevent reentrancy attacks on ETH transfers
- `Ownable` — access control, owner-only functions
- `Pausable` — emergency stop mechanism
- `Escrow` — base escrow with deposit/withdraw pattern
- `ConditionalEscrow` — escrow that releases on a condition
- `AccessControl` — role-based access (more flexible than Ownable)

## Gotchas

- `block.timestamp` can be manipulated by miners by ~15 seconds — don't use for precise timing
- Integer overflow is handled by Solidity 0.8+ automatically (reverts on overflow)
- `address.transfer()` sends 2300 gas (may fail); prefer `call{value: amount}("")`
- Storage is expensive — use `memory` for temporary data, `storage` only for persistent state
- Events are the cheap way to store data for frontend consumption — logs cost ~8 gas/byte vs ~20,000 gas for storage
