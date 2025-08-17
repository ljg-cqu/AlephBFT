# AlephBFT Consensus Mechanism Explained

AlephBFT is an asynchronous Byzantine Fault Tolerant consensus protocol that orders arbitrary messages through a DAG (Directed Acyclic Graph) structure. This document provides a comprehensive explanation with visual diagrams.

## Overview

AlephBFT allows N nodes to agree on an ordering of data items, tolerating up to f = ⌊(N-1)/3⌋ malicious nodes. The protocol is asynchronous, meaning it doesn't rely on timing assumptions for safety.

## 1. High-Level Architecture

```mermaid
graph TB
    subgraph "Node Architecture"
        DP[Data Provider] --> Creator[Unit Creator]
        Creator --> DAG[DAG Structure]
        DAG --> Ordering[Ordering Algorithm]
        Ordering --> FH[Finalization Handler]
        
        Network[Network Layer] <--> DAG
        Alerts[Alert System] <--> DAG
        Backup[Backup System] <--> DAG
    end
    
    subgraph "External Components"
        Input[Input Stream] --> DP
        FH --> Output[Output Stream]
        Storage[Persistent Storage] <--> Backup
    end
```

## 2. Unit Structure and DAG Construction

### Unit Components

```mermaid
graph LR
    subgraph "Unit Structure"
        U[Unit] --> Creator[Creator ID]
        U --> Round[Round Number]
        U --> Data[Data Item]
        U --> ParentMap[Parent Map]
        U --> ControlHash[Control Hash]
        U --> Signature[Digital Signature]
    end
    
    subgraph "Parent Relationships"
        U1[Unit R1] --> U0A[Unit R0-A]
        U1 --> U0B[Unit R0-B]
        U1 --> U0C[Unit R0-C]
        U2[Unit R2] --> U1
        U2 --> U1B[Unit R1-B]
        U2 --> U1C[Unit R1-C]
    end
```

### DAG Growth Pattern

```mermaid
graph TB
    subgraph "Round 0 (Genesis)"
        U0A[Node A Unit 0]
        U0B[Node B Unit 0]
        U0C[Node C Unit 0]
        U0D[Node D Unit 0]
    end
    
    subgraph "Round 1"
        U1A[Node A Unit 1]
        U1B[Node B Unit 1]
        U1C[Node C Unit 1]
        U1D[Node D Unit 1]
    end
    
    subgraph "Round 2"
        U2A[Node A Unit 2]
        U2B[Node B Unit 2]
        U2C[Node C Unit 2]
        U2D[Node D Unit 2]
    end
    
    %% Parent relationships
    U1A --> U0A
    U1A --> U0B
    U1A --> U0C
    U1B --> U0A
    U1B --> U0B
    U1B --> U0D
    U1C --> U0B
    U1C --> U0C
    U1C --> U0D
    U1D --> U0A
    U1D --> U0C
    U1D --> U0D
    
    U2A --> U1A
    U2A --> U1B
    U2A --> U1C
    U2B --> U1A
    U2B --> U1B
    U2B --> U1D
    U2C --> U1B
    U2C --> U1C
    U2C --> U1D
    U2D --> U1A
    U2D --> U1C
    U2D --> U1D
```

## 3. Unit Creation Rules

```mermaid
flowchart TD
    Start([Start Unit Creation]) --> CheckRound{Round = 0?}
    CheckRound -->|Yes| CreateGenesis[Create Genesis Unit]
    CheckRound -->|No| CheckPrevious{Previous unit exists?}
    
    CheckPrevious -->|No| Wait1[Wait for previous unit]
    CheckPrevious -->|Yes| CheckThreshold{≥ N-f units in previous round?}
    
    CheckThreshold -->|No| Wait2[Wait for more units]
    CheckThreshold -->|Yes| SelectParents[Select ≥ N-f parents from previous round]
    
    SelectParents --> GetData[Poll data from input stream]
    GetData --> ComputeHash[Compute control hash]
    ComputeHash --> Sign[Sign unit]
    Sign --> Broadcast[Broadcast to all nodes]
    Broadcast --> Delay[Sleep CREATE_DELAY]
    Delay --> NextRound[Increment round]
    NextRound --> CheckRound
    
    CreateGenesis --> Broadcast
    Wait1 --> CheckPrevious
    Wait2 --> CheckThreshold
```

