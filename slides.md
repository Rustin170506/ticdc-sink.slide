---
theme: seriph
class: 'text-center'
highlighter: shiki
lineNumbers: false
info: |
  TiCDC Sink Component.

  Learn more at [TiFlow](https://github.com/pingcap/tiflow/tree/master/cdc/sink)
drawings:
  persist: false
---

# TiCDC Sink Component

Focus on MQ Sink.

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Begin <carbon:arrow-right class="inline"/>
  </span>
</div>

---
layout: center
---

# Table pipeline
<br>

# Sink Interface
<br>

# Sink Implement
<br>

# MQ Sink


<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---

<div class="arch">
<div>

# Architecture

</div>

<div>

```plantuml {scale: 0.9}
@startuml

package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Owner" {
  [OwnerDDLPuller] --> [gRPC]
  [DDLSink]
  [DDLMounter]
  [Scheduler]
}

node "Processor" {
  [ProcessorDDLPuller] --> [gRPC]
  [ProcessorDDLMounter]
  [ProcessorMounter]
  [ProcessorSink]
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] --> [gRPC]
      [Sorter1]
      [TableSink1]
    }
    package "Table2 Pipeline" {
      [Puller2] --> [gRPC]
      [Sorter2]
      [TableSink2]
    }
  }
}

database "MySQL/Kafka" {
  [MySQL]
  [Broker]
}

[DDLSink] --> [MySQL]
[DDLSink] --> [Broker]
[OwnerDDLPuller] --> [DDLSink]
[OwnerDDLPuller] ..> [DDLMounter] : use
[ProcessorSink] --> [MySQL]
[ProcessorSink] --> [Broker]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> [ProcessorSink] : use
[TableSink2] ..> [ProcessorSink] : use
[ProcessorDDLPuller] --> [ProcessorSink]
[ProcessorDDLPuller] ..> [ProcessorDDLMounter] : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.arch {
  display: flex;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

---

# Table Pipeline

Each changefeed creates a processor, and each processor maintains a table pipeline.

### Pipeline
<br>
<br>

```mermaid {scale: 2}
flowchart LR
    puller((Puller)) --> sorter((Sorter)) --> mounter((Mounter)) --> sink((Sink))
```

---

# Puller

Pull DDL and Row Change data from TiKV.

<div class="puller">

<div>

| Region1       | Region2       |
| ------------- | ------------- |
|               | ts1: C -> 2   |
| ts2: A -> 6   | ts1: Resolved |
| ts1: B -> 4   |               |
| ts1: Resolved |               |
| ts2: B -> 3   | ts2: C ->3    |
| ts2: Resolved |               |
| ts3: A -> 7   |               |

</div>

<div v-click >
<arrow x1="400" y1="300" x2="600" y2="300" width="3" arrowSize="1" />
</div>

<div v-click >

| Output Chan   |
| ------------- |
| ts1: C -> 2   |
| ts2: A -> 6   |
| ts1: B -> 4   |
| ts1: Resolved |
| ts2: B ->3    |
| ts2: C ->3    |
| ts2: Resolved |
| ts3: A -> 7   |

</div>

</div>

<style>
.puller {
  margin-left: 50px;
  margin-right: 100px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>

---

# Sorter

<div grid="~ cols-3 gap-4">
<div>

| To Sort       |
| ------------- |
| ts1: C -> 2   |
| ts2: A -> 6   |
| ts1: B -> 4   |
| ts1: Resolved |
| ts2: B ->3    |
| ts2: C ->3    |
| ts2: Resolved |
| ts3: A -> 7   |

</div>

<div v-click >
<arrow x1="360" y1="300" x2="600" y2="300" width="3" arrowSize="1" />
</div>

<div v-click >

| Output Chan   |
| ------------- |
| ts1: C -> 2   |
| ts1: B -> 4   |
| ts1: Resolved |
| ts2: A -> 6   |
| ts2: B ->3    |
| ts2: C ->3    |
| ts2: Resolved |

</div>
</div>


---
class: px-20
---

# Mounter

Mounter will use the schema information to convert the row kv
into row changes that TiCDC can handle.

<br/>
<br/>
<br/>

<div grid="~ cols-2 gap-4">
<div>

```go {all|5,7}
type RawKVEntry struct {
	OpType OpType 
	Key    []byte 
	// nil for delete type
	Value []byte 
	// nil for insert type
	OldValue []byte 
	StartTs  uint64 
	// Commit or resolved TS
	CRTs uint64 
	// Additional debug info
	RegionID uint64 
}
```

</div>

<div>

```go {all|9,10}
type RowChangedEvent struct {
	StartTs  uint64 
	CommitTs uint64
	RowID int64 
	Table    *TableName
	ColInfos []rowcodec.ColInfo
	TableInfoVersion uint64 
	ReplicaID    uint64    
	Columns      []*Column 
	PreColumns   []*Column 
	IndexColumns [][]int   
	ApproximateDataSize int64
}
```

</div>
</div>

---

# Sink

Sink is responsible for sending data to MySQL or Kafka.

<div class="sink">

```mermaid {scale: 2}
flowchart LR
    sink((Sink)) --> |database/sql|mysql((MySQL))
    sink((Sink)) --> |Sarama|kafka((Kafka))
    style kafka fill:#f96
```

</div>

<style>
.sink {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>

---

# Sink Interface

<br/>
<br/>
<br/>

```go {all|6}
type Sink interface {
	EmitRowChangedEvents(ctx context.Context, rows ...*model.RowChangedEvent) error

	EmitDDLEvent(ctx context.Context, ddl *model.DDLEvent) error

	FlushRowChangedEvents(ctx context.Context, tableID model.TableID, resolvedTs uint64) (uint64, error)

	// Only for MQ Sink.
	EmitCheckpointTs(ctx context.Context, ts uint64, tables []model.TableName) error

	Close(ctx context.Context) error

	// Only for MySQL Sink.
	Barrier(ctx context.Context, tableID model.TableID) error
}
```

---

# Sink Implement

<br/>
<br/>
<br/>

<div grid="~ cols-2 gap-4">

<div>

## Processor Level Sink
<br/>

- BlackHole Sink: Do nothing
- MQSink: For MQ
- MySQLSink: For MySQL
- Buffer Sink: Buffer + Asynchronously

</div>
<div>

## Table Level Sink
<br/>

- Table Sink: Sink Minimum Management Unit

</div>

</div>

---

<div class="relation">

<div class="title">

# Relations

</div>
<div class="uml">

```plantuml {scale: 1}
@startuml

package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Processor" {
  [ProcessorMounter]
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] --> [gRPC]
      [Sorter1]
      [TableSink1] #Yellow
    }
    package "Table2 Pipeline" {
      [Puller2] --> [gRPC]
      [Sorter2]
      [TableSink2] #Yellow
    }
    package "Sink Manager" {
      folder "Combination" as Combination {
        [MQSink] #FF6655
        [BufferSink] #FF6655
      }
    }
  }
}

