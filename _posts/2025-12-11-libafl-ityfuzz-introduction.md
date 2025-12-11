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


