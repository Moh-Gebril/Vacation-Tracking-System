# Pseudocode - UC-01: Manage Time

## Overview

This document provides language-agnostic pseudocode implementations for the core functions of the Time Entry Management use case. The pseudocode demonstrates the business logic, validation rules, and data handling procedures.

---

## Table of Contents

1. [Create Time Entry](#1-create-time-entry)
2. [Start Timer](#2-start-timer)
3. [Stop Timer](#3-stop-timer)
4. [Update Time Entry](#4-update-time-entry)
5. [Delete Time Entry (Soft Delete)](#5-delete-time-entry-soft-delete)
6. [Get Time Entries](#6-get-time-entries)
7. [Validation Functions](#7-validation-functions)
8. [Helper Functions](#8-helper-functions)
9. [Scheduled Jobs](#9-scheduled-jobs)

---

## 1. Create Time Entry

```
FUNCTION createTimeEntry(userId, timeEntryData, forceOverlap = false)
    // Input validation
    validationResult = validateTimeEntryData(timeEntryData)
    IF validationResult.hasErrors THEN
        RETURN Error(400, "Validation failed", validationResult.errors)
    END IF
    
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Verify project access if project specified
    IF timeEntryData.projectId IS NOT NULL THEN
        IF NOT hasProjectAccess(userId, timeEntryData.projectId) THEN
            RETURN Error(403, "No access to specified project")
        END IF
        
        // Check if project is archived
        project = getProject(timeEntryData.projectId)
        IF project.isArchived THEN
            RETURN Error(400, "Cannot add entries to archived project")
        END IF
    END IF
    
    // Check for overlapping entries unless forced
    IF NOT forceOverlap THEN
        overlappingEntries = findOverlappingEntries(
            userId, 
            timeEntryData.startTime, 
            timeEntryData.endTime
        )
        
        IF overlappingEntries.length > 0 THEN
            RETURN Conflict(409, "Time overlap detected", overlappingEntries)
        END IF
    END IF
    
    // Calculate duration
    durationMinutes = calculateDuration(
        timeEntryData.startTime, 
        timeEntryData.endTime
    )
    
    // Prepare entry object
    timeEntry = {
        id: generateUUID(),
        userId: userId,
        projectId: timeEntryData.projectId,
        description: timeEntryData.description,
        startTime: timeEntryData.startTime,
        endTime: timeEntryData.endTime,
        durationMinutes: durationMinutes,
        notes: timeEntryData.notes,
        isBillable: timeEntryData.isBillable OR false,
        isDeleted: false,
        deletedAt: null,
        createdAt: getCurrentTimestamp(),
        updatedAt: getCurrentTimestamp(),
        createdBy: userId,
        updatedBy: null
    }
    
    // Begin database transaction
    BEGIN TRANSACTION
    
    TRY
        // Insert time entry
        entryId = database.insert("time_entry", timeEntry)
        
        // Associate tags if provided
        IF timeEntryData.tags IS NOT NULL AND timeEntryData.tags.length > 0 THEN
            FOR EACH tag IN timeEntryData.tags DO
                tagId = ensureTagExists(tag)
                database.insert("time_entry_tag", {
                    id: generateUUID(),
                    timeEntryId: entryId,
                    tagId: tagId,
                    createdAt: getCurrentTimestamp()
                })
            END FOR
        END IF
        
        // Commit transaction
        COMMIT TRANSACTION
        
        // Asynchronous operations (non-blocking)
        ASYNC {
            // Create audit log
            createAuditLog({
                timeEntryId: entryId,
                userId: userId,
                action: "CREATE",
                oldValues: null,
                newValues: timeEntry,
                ipAddress: getClientIpAddress()
            })
            
            // Invalidate cache
            cache.delete("user_time_entries:" + userId)
            
            // Publish event for analytics
            publishEvent("TimeEntryCreated", {
                entryId: entryId,
                userId: userId,
                projectId: timeEntry.projectId,
                duration: durationMinutes
            })
        }
        
        // Return success with created entry
        RETURN Success(201, "Time entry created", timeEntry)
        
    CATCH DatabaseException AS e
        ROLLBACK TRANSACTION
        logError("Failed to create time entry", e, {userId: userId})
        RETURN Error(500, "Unable to save time entry")
    CATCH Exception AS e
        ROLLBACK TRANSACTION
        logError("Unexpected error creating time entry", e, {userId: userId})
        RETURN Error(500, "Internal server error")
    END TRY
    
END FUNCTION
```

---

## 2. Start Timer

```
FUNCTION startTimer(userId)
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Check for existing active timer
    activeTimer = cache.get("active_timer:" + userId)
    IF activeTimer IS NOT NULL THEN
        RETURN Error(400, "Timer already running", activeTimer)
    END IF
    
    // Create timer session
    timerSession = {
        sessionId: generateUUID(),
        userId: userId,
        startTime: getCurrentTimestamp(),
        status: "running"
    }
    
    // Store in cache with 24-hour expiration
    cache.set("active_timer:" + userId, timerSession, TTL=86400)
    
    // Log timer start
    logInfo("Timer started", {userId: userId, sessionId: timerSession.sessionId})
    
    RETURN Success(200, "Timer started", timerSession)
    
END FUNCTION
```

---

## 3. Stop Timer

```
FUNCTION stopTimer(userId, sessionId)
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Retrieve active timer
    activeTimer = cache.get("active_timer:" + userId)
    
    IF activeTimer IS NULL THEN
        RETURN Error(404, "No active timer found")
    END IF
    
    IF activeTimer.sessionId != sessionId THEN
        RETURN Error(400, "Invalid session ID")
    END IF
    
    // Calculate elapsed time
    endTime = getCurrentTimestamp()
    durationMinutes = calculateDuration(activeTimer.startTime, endTime)
    
    // Prepare draft entry
    draftEntry = {
        startTime: activeTimer.startTime,
        endTime: endTime,
        durationMinutes: durationMinutes,
        sessionId: sessionId
    }
    
    // Remove timer from cache
    cache.delete("active_timer:" + userId)
    
    // Log timer stop
    logInfo("Timer stopped", {
        userId: userId, 
        sessionId: sessionId, 
        duration: durationMinutes
    })
    
    // Return draft entry for user to complete
    RETURN Success(200, "Timer stopped", draftEntry)
    
END FUNCTION
```

---

## 4. Update Time Entry

```
FUNCTION updateTimeEntry(userId, entryId, updateData)
    // Input validation
    validationResult = validateTimeEntryUpdate(updateData)
    IF validationResult.hasErrors THEN
        RETURN Error(400, "Validation failed", validationResult.errors)
    END IF
    
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Fetch existing entry
    existingEntry = database.findById("time_entry", entryId)
    
    IF existingEntry IS NULL OR existingEntry.isDeleted THEN
        RETURN Error(404, "Time entry not found")
    END IF
    
    // Check permissions
    canEdit = checkEditPermission(userId, existingEntry)
    IF NOT canEdit.allowed THEN
        RETURN Error(403, canEdit.reason)
    END IF
    
    // Check if project is being changed and validate access
    IF updateData.projectId IS NOT NULL AND 
       updateData.projectId != existingEntry.projectId THEN
        IF NOT hasProjectAccess(userId, updateData.projectId) THEN
            RETURN Error(403, "No access to specified project")
        END IF
    END IF
    
    // Recalculate duration if times changed
    IF updateData.startTime OR updateData.endTime THEN
        startTime = updateData.startTime OR existingEntry.startTime
        endTime = updateData.endTime OR existingEntry.endTime
        updateData.durationMinutes = calculateDuration(startTime, endTime)
    END IF
    
    // Prepare updated entry
    updatedFields = mergeObjects(existingEntry, updateData)
    updatedFields.updatedAt = getCurrentTimestamp()
    updatedFields.updatedBy = userId
    
    // Begin transaction
    BEGIN TRANSACTION
    
    TRY
        // Update time entry
        database.update("time_entry", entryId, updatedFields)
        
        // Update tags if provided
        IF updateData.tags IS NOT NULL THEN
            // Remove old tag associations
            database.delete("time_entry_tag", WHERE timeEntryId = entryId)
            
            // Add new tag associations
            FOR EACH tag IN updateData.tags DO
                tagId = ensureTagExists(tag)
                database.insert("time_entry_tag", {
                    id: generateUUID(),
                    timeEntryId: entryId,
                    tagId: tagId,
                    createdAt: getCurrentTimestamp()
                })
            END FOR
        END IF
        
        COMMIT TRANSACTION
        
        // Asynchronous operations
        ASYNC {
            // Create audit log with old and new values
            createAuditLog({
                timeEntryId: entryId,
                userId: userId,
                action: "UPDATE",
                oldValues: existingEntry,
                newValues: updatedFields,
                ipAddress: getClientIpAddress()
            })
            
            // Invalidate caches
            cache.delete("time_entry:" + entryId)
            cache.delete("user_time_entries:" + userId)
            
            // Publish event
            publishEvent("TimeEntryUpdated", {
                entryId: entryId,
                userId: userId,
                changes: getChangedFields(existingEntry, updatedFields)
            })
        }
        
        RETURN Success(200, "Time entry updated", updatedFields)
        
    CATCH Exception AS e
        ROLLBACK TRANSACTION
        logError("Failed to update time entry", e, {entryId: entryId})
        RETURN Error(500, "Unable to update time entry")
    END TRY
    
END FUNCTION
```

---

## 5. Delete Time Entry (Soft Delete)

```
FUNCTION deleteTimeEntry(userId, entryId)
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Fetch entry
    entry = database.findById("time_entry", entryId)
    
    IF entry IS NULL THEN
        RETURN Error(404, "Time entry not found")
    END IF
    
    IF entry.isDeleted THEN
        RETURN Error(410, "Time entry already deleted")
    END IF
    
    // Check delete permission
    canDelete = checkDeletePermission(userId, entry)
    IF NOT canDelete.allowed THEN
        RETURN Error(403, canDelete.reason)
    END IF
    
    // Soft delete
    TRY
        database.update("time_entry", entryId, {
            isDeleted: true,
            deletedAt: getCurrentTimestamp(),
            updatedAt: getCurrentTimestamp(),
            updatedBy: userId
        })
        
        // Asynchronous operations
        ASYNC {
            // Create audit log
            createAuditLog({
                timeEntryId: entryId,
                userId: userId,
                action: "DELETE",
                oldValues: entry,
                newValues: null,
                ipAddress: getClientIpAddress()
            })
            
            // Invalidate cache
            cache.delete("time_entry:" + entryId)
            cache.delete("user_time_entries:" + userId)
            
            // Schedule permanent deletion in 30 days
            scheduleJob("permanentDelete", {
                entryId: entryId,
                scheduledFor: addDays(getCurrentTimestamp(), 30)
            })
            
            // Publish event
            publishEvent("TimeEntryDeleted", {
                entryId: entryId,
                userId: userId
            })
        }
        
        RETURN Success(200, "Time entry deleted. Can be restored within 30 days.")
        
    CATCH Exception AS e
        logError("Failed to delete time entry", e, {entryId: entryId})
        RETURN Error(500, "Unable to delete time entry")
    END TRY
    
END FUNCTION
```

---

## 6. Get Time Entries

```
FUNCTION getTimeEntries(userId, filters, pagination)
    // Check authentication
    IF NOT isAuthenticated(userId) THEN
        RETURN Error(401, "Unauthorized")
    END IF
    
    // Build cache key
    cacheKey = buildCacheKey("user_time_entries", userId, filters, pagination)
    
    // Try cache first
    cachedResult = cache.get(cacheKey)
    IF cachedResult IS NOT NULL THEN
        RETURN Success(200, "Time entries retrieved (cached)", cachedResult)
    END IF
    
    // Build query
    query = {
        userId: userId,
        isDeleted: false
    }
    
    // Apply filters
    IF filters.projectId IS NOT NULL THEN
        query.projectId = filters.projectId
    END IF
    
    IF filters.startDate IS NOT NULL THEN
        query.startTime >= filters.startDate
    END IF
    
    IF filters.endDate IS NOT NULL THEN
        query.startTime < filters.endDate
    END IF
    
    IF filters.isBillable IS NOT NULL THEN
        query.isBillable = filters.isBillable
    END IF
    
    IF filters.tags IS NOT NULL AND filters.tags.length > 0 THEN
        query.JOIN_time_entry_tag.tagId IN filters.tags
    END IF
    
    // Apply pagination
    offset = (pagination.page - 1) * pagination.pageSize
    limit = pagination.pageSize
    
    // Execute query
    TRY
        entries = database.query("time_entry", {
            where: query,
            orderBy: "startTime DESC",
            offset: offset,
            limit: limit,
            include: ["project", "tags"]
        })
        
        // Get total count
        totalCount = database.count("time_entry", where: query)
        
        // Calculate summary statistics
        summary = {
            totalEntries: totalCount,
            totalDuration: database.sum("time_entry", "durationMinutes", where: query),
            totalPages: ceiling(totalCount / pagination.pageSize),
            currentPage: pagination.page
        }
        
        result = {
            entries: entries,
            summary: summary
        }
        
        // Cache result for 5 minutes
        cache.set(cacheKey, result, TTL=300)
        
        RETURN Success(200, "Time entries retrieved", result)
        
    CATCH Exception AS e
        logError("Failed to retrieve time entries", e, {userId: userId})
        RETURN Error(500, "Unable to retrieve time entries")
    END TRY
    
END FUNCTION
```

---

## 7. Validation Functions

### 7.1 Validate Time Entry Data

```
FUNCTION validateTimeEntryData(data)
    errors = []
    
    // Required fields
    IF data.description IS NULL OR isEmpty(data.description) THEN
        errors.add({field: "description", message: "Description is required"})
    END IF
    
    IF data.description.length > 500 THEN
        errors.add({field: "description", message: "Description must not exceed 500 characters"})
    END IF
    
    IF data.startTime IS NULL THEN
        errors.add({field: "startTime", message: "Start time is required"})
    END IF
    
    // Validate time format
    IF NOT isValidTimestamp(data.startTime) THEN
        errors.add({field: "startTime", message: "Invalid start time format"})
    END IF
    
    IF data.endTime IS NOT NULL THEN
        IF NOT isValidTimestamp(data.endTime) THEN
            errors.add({field: "endTime", message: "Invalid end time format"})
        END IF
        
        // End time must be after start time
        IF data.endTime <= data.startTime THEN
            errors.add({field: "endTime", message: "End time must be after start time"})
        END IF
        
        // Duration must not exceed 24 hours
        duration = calculateDuration(data.startTime, data.endTime)
        IF duration > 1440 THEN
            errors.add({field: "duration", message: "Duration cannot exceed 24 hours"})
        END IF
    END IF
    
    // Start time cannot be in the future
    IF data.startTime > getCurrentTimestamp() THEN
        errors.add({field: "startTime", message: "Cannot log time in the future"})
    END IF
    
    // Validate project ID format if provided
    IF data.projectId IS NOT NULL AND NOT isValidUUID(data.projectId) THEN
        errors.add({field: "projectId", message: "Invalid project ID format"})
    END IF
    
    RETURN {
        hasErrors: errors.length > 0,
        errors: errors
    }
    
END FUNCTION
```

### 7.2 Check Edit Permission

```
FUNCTION checkEditPermission(userId, timeEntry)
    // User owns the entry
    IF timeEntry.userId != userId THEN
        userRole = getUserRole(userId)
        
        // Only managers and admins can edit others' entries
        IF userRole NOT IN ["manager", "admin"] THEN
            RETURN {
                allowed: false,
                reason: "You can only edit your own time entries"
            }
        END IF
    END IF
    
    // Check if entry is locked (older than 7 days)
    daysSinceCreation = daysBetween(timeEntry.createdAt, getCurrentTimestamp())
    
    IF daysSinceCreation > 7 THEN
        userRole = getUserRole(userId)
        
        // Only managers and admins can edit locked entries
        IF userRole NOT IN ["manager", "admin"] THEN
            RETURN {
                allowed: false,
                reason: "Entry is locked. Entries older than 7 days require manager approval."
            }
        END IF
    END IF
    
    RETURN {allowed: true}
    
END FUNCTION
```

### 7.3 Find Overlapping Entries

```
FUNCTION findOverlappingEntries(userId, startTime, endTime)
    // Query for overlapping entries
    // An entry overlaps if:
    // 1. It starts before the new entry ends AND ends after the new entry starts
    // 2. It starts within the new entry's time range
    // 3. It completely contains the new entry's time range
    
    query = "
        SELECT * FROM time_entry
        WHERE user_id = ?
          AND is_deleted = false
          AND (
              (start_time < ? AND end_time > ?) OR
              (start_time >= ? AND start_time < ?) OR
              (start_time <= ? AND end_time >= ?)
          )
    "
    
    overlapping = database.executeQuery(query, [
        userId,
        endTime, startTime,
        startTime, endTime,
        startTime, endTime
    ])
    
    RETURN overlapping
    
END FUNCTION
```

---

## 8. Helper Functions

### 8.1 Calculate Duration

```
FUNCTION calculateDuration(startTime, endTime)
    // Calculate difference in minutes
    IF endTime IS NULL THEN
        RETURN null
    END IF
    
    milliseconds = endTime.getTime() - startTime.getTime()
    minutes = floor(milliseconds / 60000)
    
    RETURN minutes
    
END FUNCTION
```

### 8.2 Ensure Tag Exists

```
FUNCTION ensureTagExists(tagName)
    // Normalize tag name
    normalizedName = trim(toLowerCase(tagName))
    
    // Check if tag already exists
    existingTag = database.findOne("tag", {name: normalizedName})
    
    IF existingTag IS NOT NULL THEN
        RETURN existingTag.id
    END IF
    
    // Create new tag
    newTag = {
        id: generateUUID(),
        name: normalizedName,
        color: generateRandomColor(),
        createdAt: getCurrentTimestamp()
    }
    
    tagId = database.insert("tag", newTag)
    
    RETURN tagId
    
END FUNCTION
```

### 8.3 Create Audit Log

```
FUNCTION createAuditLog(auditData)
    auditEntry = {
        id: generateUUID(),
        timeEntryId: auditData.timeEntryId,
        userId: auditData.userId,
        action: auditData.action,
        oldValues: toJSON(auditData.oldValues),
        newValues: toJSON(auditData.newValues),
        createdAt: getCurrentTimestamp(),
        ipAddress: auditData.ipAddress
    }
    
    TRY
        database.insert("audit_log", auditEntry)
    CATCH Exception AS e
        // Log error but don't fail the main operation
        logError("Failed to create audit log", e, auditData)
    END TRY
    
END FUNCTION
```

### 8.4 Build Cache Key

```
FUNCTION buildCacheKey(prefix, userId, filters, pagination)
    components = [prefix, userId]
    
    IF filters IS NOT NULL THEN
        FOR EACH key, value IN filters DO
            components.add(key + ":" + toString(value))
        END FOR
    END IF
    
    IF pagination IS NOT NULL THEN
        components.add("page:" + toString(pagination.page))
        components.add("size:" + toString(pagination.pageSize))
    END IF
    
    cacheKey = join(components, ":")
    
    RETURN cacheKey
    
END FUNCTION
```

---

## 9. Scheduled Jobs

### 9.1 Permanent Delete Job

```
FUNCTION permanentDeleteOldEntries()
    // Run daily
    // Delete entries that were soft-deleted more than 30 days ago
    
    cutoffDate = subtractDays(getCurrentTimestamp(), 30)
    
    query = "
        SELECT id FROM time_entry
        WHERE is_deleted = true
          AND deleted_at < ?
    "
    
    entriesToDelete = database.executeQuery(query, [cutoffDate])
    
    deletedCount = 0
    
    FOR EACH entry IN entriesToDelete DO
        TRY
            // Delete associated records
            database.delete("time_entry_tag", WHERE timeEntryId = entry.id)
            
            // Delete the entry
            database.delete("time_entry", WHERE id = entry.id)
            
            deletedCount = deletedCount + 1
            
        CATCH Exception AS e
            logError("Failed to permanently delete entry", e, {entryId: entry.id})
        END TRY
    END FOR
    
    logInfo("Permanent delete job completed", {deletedCount: deletedCount})
    
END FUNCTION
```

### 9.2 Cache Warming Job

```
FUNCTION warmCache()
    // Run every hour
    // Pre-load frequently accessed data into cache
    
    // Get active users (users who logged in within last 24 hours)
    activeUsers = getActiveUsers()
    
    FOR EACH user IN activeUsers DO
        TRY
            // Cache today's time entries
            todayStart = startOfDay(getCurrentTimestamp())
            todayEnd = endOfDay(getCurrentTimestamp())
            
            entries = getTimeEntries(user.id, {
                startDate: todayStart,
                endDate: todayEnd
            }, {page: 1, pageSize: 50})
            
        CATCH Exception AS e
            logError("Failed to warm cache for user", e, {userId: user.id})
        END TRY
    END FOR
    
END FUNCTION
```

### 9.3 Send Daily Reminder

```
FUNCTION sendDailyReminders()
    // Run daily at user's preferred time
    
    // Get users with reminders enabled
    usersWithReminders = database.query("user_preference", {
        preferenceKey: "reminder_enabled",
        preferenceValue: "true"
    })
    
    FOR EACH userPref IN usersWithReminders DO
        TRY
            userId = userPref.userId
            
            // Check if user logged time today
            today = startOfDay(getCurrentTimestamp())
            
            entryCount = database.count("time_entry", {
                userId: userId,
                startTime >= today,
                isDeleted: false
            })
            
            IF entryCount == 0 THEN
                // Send reminder notification
                sendNotification(userId, {
                    type: "reminder",
                    title: "Don't forget to log your time!",
                    message: "You haven't logged any time today.",
                    action: "LOG_TIME"
                })
            END IF
            
        CATCH Exception AS e
            logError("Failed to send reminder", e, {userId: userPref.userId})
        END TRY
    END FOR
    
END FUNCTION
```

---

## Error Codes Reference

| Code | Description | User Action |
|------|-------------|-------------|
| 400 | Bad Request - Validation Error | Fix input data |
| 401 | Unauthorized - Not authenticated | Log in |
| 403 | Forbidden - No permission | Request access |
| 404 | Not Found - Entry doesn't exist | Check entry ID |
| 409 | Conflict - Time overlap | Confirm or adjust times |
| 410 | Gone - Entry already deleted | Cannot recover |
| 500 | Internal Server Error | Retry or contact support |
| 503 | Service Unavailable | Wait and retry |

---

## Performance Considerations

### Time Complexity
- `createTimeEntry`: O(1) for insert + O(n) for tags where n = number of tags
- `getTimeEntries`: O(log n) with indexes + O(m) for m results
- `findOverlappingEntries`: O(log n) with proper indexes
- `deleteTimeEntry`: O(1) for soft delete

### Space Complexity
- Cache storage: O(k) where k = number of active users
- Audit logs: O(n) where n = total operations (with retention policy)

### Optimization Strategies
1. Use database indexes on frequently queried columns
2. Implement caching for read-heavy operations
3. Use connection pooling for database access
4. Batch operations where possible
5. Implement pagination for large result sets

---

[← Back to Use Case](./README.md) | [← View Sequence Diagram](./sequence-diagram.md)
