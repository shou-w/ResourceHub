## Recovery Flow for Each Container Failure 🔄

### 1. KnowledgeController Service Container Failure 🛑

```mermaid
graph TD
    A[KnowledgeController stops during processing] --> B[RabbitMQ detects connection loss]
    B --> C[Returns in-process messages to 'Ready' state]
    C --> D[New KnowledgeController container starts]
    D --> E[Reconnects to RabbitMQ]
    E --> F[Retrieves messages and resumes processing]
    
    G[If transaction was in progress on pgvector] --> H[Automatic rollback]
    H --> F
    
    style A fill:#ff6b6b
    style D fill:#51cf66
    style F fill:#4dabf7
```

**Impact Scope**: Only files being processed
**Data Loss**: None (rolled back by transaction)
**Recovery Time**: Several seconds to tens of seconds

---

### 2. RabbitMQ Container Failure 📬

```mermaid
graph TD
    A[RabbitMQ stops] --> B[Cannot accept new messages]
    B --> C[KnowledgeController service detects connection error]
    C --> D[KnowledgeController keeps retrying connection]
    
    E[RabbitMQ restarts] --> F[Restores persisted messages]
    F --> G[KnowledgeController automatically reconnects]
    G --> H[Resumes processing from pending messages]
    
    style A fill:#ff6b6b
    style E fill:#51cf66
    style H fill:#4dabf7
```

**Impact Scope**: New file processing stops
**Data Loss**: None (messages are persisted)
**Recovery Time**: Depends on RabbitMQ startup time

---

### 3. pgvector Container Failure 💾

```mermaid
graph TD
    A[pgvector stops] --> B[DB write errors occur]
    B --> C[Ongoing transactions automatically rollback]
    C --> D[KnowledgeController detects error]
    D --> E[Sends NACK to RabbitMQ]
    E --> F[Messages remain in queue]
    
    G[pgvector restarts] --> H[KnowledgeController reconnects]
    H --> I[Starts processing pending messages]
    I --> J[Resumes with guaranteed data consistency]
    
    style A fill:#ff6b6b
    style G fill:#51cf66
    style J fill:#4dabf7
```

**Impact Scope**: All DB write operations
**Data Loss**: None (protected by transactions)
**Recovery Time**: DB startup time + connection pool reconstruction

---

### 4. Redis Container Failure 🔴

```mermaid
graph TD
    A[Redis stops] --> B[Status info read/write unavailable]
    B --> C[Processing can continue]
    C --> D[But status check API unavailable]
    
    E[Redis restarts] --> F{Persistence enabled?}
    F -->|AOF/RDB enabled| G[Restores from saved data]
    F -->|No persistence| H[Status information lost]
    
    G --> I[Normal operation resumes]
    H --> J[Processing file status unknown]
    J --> K[Completed data verifiable in pgvector]
    
    style A fill:#ff6b6b
    style E fill:#51cf66
    style I fill:#4dabf7
```

**Impact Scope**: Status check functionality only
**Data Loss**: Depends on persistence settings (main data unaffected)
**Recovery Time**: Immediate (no persistence) ~ few seconds (with persistence)

---

### Multiple Container Failures 🔥

| Failure Pattern | Impact | Recovery Priority |
|-----------------|--------|-------------------|
| pgvector + Redis | All processing stops | 1. pgvector → 2. Redis |
| RabbitMQ + KnowledgeController | All processing stops | 1. RabbitMQ → 2. KnowledgeController |
| All containers | Complete system down | 1. pgvector → 2. Redis → 3. RabbitMQ → 4. KnowledgeController |

### Key Points 📌

1. **Data Integrity**: Data consistency is guaranteed in all patterns
2. **Automatic Recovery**: All cases can auto-recover without manual intervention
3. **Idempotent Processing**: Processing the same file multiple times yields the same result
4. **Observability**: Final state can be verified from pgvector even if Redis is down