## 4. Consensus Algorithm Flow

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    participant N4 as Node 4
    
    Note over N1,N4: Round 0 - Genesis Units
    N1->>N1: Create Unit(0,1)
    N2->>N2: Create Unit(0,2)
    N3->>N3: Create Unit(0,3)
    N4->>N4: Create Unit(0,4)
    
    N1->>N2: Broadcast Unit(0,1)
    N1->>N3: Broadcast Unit(0,1)
    N1->>N4: Broadcast Unit(0,1)
    
    Note over N1,N4: Similar broadcasts for all nodes...
    
    Note over N1,N4: Round 1 - After receiving ≥ N-f units
    N1->>N1: Create Unit(1,1) with parents
    N2->>N2: Create Unit(1,2) with parents
    N3->>N3: Create Unit(1,3) with parents
    N4->>N4: Create Unit(1,4) with parents
    
    Note over N1,N4: Continue DAG construction...
    
    Note over N1,N4: Ordering Phase
    N1->>N1: Run OrderData algorithm
    N2->>N2: Run OrderData algorithm
    N3->>N3: Run OrderData algorithm
    N4->>N4: Run OrderData algorithm
```

## 5. Ordering Algorithm (Head Selection)

```mermaid
flowchart TD
    Start([OrderData Algorithm]) --> InitRound[r = 0]
    InitRound --> FindHead{Find Head r}
    
    FindHead -->|None| End([Return ordered sequence])
    FindHead -->|Some U| GetBelow[Get all units below U]
    
    GetBelow --> RemovePrevious[Remove units from previous batches]
    RemovePrevious --> SortBatch[Sort remaining units canonically]
    SortBatch --> AddToBatch[Add to batch r]
    AddToBatch --> ExtractData[Extract data from units]
    ExtractData --> AppendOrder[Append to order sequence]
    AppendOrder --> IncrementRound[r = r + 1]
    IncrementRound --> FindHead
```

### Head Selection Process

```mermaid
flowchart TD
    StartHead([Head Selection for Round r]) --> CheckHeight{DAG height >= r+3?}
    CheckHeight -->|No| ReturnNone[Return None]
    CheckHeight -->|Yes| GetUnits[Get units in round r]
    
    GetUnits --> SortUnits[Sort units canonically]
    SortUnits --> TestFirst[Test first unit]
    
    TestFirst --> Decide{Decide unit?}
    Decide -->|None| ReturnNone
    Decide -->|Some true| ReturnUnit[Return this unit as head]
    Decide -->|Some false| NextUnit[Try next unit]
    
    NextUnit --> MoreUnits{More units?}
    MoreUnits -->|Yes| TestFirst
    MoreUnits -->|No| ReturnNone
    
    ReturnUnit --> HeadFound([Head found])
```

## 6. Virtual Voting Mechanism

```mermaid
graph TB
    subgraph "Virtual Voting for Unit U in Round r"
        U[Unit U - Round r]
        
        subgraph "Round r+1 Voters"
            V1[Unit V1]
            V2[Unit V2]
            V3[Unit V3]
            V4[Unit V4]
        end
        
        subgraph "Round r+2 Voters"
            W1[Unit W1]
            W2[Unit W2]
            W3[Unit W3]
            W4[Unit W4]
        end
        
        subgraph "Round r+3 Voters (Decision Round)"
            X1[Unit X1]
            X2[Unit X2]
            X3[Unit X3]
            X4[Unit X4]
        end
    end
    
    U -.-> V1
    U -.-> V2
    U -.-> V3
    U -.-> V4
    
    V1 --> W1
    V2 --> W2
    V3 --> W3
    V4 --> W4
    
    W1 --> X1
    W2 --> X2
    W3 --> X3
    W4 --> X4
    
    X1 --> Decision[Final Decision]
    X2 --> Decision
    X3 --> Decision
    X4 --> Decision
```

### Voting Rules

```mermaid
flowchart TD
    StartVote([Vote(U, V)]) --> CheckDiff{round_diff = V.round - U.round}
    
    CheckDiff -->|= 1| DirectParent{U ∈ parents(V)?}
    DirectParent -->|Yes| ReturnTrue[Return true]
    DirectParent -->|No| ReturnFalse[Return false]
    
    CheckDiff -->|> 1| GetParentVotes[Get votes from parents(V)]
    GetParentVotes --> AllSame{All votes same?}
    
    AllSame -->|Yes| ReturnVote[Return that vote]
    AllSame -->|No| CommonVote[Return CommonVote(round_diff)]
    
    CommonVote --> CheckRoundDiff{round_diff ≤ 4?}
    CheckRoundDiff -->|Yes, = 3| ReturnFalse
    CheckRoundDiff -->|Yes, ≠ 3| ReturnTrue
    CheckRoundDiff -->|No| ReturnParity[Return (round_diff % 2) == 1]