database "Kafka" {
  [Broker]
}

note right of [MQSink]
  It can be either 
  MySQL Sink or BlackHoleSink.
end note

Combination --> [Broker]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> Combination : use
[TableSink2] ..> Combination : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.relation {
  display: flex;
  justify-content: flex-start;
}

.relation img {
  height: 500px;
}

.relation .title {
  flex-grow: 4;
}

.relation .uml {
  flex-grow: 2;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

---

# Data Sequence

## Row Change Data Sequence

<br/>

```plantuml {scale: 0.9}
@startuml
SinkNode -> TableSink: calls emitRowToSink
SinkNode <-- TableSink: Added to buffer
SinkNode -> TableSink: SinkNode calls FlushRowChangedEvents
TableSink -> ProcessorSink: calls EmitRowChangedEvents
TableSink <-- ProcessorSink: Added to buffer
loop BufferSink
  ProcessorSink -> ProcessorSink: BufferSink cache is full
  ProcessorSink -> Producer: calls EmitRowChangedEvents of MQSink
end
Producer -> LeaderBroker: Async send
Producer <-- LeaderBroker: ACK

note over of ProcessorSink
  It is a combination of BufferSink and MQSink.
end note
@enduml
```

---

# Data Sequence

<div class="seq">
<div>

## Resolved TS Flush Sequence

</div>
<div>

```plantuml {scale: 0.8}
@startuml
SinkNode -> TableSink: calls flushSink
TableSink -> ProcessorSink: calls flushBackendSink
TableSink <-- ProcessorSink: flush msg sent
Producer -> LeaderBroker: Async send
Producer <-- LeaderBroker: ACK
loop BufferSink
  ProcessorSink -> ProcessorSink: BufferSink flush cache is full and calls FlushRowChangedEvents of MQSink
end
loop MQSink
  ProcessorSink -> ProcessorSink: bgFlushTs receives Resolved TS
  group Flush 
    ProcessorSink -> Producer: calls Flush
    Producer -> Producer: Waiting for all msgs ACK
    ProcessorSink <-- Producer: Flushed
  end
end
note over of ProcessorSink
  It is a combination of BufferSink and MQSink.
end note
@enduml
```

</div>
</div>

<style>
.seq {
  display: flex;
}

h2 {
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>