---
layout: post
title: "Ityfuzz Design Summary"
date: 2025-12-11
---

## Overview

ItyFuzz is a feedback-driven fuzzer for smart contract VMs (primarily EVM) built on top of LibAFL. Instead of fuzzing traditional programs, ItyFuzz fuzzes **EVM transactions** to discover vulnerabilities in smart contracts. The key innovation is using **VM-level coverage** (jump/branch coverage within the VM execution) rather than program instruction coverage, combined with specialized feedback mechanisms for smart contract vulnerability detection.

## Introduction to LibAFL

### Core Concepts

LibAFL is a framework for building feedback-driven fuzzers. It provides a modular architecture with the following core components:

- **Observer**: An entity that provides information observed during execution (e.g., coverage maps). The information is volatile and not preserved across executions, but can be serialized when an input is considered interesting.

- **Executor**: Defines how to execute the target program and all volatile operations related to a single run. It can hold a set of Observers and is responsible for informing the program about the input to use.

- **Feedback**: Classifies the outcome of an execution as "interesting" or not. Typically processes information from Observers to decide if an input should be added to the corpus. The concept of "interestingness" is abstract but often relates to novelty search (e.g., reaching previously unseen edges).

- **Input**: The internal representation of program input. While often a byte array, it can be more complex (e.g., Abstract Syntax Trees for grammar fuzzing). Inputs must be serializable and contain only owned data.

- **Corpus**: Where testcases (Inputs with metadata) are stored. Testcases are added when considered interesting or when they fulfill an objective (like crashing the program).

- **Scheduler**: The policy for retrieving the next testcase from the Corpus (e.g., FIFO, weighted random selection).

- **Mutator**: Takes one or more Inputs and generates a new Input derived from them. Mutators can be composed and are generally linked to specific Input types.

- **Generator**: Generates an Input from scratch (e.g., random generator, grammar generator).

- **Stage**: Operates on a single Input received from the Corpus. For instance, a Mutational Stage applies a Mutator and executes the generated input one or more times.

### Architecture

The LibAFL architecture is built around entities to allow code reuse and low-cost abstractions. LibAFL uses a component-based approach rather than traditional object-oriented inheritance, following Rust's idiomatic patterns. The library introduces two key entities:

- **Testcase**: A container for an Input stored in the Corpus along with its metadata. In the implementation, the Corpus stores Testcases rather than raw Inputs.

- **State**: Contains all metadata that evolves while running the fuzzer, including the Corpus itself. The State contains only owned objects that are serializable, and it is serializable itself. This allows fuzzers to serialize their state when pausing or, when doing in-process fuzzing, serialize on crash and deserialize in a new process to continue fuzzing with all metadata preserved.

Additionally, entities that are "actions", like the `CorpusScheduler` and `Feedbacks`, are grouped in a common place, the `Fuzzer`. The data structures that are modified most frequently are those related to testcases and the fuzzer global state.

ItyFuzz implements each of these LibAFL concepts, adapting them for smart contract fuzzing where the "program" is a VM executing transactions rather than a traditional binary.

## Core Architecture

### 1. Generic VM Abstraction (`generic_vm/`)

ItyFuzz abstracts smart contract execution through a generic VM interface:

- **`GenericVM` trait**: Defines the interface for executing transactions on any VM (EVM, Move, etc.)
  - `execute()`: Runs a transaction and returns execution results
  - `get_jmp()`, `get_read()`, `get_write()`, `get_cmp()`: Access coverage maps (4096-size arrays)
  - These maps track VM-level coverage: jumps, storage reads/writes, and comparisons

- **`VMStateT` trait**: Represents the state of a VM
  - Can be hashed, compared, and serialized
  - Tracks post-execution state for incomplete transactions

**Design Decision**: This abstraction allows ItyFuzz to support multiple VMs (EVM, Move) while sharing the same fuzzing logic.

### 2. LibAFL Component Mapping

