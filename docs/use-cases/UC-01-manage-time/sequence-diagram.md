# Sequence Diagram - UC-01: Manage Time

## Overview

This document illustrates the interactions between different system components and actors during time entry management operations. The sequence diagrams show the message flow and timing of operations.

---

## Main Flow: Create Time Entry

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant Validator as Validation Service
    participant TimeService as Time Entry Service
    participant DB as Database
    participant Cache as Cache Service
    participant Audit as Audit Service
    participant Queue as Message Queue

    User->>UI: Click "Add Time Entry"
    activate UI
    UI->>UI: Display Time Entry Form
    UI-->>User: Show Form
    deactivate UI

    User->>UI: Enter Details & Submit
    activate UI
    UI->>UI: Client-side Validation
    
    UI->>API: POST /api/v1/time-entries
    activate API
    
    API->>Auth: Validate JWT Token
    activate Auth
    Auth->>Auth: Verify Token Signature
    Auth->>Auth: Check Expiration
    Auth-->>API: Token Valid (User ID)
    deactivate Auth
    
    API->>Validator: Validate Request Data
    activate Validator
    Validator->>Validator: Check Required Fields
    Validator->>Validator: Validate Time Range
    Validator->>Validator: Check Duration Limits
    Validator-->>API: Validation Passed
    deactivate Validator
    
    API->>TimeService: createTimeEntry(data)
    activate TimeService
    
    TimeService->>DB: Check for Overlapping Entries
    activate DB
    DB-->>TimeService: No Overlap Found
    deactivate DB
    
    TimeService->>TimeService: Calculate Duration
    TimeService->>TimeService: Set Metadata (created_by, timestamps)
    
    TimeService->>DB: BEGIN TRANSACTION
    activate DB
    
    TimeService->>DB: INSERT INTO time_entry
    DB-->>TimeService: Entry Created (ID: 123)
    
    TimeService->>DB: INSERT INTO time_entry_tag (if tags present)
    DB-->>TimeService: Tags Associated
    
    TimeService->>DB: COMMIT TRANSACTION
    deactivate DB
    
    TimeService->>Audit: logAction(CREATE, entryId, userId, newData)
    activate Audit
    Audit->>DB: INSERT INTO audit_log
    activate DB
    DB-->>Audit: Audit Logged
    deactivate DB
    deactivate Audit
    
    TimeService->>Cache: invalidateUserTimeCache(userId)
    activate Cache
    Cache-->>TimeService: Cache Cleared
    deactivate Cache
    
    TimeService->>Queue: Publish TimeEntryCreated Event
    activate Queue
    Queue-->>TimeService: Event Published
    deactivate Queue
    
    TimeService-->>API: Success (Entry Data)
    deactivate TimeService
    
    API-->>UI: 201 Created (Entry JSON)
    deactivate API
    
    UI->>UI: Update UI with New Entry
    UI-->>User: Display Success Message
    deactivate UI
    
    Note over Queue: Async Processing
    Queue->>Queue: Process Event (Analytics, Notifications)