```

## 7. Fork Detection and Alert System

```mermaid
sequenceDiagram
    participant N1 as Honest Node 1
    participant N2 as Honest Node 2
    participant N3 as Malicious Node 3
    participant N4 as Honest Node 4
    
    Note over N1,N4: Normal Operation
    N3->>N1: Unit A (Round 1)
    N3->>N2: Unit B (Round 1) - FORK!
    
    N1->>N2: Forward Unit A
    N2->>N1: Forward Unit B
    
    Note over N1,N4: Fork Detection
    N1->>N1: Detect fork (A ≠ B, same round/creator)
    N2->>N2: Detect fork (A ≠ B, same round/creator)
    
    Note over N1,N4: Alert Broadcasting
    N1->>N2: ForkAlert(N3, proof=(A,B), units=[...])
    N1->>N4: ForkAlert(N3, proof=(A,B), units=[...])
    
    N2->>N1: ForkAlert(N3, proof=(A,B), units=[...])
    N2->>N4: ForkAlert(N3, proof=(A,B), units=[...])
    
    Note over N1,N4: Reliable Broadcast ensures all honest nodes get alerts
```

### Alert Processing

```mermaid
flowchart TD
    ReceiveUnit[Receive Unit] --> CheckFork{Fork detected?}
    CheckFork -->|No| AddToDAG[Add to DAG normally]
    CheckFork -->|Yes| CreateAlert[Create ForkAlert]
    
    CreateAlert --> ReliableBroadcast[Reliable Broadcast Alert]
    ReliableBroadcast --> MarkForker[Mark node as forker]
    MarkForker --> FilterUnits[Only accept committed units from forker]
    
    ReceiveAlert[Receive ForkAlert] --> ValidateProof{Valid fork proof?}
    ValidateProof -->|No| Discard[Discard alert]
    ValidateProof -->|Yes| AcceptCommitted[Accept committed units]
    AcceptCommitted --> UpdateForkerList[Update forker list]
```

## 8. Complete Protocol Flow

```mermaid
graph TB
    subgraph "Input Layer"
        DataStream[Data Stream] --> DataProvider[Data Provider]
    end
    
    subgraph "Consensus Core"
        DataProvider --> Creator[Unit Creator]
        Creator --> |New Units| DAG[DAG Manager]
        
        Network[Network Hub] <--> |Units/Alerts| DAG
        DAG --> |Fork Detection| AlertSystem[Alert System]
        AlertSystem --> |Reliable Broadcast| Network
        
        DAG --> |Ordered Units| Consensus[Consensus Handler]
        Consensus --> |Finalized Data| FinalizationHandler[Finalization Handler]
    end
    
    subgraph "Persistence Layer"
        BackupSaver[Backup Saver] <--> |Save/Load| Storage[Persistent Storage]
        DAG <--> BackupSaver
    end
    
    subgraph "Output Layer"
        FinalizationHandler --> OutputStream[Ordered Output Stream]
    end
    
    subgraph "Validation"
        Validator[Unit Validator] --> DAG
        Validator --> |Signature Check| Keychain[Multi-Keychain]
    end
```

## Key Properties

### Safety Properties
- **Consistency**: All honest nodes produce the same ordering (one is prefix of another)
- **Validity**: Only data from honest nodes appears in output
- **Integrity**: No data corruption or unauthorized modifications

### Liveness Properties  
- **Termination**: Protocol continues to make progress
- **Fairness**: All honest nodes' data eventually gets ordered

### Byzantine Fault Tolerance
- Tolerates up to f = ⌊(N-1)/3⌋ malicious nodes
- Handles various attack scenarios including fork bombs
- Asynchronous safety (no timing assumptions needed for correctness)

## Implementation Highlights

1. **Asynchronous Design**: No reliance on synchronized clocks or timing assumptions
2. **DAG-based Structure**: Efficient parallel processing of units
3. **Virtual Voting**: Deterministic decision making without explicit voting rounds  
4. **Fork Resilience**: Robust handling of malicious behavior through alert system
5. **Crash Recovery**: Persistent storage enables recovery from failures
6. **Modular Architecture**: Clean separation of concerns with well-defined interfaces

This consensus mechanism provides a robust foundation for blockchain and distributed systems requiring strong consistency guarantees in adversarial environments.