#### **Executor** (`src/executor.rs`)
- **LibAFL Concept**: `Executor` defines how to execute the target program
- **ItyFuzz Implementation**: `FuzzExecutor` wraps a `GenericVM` and implements LibAFL's `Executor` trait
  - `run_target()`: Executes a VM input (transaction) using the wrapped VM
  - Stores execution results in the fuzzer state
  - Holds observers (coverage maps) that get updated during execution

#### **Input** (`src/input.rs`)
- **LibAFL Concept**: `Input` represents program input data
- **ItyFuzz Implementation**: `VMInputT` trait extends LibAFL's `Input` for VM transactions
  - Contains: caller address, contract address, calldata, VM state before execution
  - Can be mutated (via `mutate()` method)
  - Can generate concise serializable representation (`ConciseSerde`)
  - Tracks which VM state (from infant corpus) it was derived from

#### **State** (`src/state.rs`)
- **LibAFL Concept**: `State` holds all fuzzing state (corpus, metadata, RNG, etc.)
- **ItyFuzz Implementation**: `FuzzState` implements LibAFL's `State` trait with:
  - **Main Corpus**: Stores interesting transaction inputs (`txn_corpus`)
  - **Infant State Corpus**: Stores interesting VM states (`infant_states_state`)
  - **Solutions Corpus**: Stores inputs that triggered vulnerabilities
  - **Execution Result**: Current execution output, reverted status, new VM state
  - **Metadata**: Various metadata maps for feedback mechanisms

**Key Innovation**: ItyFuzz maintains **two corpora**:
1. **Transaction Corpus**: Inputs (transactions) that achieved interesting coverage
2. **Infant State Corpus**: VM states that are interesting for further exploration

#### **Observer** (via `GenericVM` coverage maps)
- **LibAFL Concept**: `Observer` collects information during execution
- **ItyFuzz Implementation**: Uses static coverage maps accessed via `GenericVM`:
  - `JMP_MAP`: Jump/branch coverage (4096 bytes)
  - `READ_MAP`: Storage read coverage
  - `WRITE_MAP`: Storage write coverage  
  - `CMP_MAP`: Comparison operand distances
- These maps are updated during VM execution and read by feedback mechanisms

#### **Feedback** (`src/feedback.rs`)
- **LibAFL Concept**: `Feedback` determines if an execution is "interesting"
- **ItyFuzz Implementation**: Three types of feedback:

  1. **Coverage Feedback** (`MaxMapFeedback` from LibAFL):
     - Uses `JMP_MAP` to detect new code paths
     - If new coverage → input added to transaction corpus

  2. **CmpFeedback** (Comparison Feedback):
     - Tracks comparison distances in `CMP_MAP`
     - When operands get closer (smaller distance), votes for the VM state
     - Helps explore paths guarded by comparisons

  3. **DataflowFeedback**:
     - Tracks storage read/write patterns
     - Interesting if a written slot is later read with a new value

  4. **OracleFeedback** (Objective Feedback):
     - Executes vulnerability oracles (reentrancy, arbitrary transfer, etc.)
     - Returns `true` if a vulnerability is found → input added to solutions corpus

#### **Scheduler** (`src/scheduler.rs`)
- **LibAFL Concept**: `Scheduler` selects which input to fuzz next
- **ItyFuzz Implementation**: `SortedDroppingScheduler` with voting mechanism:
  - Each input/state has votes and visits
  - Selection is weighted by votes (probabilistic)
  - When corpus grows too large, drops low-scoring entries
  - Maintains dependency tree for garbage collection

**Two Schedulers**:
- **Main Scheduler**: Selects from transaction corpus (`PowerABIScheduler` for EVM)
- **Infant Scheduler**: Selects from infant state corpus (`SortedDroppingScheduler`)

#### **Mutator** (`src/evm/mutator.rs`)
- **LibAFL Concept**: `Mutator` generates new inputs from existing ones
- **ItyFuzz Implementation**: `FuzzMutator` mutates:
  - Transaction parameters (caller, contract, calldata)
  - ABI function arguments
  - VM state selection (from infant corpus)