```

---

## Alternative Flow: Timer Mode

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant TimeService as Time Entry Service
    participant Cache as Cache/Redis
    participant DB as Database
    participant Audit as Audit Service

    User->>UI: Click "Start Timer"
    activate UI
    
    UI->>API: POST /api/v1/timer/start
    activate API
    
    API->>Auth: Validate Token
    activate Auth
    Auth-->>API: Valid (User ID)
    deactivate Auth
    
    API->>Cache: CHECK active_timer:userId
    activate Cache
    Cache-->>API: No Active Timer
    deactivate Cache
    
    API->>TimeService: startTimer(userId)
    activate TimeService
    
    TimeService->>TimeService: Generate Timer Session ID
    TimeService->>Cache: SET active_timer:userId
    activate Cache
    Cache->>Cache: Store {sessionId, startTime, status: "running"}
    Cache-->>TimeService: Timer Started
    deactivate Cache
    
    TimeService-->>API: Success (Timer Data)
    deactivate TimeService
    
    API-->>UI: 200 OK (Timer Session)
    deactivate API
    
    UI->>UI: Start Real-time Counter
    UI-->>User: Display Running Timer
    deactivate UI
    
    Note over UI,User: Time Passes... User Works on Task
    
    User->>UI: Click "Stop Timer"
    activate UI
    
    UI->>API: POST /api/v1/timer/stop
    activate API
    
    API->>Auth: Validate Token
    activate Auth
    Auth-->>API: Valid
    deactivate Auth
    
    API->>Cache: GET active_timer:userId
    activate Cache
    Cache-->>API: Timer Data (startTime)
    deactivate Cache
    
    API->>TimeService: stopTimer(userId, sessionId)
    activate TimeService
    
    TimeService->>TimeService: Calculate Duration
    TimeService->>TimeService: Prepare Entry Draft
    
    TimeService->>Cache: DELETE active_timer:userId
    activate Cache
    Cache-->>TimeService: Timer Cleared
    deactivate Cache
    
    TimeService-->>API: Draft Entry Data
    deactivate TimeService
    
    API-->>UI: 200 OK (Draft with Duration)
    deactivate API
    
    UI->>UI: Show Details Form (Pre-filled)
    UI-->>User: Request Additional Info
    deactivate UI
    
    User->>UI: Enter Description, Project, Submit
    activate UI
    
    UI->>API: POST /api/v1/time-entries (with timer data)
    activate API
    
    API->>TimeService: createTimeEntry(draftData)
    activate TimeService
    
    TimeService->>DB: INSERT time_entry
    activate DB
    DB-->>TimeService: Entry Created
    deactivate DB
    
    TimeService->>Audit: logAction(CREATE)
    activate Audit
    Audit->>DB: INSERT audit_log
    activate DB
    DB-->>Audit: Logged
    deactivate DB
    deactivate Audit
    
    TimeService-->>API: Success
    deactivate TimeService
    
    API-->>UI: 201 Created
    deactivate API
    
    UI-->>User: Display Success
    deactivate UI
```

---

## Alternative Flow: Edit Time Entry

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant TimeService as Time Entry Service
    participant DB as Database
    participant Cache as Cache Service
    participant Audit as Audit Service

    User->>UI: Click Edit on Entry
    activate UI
    
    UI->>API: GET /api/v1/time-entries/{entryId}
    activate API
    
    API->>Auth: Validate Token
    activate Auth
    Auth-->>API: Valid (User ID)
    deactivate Auth
    
    API->>Cache: GET time_entry:{entryId}
    activate Cache
    
    alt Cache Hit
        Cache-->>API: Entry Data (from cache)
    else Cache Miss
        Cache-->>API: Not Found
        API->>DB: SELECT * FROM time_entry WHERE id = entryId
        activate DB
        DB-->>API: Entry Data
        deactivate DB
        API->>Cache: SET time_entry:{entryId}
        Cache-->>API: Cached
    end
    deactivate Cache
    
    API->>TimeService: checkEditPermission(userId, entry)
    activate TimeService
    
    TimeService->>TimeService: Check Ownership
    TimeService->>TimeService: Check Lock Status (7 days)
    
    alt Entry Locked & User Not Manager
        TimeService-->>API: Permission Denied
        API-->>UI: 403 Forbidden
        UI-->>User: Show Error: Entry Locked
    else Permission Granted
        TimeService-->>API: Edit Allowed
        deactivate TimeService
        
        API-->>UI: 200 OK (Entry Data)
        deactivate API
        
        UI->>UI: Display Edit Form
        UI-->>User: Show Editable Form
        deactivate UI
        
        User->>UI: Modify Fields & Save
        activate UI
        
        UI->>API: PUT /api/v1/time-entries/{entryId}
        activate API
        
        API->>Auth: Validate Token
        activate Auth
        Auth-->>API: Valid
        deactivate Auth
        
        API->>TimeService: updateTimeEntry(entryId, newData, userId)
        activate TimeService
        
        TimeService->>DB: SELECT * FROM time_entry WHERE id = entryId FOR UPDATE
        activate DB
        DB-->>TimeService: Original Entry Data
        deactivate DB
        
        TimeService->>TimeService: Validate Changes
        TimeService->>TimeService: Calculate New Duration
        
        TimeService->>DB: BEGIN TRANSACTION
        activate DB
        
        TimeService->>DB: UPDATE time_entry SET ... WHERE id = entryId
        DB-->>TimeService: Updated
        
        TimeService->>DB: COMMIT TRANSACTION
        deactivate DB
        
        TimeService->>Audit: logAction(UPDATE, entryId, userId, oldData, newData)
        activate Audit
        Audit->>DB: INSERT INTO audit_log
        activate DB
        DB-->>Audit: Logged
        deactivate DB
        deactivate Audit
        
        TimeService->>Cache: invalidate time_entry:{entryId}
        activate Cache
        Cache-->>TimeService: Cache Invalidated
        deactivate Cache
        
        TimeService->>Cache: invalidateUserTimeCache(userId)
        activate Cache
        Cache-->>TimeService: Cache Cleared
        deactivate Cache
        
        TimeService-->>API: Update Successful
        deactivate TimeService
        
        API-->>UI: 200 OK (Updated Entry)
        deactivate API
        
        UI->>UI: Refresh Entry Display
        UI-->>User: Show Success Message
        deactivate UI
    end