#### **Stage** (`src/evm/scheduler.rs`)
- **LibAFL Concept**: `Stage` operates on a single input
- **ItyFuzz Implementation**: `PowerABIMutationalStage`:
  - Selects an input from corpus
  - Applies mutations
  - Executes mutated input
  - Feedback determines if it's interesting

### 3. Fuzzing Loop Flow

The main fuzzing loop (`src/fuzzer.rs` → `ItyFuzzer`) works as follows:

1. **Select Input**: Scheduler picks a transaction from corpus
2. **Select VM State**: Get a VM state from infant corpus (or use initial state)
3. **Create Input**: Combine transaction + VM state into `VMInput`
4. **Execute**: `FuzzExecutor` runs the transaction on the VM
   - Coverage maps are updated during execution
5. **Evaluate Feedback**:
   - **Infant Feedback** (`CmpFeedback`): Is the resulting VM state interesting?
     - If yes → add to infant corpus
   - **Coverage Feedback**: Did we hit new code paths?
     - If yes → add to transaction corpus
   - **Oracle Feedback**: Did we find a vulnerability?
     - If yes → add to solutions corpus and exit (or continue if `RUN_FOREVER`)
6. **Repeat**: Continue with next input

### 4. Key Design Patterns

#### **Infant State Corpus**
- VM states are treated as first-class inputs
- States that lead to interesting comparisons or new coverage are saved
- Mutations can start from any saved state
- Enables exploring deep execution paths

#### **Two-Level Feedback**
- **Transaction-level**: Coverage-based feedback for inputs
- **State-level**: Comparison/dataflow feedback for VM states
- States can be "sponsored" (given votes) if they lead to interesting results

#### **Oracle System** (`src/oracle.rs`)
- **Oracles**: Check for specific vulnerabilities (reentrancy, arbitrary transfer, etc.)
- **Producers**: Generate data needed by oracles (e.g., token balances)
- Oracles run after each execution and report bugs via `ORACLE_OUTPUT`

#### **How Oracles Find Vulnerabilities (Solutions)**

Oracles are the mechanism by which ItyFuzz detects vulnerabilities. They implement the `Oracle` trait and are executed after each non-reverted transaction execution through `OracleFeedback`.

**Oracle Execution Flow**:

1. **After Execution**: Once a transaction executes successfully (doesn't revert), `OracleFeedback.is_interesting()` is called
2. **Producer Phase**: First, all registered `Producer`s are executed to generate necessary data (e.g., fetching token balances, preparing test data)
3. **Oracle Phase**: Each oracle's `oracle()` method is called with an `OracleCtx` containing:
   - Pre-execution VM state
   - Post-execution VM state  
   - The executed input
   - Access to the executor for making additional calls
4. **Vulnerability Detection**: Oracles analyze the execution context and check for specific vulnerability patterns:
   - **ReentrancyOracle**: Detects reentrancy attacks by checking if external calls occur between storage reads and writes
   - **ArbitraryERC20TransferOracle**: Checks if arbitrary ERC20 token transfers occurred (unauthorized transfers)
   - **ArbitraryCallOracle**: Detects if arbitrary external calls were made
   - **SelfdestructOracle**: Checks if contracts were self-destructed
   - **EchidnaOracle**: Calls functions starting with `echidna_` and checks if they return false (invariant violation)
   - **InvariantOracle**: Calls functions starting with `invariant_` and checks if they fail
   - **TypedBugOracle**: Detects typed bugs based on execution patterns
5. **Bug Reporting**: When a vulnerability is found:
   - Oracle returns a `bug_idx` (unique identifier for the bug type)
   - `EVMBugResult` is created with bug type, description, and input
   - Result is pushed to `ORACLE_OUTPUT` global variable
   - Bug is registered in `BugMetadata` to avoid duplicate reports
6. **Solution Creation**: If any oracle returns a bug:
   - `OracleFeedback` returns `true` (execution is "interesting")
   - The input is added to the **Solutions Corpus** (not the regular corpus)
   - The fuzzer can exit (unless `RUN_FOREVER` is set) or continue to find more bugs
   - The input is minimized using `SequentialMinimizer` to create a minimal reproduction case

**Key Features**:
- Oracles can make additional calls to the VM state (`call_post_batch()`, `call_post_batch_dyn()`) to verify conditions
- Each oracle has a unique `bug_idx` to distinguish bug types
- Known bugs are tracked in `BugMetadata` to prevent duplicate reporting
- Oracles can have multiple stages (via `transition()` method) for complex vulnerability detection

#### **Middleware System** (`src/evm/middlewares/`)

Middlewares are hooks that execute during VM execution, allowing ItyFuzz to observe and modify execution behavior. They implement the `Middleware` trait and are called at specific points during transaction execution.

**Purpose**: Middlewares enable:
- **Observation**: Collecting execution data (coverage, call traces, reentrancy patterns)
- **Modification**: Altering execution behavior (cheatcodes, SHA3 bypass)
- **Integration**: Connecting external tools (on-chain data, concolic execution)

**Middleware Execution Points**:
- `before_execute()`: Called before executing a transaction
- `on_step()`: Called on each VM instruction/opcode
- `on_return()`: Called when a call returns
- `on_insert()`: Called when bytecode is inserted/deployed

**Available Middlewares**:

1. **Coverage** (`coverage.rs`): Records instruction-level and branch-level coverage for each contract, mapping program counters to source code locations for detailed coverage reporting.

2. **Cheatcode** (`cheatcode/`): Handles Foundry-style cheatcode calls (e.g., `vm.prank()`, `vm.warp()`, `vm.deal()`) by intercepting calls to a special cheatcode address and implementing the requested VM modifications.

3. **ReentrancyTracer** (`reentrancy.rs`): Tracks storage read/write patterns during execution to identify potential reentrancy vulnerabilities by monitoring if external calls occur between reads and writes.

4. **Sha3Bypass** (`sha3_bypass.rs`): Bypasses SHA3/Keccak256 hashing operations for symbolic/concolic execution by replacing hash results with symbolic values, enabling deeper exploration of hash-dependent code paths.

5. **Sha3TaintAnalysis** (part of `sha3_bypass.rs`): Performs taint analysis to track which input bytes influence SHA3 operations, helping identify which parts of input affect hash-dependent branches.

6. **CallPrinter** (`call_printer.rs`): Records and formats call traces during execution, generating human-readable transaction traces with source code mapping for debugging and reporting.

7. **OnChain** (in `evm/onchain/`): Fetches real on-chain contract code and state when contracts are accessed, enabling fuzzing of deployed contracts with their actual on-chain state.

8. **Flashloan** (in `evm/onchain/flashloan.rs`): Simulates flashloan attacks by providing temporary liquidity to contracts, enabling exploration of flashloan-based exploit scenarios.

**Middleware Order**: Middlewares are executed in registration order. The Cheatcode middleware should be first as it consumes cheatcode calls that shouldn't be visible to other middlewares.

### 5. EVM-Specific Implementation

The EVM implementation (`src/evm/`) provides:

- **`EVMExecutor`**: Implements `GenericVM` using `revm`
- **`EVMInput`**: EVM-specific transaction representation
- **`EVMState`**: EVM state (accounts, storage, etc.)
- **`FuzzHost`**: Custom EVM host that integrates middlewares
- **ABI Support**: Parses and mutates function calls using contract ABIs

### 6. Entry Point

`src/fuzzers/evm_fuzzer.rs` → `evm_fuzzer()`:
- Sets up all components (executor, feedbacks, schedulers, oracles)
- Initializes corpus from contracts
- Starts the fuzzing loop via `ItyFuzzer::fuzz_loop()`

## Deep Dive: Key Concepts for Security Researchers

### EVM Coverage Handling and JMP_MAP Feedback

**Coverage Maps**: ItyFuzz uses static arrays (4096 elements) to track execution coverage at the VM level. Unlike traditional fuzzers that track instruction coverage, ItyFuzz tracks:

- **`JMP_MAP`** (Jump Map): The primary coverage mechanism. Tracks which jump/branch edges have been executed.
- **`READ_MAP`**: Tracks which storage slots have been read.
- **`WRITE_MAP`**: Tracks which storage slots have been written.
- **`CMP_MAP`**: Tracks comparison operand distances (for comparison-guided fuzzing).

**How JMP_MAP Works**:

1. **During Execution**: When a `JUMPI` (conditional jump) opcode is executed in the EVM:
   - The program counter (PC) and jump destination are used to compute a hash: `idx = (PC * jump_dest) % MAP_SIZE`
   - The corresponding `JMP_MAP[idx]` is incremented (saturating add)
   - This creates a coverage "edge" representing the (PC, destination) pair

2. **Why JMP_MAP Instead of Instruction Coverage**:
   - **Branch Coverage**: JMP_MAP tracks which branches are taken, not just which instructions execute. This is more valuable for finding bugs as bugs often hide in specific branch paths.
   - **Efficiency**: A 4096-byte array is extremely fast to update and check, with minimal memory overhead.
   - **VM-Level Abstraction**: Works across all contracts in a transaction, not per-contract, making it suitable for multi-contract interactions.

3. **Feedback Mechanism**:
   - After execution, `MaxMapFeedback` (from LibAFL) checks if any `JMP_MAP[idx]` transitioned from 0 to non-zero
   - If new coverage is found → the input is marked as "interesting" and added to the transaction corpus
   - The coverage map is cleared between executions (it's volatile, as per LibAFL's Observer concept)

4. **Coverage Middleware**: The `Coverage` middleware also tracks instruction-level and branch-level coverage per contract for detailed reporting, but this is separate from the JMP_MAP used for feedback.

**Example**: If a contract has `if (balance > 100) { ... }`, the JMP_MAP will track whether the branch was taken (balance > 100) or not (balance <= 100). The fuzzer will try to explore both paths by mutating inputs to satisfy both conditions.

### On-Chain Fuzzing

On-chain fuzzing allows ItyFuzz to fuzz **deployed contracts** on real blockchains (Mainnet, Arbitrum, etc.) rather than just local bytecode.

**How It Works**:

1. **OnChain Middleware**: When enabled, the `OnChain` middleware intercepts EVM opcodes during execution:
   - **SLOAD (0x54)**: When a storage slot is read, fetches the actual value from the blockchain RPC
   - **CALL/CALLCODE/DELEGATECALL/STATICCALL**: When a contract is called, fetches the actual bytecode from the blockchain
   - **EXTCODESIZE/EXTCODECOPY/EXTCODEHASH**: Fetches contract code on-demand
   - **BALANCE (0x31)**: Fetches actual account balances
   - **TIMESTAMP (0x42)**: Uses real block timestamps
   - **CHAINID (0x46)**: Uses the actual chain ID

2. **Storage Fetching Modes**:
   - **`Dump`**: Fetches all storage slots at once (faster but more RPC calls)
   - **`OneByOne`**: Fetches storage slots on-demand as they're accessed (slower but fewer initial calls)

3. **Caching**: The middleware caches:
   - Contract bytecode (by address)
   - Storage slots (by address + slot)
   - ABIs (decompiled from bytecode using evmole)
   - To avoid redundant RPC calls

4. **Use Cases**:
   - **Fuzzing Production Contracts**: Test deployed contracts with their real on-chain state
   - **Integration Testing**: Fuzz contracts that interact with other on-chain protocols (Uniswap, Aave, etc.)
   - **State-Aware Fuzzing**: Explore contracts in their actual deployed state, not just initial state

5. **Limitations**:
   - Requires RPC endpoint access (Infura, Alchemy, etc.)
   - Slower than local fuzzing due to network latency
   - May hit RPC rate limits
   - Some addresses are blacklisted (e.g., problematic contracts)

**Example**: When fuzzing a DeFi protocol, the OnChain middleware will fetch real token balances, pool reserves, and contract code from the blockchain, allowing the fuzzer to explore realistic interaction scenarios.

### Feedback Stages: Infant, Coverage, and Oracle

ItyFuzz uses a **three-stage feedback system** that evaluates executions at different levels:

#### 1. Infant Feedback (State-Level)

**Purpose**: Determines if a VM state is interesting for further exploration, independent of coverage.

**When Evaluated**: After every execution, before coverage and oracle feedback.

**How It Works**:
- **CmpFeedback**: Tracks comparison distances in `CMP_MAP`
  - For each comparison (e.g., `if (balance > threshold)`), records the distance between operands
  - If a comparison gets "closer" (smaller distance), the state is interesting
  - Votes for the state in the infant scheduler
  - Example: If `threshold = 100` and `balance = 150`, distance is 50. If next execution has `balance = 120`, distance is 20 → interesting!

- **State Change Detection**: If the VM state changed (storage modified) or has post-execution, the state is interesting

- **Known State Filtering**: Maintains a hash set of seen states to avoid re-exploring identical states

**Result**: If interesting, the VM state is added to the **Infant State Corpus** for future mutation starting points.

**Why "Infant"**: These states are "infant" (newly discovered) and may lead to interesting paths if explored further.

#### 2. Coverage Feedback (Input-Level)

**Purpose**: Determines if an input (transaction) achieved new code coverage.

**When Evaluated**: After infant feedback, only if execution didn't find a solution.

**How It Works**:
- Uses `MaxMapFeedback` from LibAFL
- Checks if `JMP_MAP` has any new non-zero entries compared to previous executions
- If new coverage → input is interesting

**Result**: If interesting, the input is added to the **Transaction Corpus** for future mutations.

**Note**: Coverage feedback is evaluated on the **input**, not the state. Multiple inputs can reach the same state, but only inputs with new coverage are kept.

#### 3. Oracle Feedback (Solution Detection)

**Purpose**: Detects actual vulnerabilities (solutions).

**When Evaluated**: After every non-reverted execution, in parallel with infant feedback.

**How It Works**:
- Executes all registered oracles (ReentrancyOracle, ArbitraryTransferOracle, etc.)
- Each oracle analyzes the execution context and checks for vulnerability patterns
- If any oracle finds a bug → returns `true`

**Result**: If interesting (vulnerability found):
- Input is added to the **Solutions Corpus** (not regular corpus)
- Bug is reported via `ORACLE_OUTPUT`
- Input is minimized to create a minimal reproduction case
- Fuzzer can exit (unless `RUN_FOREVER` is set)

**Priority**: Oracle feedback takes precedence. If a solution is found, coverage feedback is skipped.

**Execution Order** (from `evaluate_input_events` in `fuzzer.rs`):
```
1. Execute transaction
2. Infant Feedback → Add to infant corpus if interesting
3. Oracle Feedback → Check for vulnerabilities
4. If no solution:
   5. Coverage Feedback → Add to transaction corpus if new coverage
```

### Presets and Exploit Templates

Presets are **exploit templates** that guide the fuzzer to explore specific attack patterns or function call sequences.

**What Are Presets**:

- **ExploitTemplate**: A JSON structure defining:
  - `exploit_name`: Name of the exploit pattern
  - `function_sigs`: Required function signatures that must exist in contracts
  - `calls`: Function signatures to call in sequence

**How They Work**:

1. **Template Matching**: During corpus initialization (`evm_fuzzer.rs`):
   - Loads exploit templates from a JSON file
   - Matches templates against deployed contracts by checking if all `function_sigs` exist
   - Creates a mapping: `function_sig → (address, ABI)`

2. **State Initialization**: If templates match:
   - `state.init_presets()` stores matched templates and the sig-to-address mapping
   - `interesting_signatures` list is populated with function signatures to prioritize

3. **Mutation Guidance**: During mutation:
   - `state.has_preset()` checks if presets are available
   - `state.get_next_call()` randomly selects a function signature from `interesting_signatures`
   - Mutator can use this to guide transaction generation toward exploit patterns

4. **Example Preset** (`pair.rs`):
   ```rust
   // For Uniswap V2 pair exploits
   // Function sig: 0xbc25cf77 (some pair function)
   // When this function is called, preset modifies the input
   // to call it 37 times (repeat = 37)
   ```

**Use Cases**:
- **Known Exploit Patterns**: Guide fuzzer to explore known vulnerability patterns (reentrancy, flashloan attacks, etc.)
- **Protocol-Specific**: Target specific DeFi protocols (Uniswap, Aave) with known interaction patterns
- **Research**: Test hypotheses about exploit sequences

**Limitations**:
- Requires knowing function signatures in advance
- Only guides mutation, doesn't guarantee finding exploits
- Must be enabled via `use_presets` feature flag

**File Format** (JSON):
```json
[
  {
    "exploit_name": "Uniswap V2 Pair Exploit",
    "function_sigs": ["0xbc25cf77"],
    "calls": ["0xbc25cf77"]
  }
]
```

### Solutions: Vulnerability Reporting and Minimization

When an oracle detects a vulnerability, ItyFuzz creates a **Solution** - a minimal, reproducible test case.

**Solution Creation Process**:

1. **Detection**: Oracle returns a `bug_idx` indicating a vulnerability was found

2. **Registration**: 
   - Bug is registered in `BugMetadata` with the corpus index
   - `EVMBugResult` is created with:
     - `bug_type`: Type of vulnerability (e.g., "Reentrancy", "ArbitraryTransfer")
     - `bug_info`: Human-readable description
     - `input`: The concise input that triggered the bug
     - `bug_idx`: Unique identifier
     - `sourcemap`: Optional source code mapping

3. **Minimization**: `SequentialMinimizer` reduces the input to a minimal reproduction:
   - Removes unnecessary transactions
   - Simplifies transaction parameters
   - Keeps only the essential sequence that triggers the bug

4. **Output Generation**:
   - **Console**: Prints bug description and transaction trace
   - **JSON**: Writes to `vuln_info.jsonl` with all bug details
   - **Foundry Test**: Generates a Foundry test file (`.t.sol`) that can be run to reproduce the bug
   - **Replayable Format**: Saves concise input format for replay

5. **Test File Generation** (`solution/mod.rs`):
   - Uses Handlebars template (`foundry_test.hbs`)
   - Generates a complete Foundry test with:
     - Setup code (forking blockchain if on-chain)
     - Transaction sequence
     - Assertions
     - Can be run directly with `forge test`

**Solution Corpus**: Solutions are stored separately from the regular corpus in `OnDiskCorpus` at the `solutions/` directory.

**Why Solutions Matter**:
- **Reproducibility**: Minimal test cases can be shared and verified
- **Integration**: Foundry tests can be added to test suites
- **Documentation**: Clear evidence of vulnerabilities for bug reports

## For Developers

### Where to Start

1. **Understanding the Flow**: Start with `evm_fuzzer.rs` to see how components are wired together
2. **Adding Oracles**: Implement `Oracle` trait in `src/evm/oracles/` to detect new vulnerability types
3. **Adding Middlewares**: Implement `Middleware` trait to hook into VM execution
4. **Custom Feedback**: Extend feedback mechanisms in `src/feedback.rs`
5. **VM Support**: Implement `GenericVM` and `VMStateT` for new VMs

### Key Files Reference

- **Fuzzer Core**: `src/fuzzer.rs` - Main fuzzing loop and input evaluation
- **Execution**: `src/executor.rs` - Wraps VM execution
- **Feedback Logic**: `src/feedback.rs` - Determines interestingness
- **State Management**: `src/state.rs` - Corpus and state management
- **Scheduling**: `src/scheduler.rs` - Input/state selection
- **VM Interface**: `src/generic_vm/` - Generic VM abstraction
- **EVM Implementation**: `src/evm/` - EVM-specific code

### Important Concepts

- **Coverage Maps**: Static arrays (4096 elements) tracking execution coverage
- **Infant States**: VM states saved for future exploration
- **Voting System**: States/inputs get votes based on interestingness
- **Two Corpora**: Separate storage for transactions vs. VM states
- **Staged Execution**: Transactions can be incomplete, requiring post-execution steps