```

---

## Alternative Flow: Delete Time Entry (Soft Delete)

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant TimeService as Time Entry Service
    participant DB as Database
    participant Cache as Cache Service
    participant Audit as Audit Service
    participant Scheduler as Cleanup Scheduler

    User->>UI: Click Delete on Entry
    activate UI
    
    UI->>UI: Show Confirmation Dialog
    UI-->>User: "Are you sure?"
    deactivate UI
    
    User->>UI: Confirm Deletion
    activate UI
    
    UI->>API: DELETE /api/v1/time-entries/{entryId}
    activate API
    
    API->>Auth: Validate Token
    activate Auth
    Auth-->>API: Valid (User ID)
    deactivate Auth
    
    API->>TimeService: deleteTimeEntry(entryId, userId)
    activate TimeService
    
    TimeService->>DB: SELECT * FROM time_entry WHERE id = entryId
    activate DB
    DB-->>TimeService: Entry Data
    deactivate DB
    
    TimeService->>TimeService: Check Delete Permission
    
    alt Permission Denied
        TimeService-->>API: 403 Forbidden
        API-->>UI: Permission Error
        UI-->>User: Show Error Message
    else Permission Granted
        TimeService->>DB: BEGIN TRANSACTION
        activate DB
        
        TimeService->>DB: UPDATE time_entry SET is_deleted=true, deleted_at=NOW() WHERE id=entryId
        DB-->>TimeService: Entry Soft Deleted
        
        TimeService->>DB: COMMIT TRANSACTION
        deactivate DB
        
        TimeService->>Audit: logAction(DELETE, entryId, userId, oldData)
        activate Audit
        Audit->>DB: INSERT INTO audit_log (action=DELETE)
        activate DB
        DB-->>Audit: Logged
        deactivate DB
        deactivate Audit
        
        TimeService->>Cache: invalidate time_entry:{entryId}
        activate Cache
        Cache-->>TimeService: Invalidated
        deactivate Cache
        
        TimeService->>Cache: invalidateUserTimeCache(userId)
        activate Cache
        Cache-->>TimeService: Cache Cleared
        deactivate Cache
        
        TimeService->>Scheduler: scheduleCleanup(entryId, deleteAfter: 30 days)
        activate Scheduler
        Scheduler-->>TimeService: Scheduled
        deactivate Scheduler
        
        TimeService-->>API: Deletion Successful
        deactivate TimeService
        
        API-->>UI: 200 OK (Success Message)
        deactivate API
        
        UI->>UI: Remove Entry from Display
        UI-->>User: Show Success: "Entry deleted. Can be restored within 30 days."
        deactivate UI
    end
```

---

## Error Handling Sequence

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant TimeService as Time Entry Service
    participant DB as Database
    participant Logger as Error Logger
    participant Monitor as Monitoring Service

    User->>UI: Submit Time Entry
    activate UI
    
    UI->>API: POST /api/v1/time-entries
    activate API
    
    API->>TimeService: createTimeEntry(data)
    activate TimeService
    
    TimeService->>DB: INSERT INTO time_entry
    activate DB
    
    DB-->>TimeService: ERROR: Database Connection Lost
    deactivate DB
    
    TimeService->>TimeService: Catch Exception
    
    TimeService->>Logger: log.error(exception, context)
    activate Logger
    Logger->>Logger: Write to Error Log
    Logger->>Monitor: Send Error Metric
    activate Monitor
    Monitor->>Monitor: Check Error Rate
    
    alt Critical Error Rate
        Monitor->>Monitor: Trigger Alert to Ops Team
        Monitor-->>Logger: Alert Sent
    end
    deactivate Monitor
    deactivate Logger
    
    TimeService-->>API: ERROR: Service Unavailable
    deactivate TimeService
    
    API->>API: Map to HTTP Status
    
    API-->>UI: 503 Service Unavailable {error: "Unable to save entry"}
    deactivate API
    
    UI->>UI: Display User-Friendly Error
    UI-->>User: "Unable to save. Please try again in a moment."
    
    UI->>UI: Enable Retry Button
    deactivate UI
    
    Note over UI,User: User Can Retry or Contact Support
```

---

## Overlap Detection Sequence

```mermaid
sequenceDiagram
    actor User
    participant UI as Web UI
    participant API as API Gateway
    participant TimeService as Time Entry Service
    participant DB as Database

    User->>UI: Submit Time Entry
    activate UI
    
    UI->>API: POST /api/v1/time-entries (9:00 AM - 11:00 AM)
    activate API
    
    API->>TimeService: createTimeEntry(data)
    activate TimeService
    
    TimeService->>DB: Query Overlapping Entries
    activate DB
    Note over DB: SELECT * FROM time_entry<br/>WHERE user_id = ?<br/>AND is_deleted = false<br/>AND ((start_time <= ? AND end_time > ?)<br/>OR (start_time < ? AND end_time >= ?))
    
    DB-->>TimeService: Found Overlapping Entry (10:00 AM - 12:00 PM)
    deactivate DB
    
    TimeService-->>API: Conflict Detected
    deactivate TimeService
    
    API-->>UI: 409 Conflict {overlappingEntries: [...]}
    deactivate API
    
    UI->>UI: Display Warning Dialog
    UI-->>User: "Time overlap detected with existing entry. Continue anyway?"
    deactivate UI
    
    alt User Cancels
        User->>UI: Click Cancel
        activate UI
        UI-->>User: Return to Form
        deactivate UI
    else User Confirms
        User->>UI: Click Continue Anyway
        activate UI
        
        UI->>API: POST /api/v1/time-entries?force=true
        activate API
        
        API->>TimeService: createTimeEntry(data, force=true)
        activate TimeService
        
        TimeService->>TimeService: Skip Overlap Check
        
        TimeService->>DB: INSERT time_entry
        activate DB
        DB-->>TimeService: Entry Created
        deactivate DB
        
        TimeService-->>API: Success
        deactivate TimeService
        
        API-->>UI: 201 Created
        deactivate API
        
        UI-->>User: Success Message
        deactivate UI
    end
```

---

## Sequence Diagram Notation Reference

### Participants
- **Actor (User)**: Human user interacting with the system
- **UI**: Frontend/Client application
- **API Gateway**: API entry point and router
- **Auth Service**: Authentication and authorization
- **Time Entry Service**: Business logic for time management
- **Database**: Data persistence layer
- **Cache**: Redis or similar caching layer
- **Audit Service**: Audit logging service
- **Queue**: Message queue for async processing

### Message Types
- **Solid Arrow (→)**: Synchronous call
- **Dashed Arrow (-->)**: Response/Return
- **Activation Bar**: When component is actively processing

### Timing Characteristics

| Operation | Expected Duration |
|-----------|------------------|
| API Request → Response | 100-500ms |
| Database Query | 10-100ms |
| Cache Operation | 1-10ms |
| Validation | < 50ms |
| Authentication | 20-100ms |
| Audit Logging (async) | N/A (non-blocking) |

---

## Component Interaction Matrix

| Source | Target | Interaction Type | Frequency | Notes |
|--------|--------|-----------------|-----------|-------|
| UI | API Gateway | HTTP REST | High | All user actions |
| API Gateway | Auth Service | gRPC/HTTP | Every request | Token validation |
| API Gateway | Time Service | Internal Call | High | Business logic |
| Time Service | Database | SQL | High | Data persistence |
| Time Service | Cache | Redis Protocol | Very High | Read-through cache |
| Time Service | Audit Service | Async Message | Medium | All mutations |
| Time Service | Queue | AMQP/Kafka | Low | Event publishing |

---

## Security Considerations

1. **Authentication Flow**
   - JWT token validated on every request
   - Token includes user ID and permissions
   - Expired tokens rejected at API Gateway

2. **Authorization Checks**
   - Ownership verification before modifications
   - Role-based permission checks
   - Manager override for locked entries

3. **Data Protection**
   - All communications over HTTPS/TLS
   - Database credentials secured in vault
   - Sensitive data encrypted at rest

---

## Performance Optimization

1. **Caching Strategy**
   - Time entries cached for 5 minutes
   - User cache invalidated on mutations
   - Cache-aside pattern with read-through

2. **Database Optimization**
   - Indexed queries for performance
   - Transaction batching where possible
   - Connection pooling

3. **Async Processing**
   - Audit logging non-blocking
   - Event publishing asynchronous
   - Background jobs for cleanup

---

[← Back to Use Case](./README.md) | [View Pseudocode →](./pseudocode.md)
