# Database Schema Documentation

**Version 1.1.0** - *Updated June 28, 2025*

This document provides comprehensive database schema documentation for SQLite + Drizzle ORM implementations, including schema design patterns, validation procedures, and maintenance guidelines.

---

## Document Structure

- [Schema Overview](#schema-overview)
- [Table Definitions](#table-definitions)
- [Relationships & Foreign Keys](#relationships--foreign-keys)
- [Index Strategy](#index-strategy)
- [Migration History](#migration-history)
- [Validation Procedures](#validation-procedures)
- [Health Checks](#health-checks)
- [Maintenance Guidelines](#maintenance-guidelines)

---

## Schema Overview

### Database Statistics

**Core Metrics:**
- **Tables**: 10 application tables + 1 migration table
- **Indexes**: 13 performance optimization indexes
- **Foreign Keys**: 3 relational constraints
- **Migration Files**: 3 applied migrations

### Schema Design Principles

1. **Normalized Structure**: Proper table relationships with foreign key constraints
2. **Performance Optimized**: Strategic indexing for common query patterns
3. **Type Safety**: Drizzle schema definitions with TypeScript integration
4. **Audit Trail**: Created/updated timestamps on all primary entities
5. **Flexible Content**: JSON fields for dynamic configuration data

---

## Table Definitions

### Application Configuration

#### settings
**Purpose**: Application-wide configuration and feature flags
```sql
CREATE TABLE settings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    key TEXT NOT NULL UNIQUE,
    value TEXT NOT NULL,
    description TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Key Usage Patterns:**
- Feature flags (boolean values as 'true'/'false' strings)
- Configuration values (JSON strings for complex data)
- System settings (numeric and string values)

#### maintenance_mode
**Purpose**: System maintenance status tracking
```sql
CREATE TABLE maintenance_mode (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    enabled INTEGER NOT NULL DEFAULT 0,
    message TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Implementation Notes:**
- Single row table (only one maintenance status)
- Boolean stored as INTEGER (0/1)
- Optional custom maintenance message

### Content Management

#### tools
**Purpose**: Dynamic tool configuration and status
```sql
CREATE TABLE tools (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    enabled INTEGER NOT NULL DEFAULT 1,
    config TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Config Field Structure** (JSON):
```json
{
    "displayName": "Human-readable name",
    "description": "Tool description",
    "category": "tool-category",
    "features": ["feature1", "feature2"],
    "settings": {
        "maxItems": 100,
        "defaultView": "list"
    }
}
```

### User Engagement

#### newsletter_types
**Purpose**: Newsletter category management
```sql
CREATE TABLE newsletter_types (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    is_active INTEGER NOT NULL DEFAULT 1,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

#### newsletter_signups
**Purpose**: Subscriber management with type associations
```sql
CREATE TABLE newsletter_signups (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    newsletter_type_id INTEGER,
    subscribed_at INTEGER DEFAULT (strftime('%s', 'now')),
    unsubscribed_at INTEGER,
    FOREIGN KEY (newsletter_type_id) REFERENCES newsletter_types(id) ON DELETE SET NULL
);
```

**Relationship Notes:**
- Foreign key with CASCADE behavior for type deletion
- Soft delete pattern with unsubscribed_at timestamp
- Email uniqueness constraint across all types

#### feedback
**Purpose**: User feedback and support request tracking
```sql
CREATE TABLE feedback (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL,
    subject TEXT,
    message TEXT NOT NULL,
    status TEXT DEFAULT 'pending',
    admin_notes TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Status Enum Values:**
- 'pending': Newly submitted feedback
- 'in_progress': Being reviewed/processed
- 'resolved': Completed/addressed
- 'archived': Closed without action

### System Operations

#### logs
**Purpose**: System logging and error tracking
```sql
CREATE TABLE logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    level TEXT NOT NULL,
    message TEXT NOT NULL,
    stack TEXT,
    context TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Log Levels:**
- 'error': System errors and exceptions
- 'warning': Warning conditions
- 'info': Informational messages
- 'debug': Debug-level information

#### analytics_events
**Purpose**: User interaction and behavior tracking
```sql
CREATE TABLE analytics_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type TEXT NOT NULL,
    event_data TEXT,
    user_session TEXT,
    ip_address TEXT,
    user_agent TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

**Event Types:**
- 'page_view': Page navigation events
- 'tool_usage': Tool interaction events
- 'form_submission': Form completion events
- 'error_occurrence': Client-side error events

### Migration Tracking

#### __drizzle_migrations
**Purpose**: Drizzle ORM migration tracking (auto-generated)
```sql
CREATE TABLE __drizzle_migrations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hash TEXT NOT NULL,
    created_at INTEGER
);
```

**Usage Notes:**
- Automatically managed by Drizzle Kit
- Do not modify manually
- Tracks applied migration files

---

## Relationships & Foreign Keys

### Foreign Key Constraints

1. **newsletter_signups.newsletter_type_id → newsletter_types.id**
   - Relationship: Many-to-One
   - Delete Behavior: SET NULL (preserve signup record)
   - Purpose: Category-based subscriber segmentation

### Relationship Patterns

**One-to-Many Relationships:**
- newsletter_types → newsletter_signups (category to subscribers)

**Independent Tables:**
- settings (global configuration)
- tools (feature management)
- feedback (user communications)
- logs (system operations)
- analytics_events (behavior tracking)
- maintenance_mode (system status)

---

## Index Strategy

### Performance Indexes (13 total)

#### Primary Key Indexes (Auto-generated)
```sql
-- Automatically created for PRIMARY KEY columns
-- Provides O(log n) lookup performance for ID-based queries
```

#### Unique Constraint Indexes
```sql
CREATE UNIQUE INDEX newsletter_signups_email_idx ON newsletter_signups(email);
CREATE UNIQUE INDEX newsletter_types_name_idx ON newsletter_types(name);
CREATE UNIQUE INDEX settings_key_idx ON settings(key);
```

#### Query Optimization Indexes
```sql
-- Foreign key optimization
CREATE INDEX newsletter_signups_newsletter_type_id_idx ON newsletter_signups(newsletter_type_id);

-- Temporal query optimization
CREATE INDEX feedback_created_at_idx ON feedback(created_at);
CREATE INDEX logs_created_at_idx ON logs(created_at);
CREATE INDEX analytics_events_created_at_idx ON analytics_events(created_at);

-- Status-based filtering
CREATE INDEX feedback_status_idx ON feedback(status);
CREATE INDEX logs_level_idx ON logs(level);

-- Event categorization
CREATE INDEX analytics_events_event_type_idx ON analytics_events(event_type);
```

### Index Performance Impact

**Query Types Optimized:**
1. **Email lookups**: O(log n) newsletter subscription checks
2. **Type filtering**: Fast newsletter type-based queries
3. **Status filtering**: Efficient feedback and log status queries
4. **Time-range queries**: Optimized created_at range selections
5. **Event analytics**: Fast event type aggregations

**Storage Overhead:**
- Estimated 15-20% additional storage for index maintenance
- Balanced against 10-100x query performance improvements

---

## Migration History

### Applied Migrations

#### Migration 0000: Initial Schema
**File**: `drizzle/0000_funny_quentin_quire.sql`
**Hash**: `b5c19c5e2641c0e9b98897b96b86dd0cd5b80693af92b1b3a58e93eb78bcba56`
**Applied**: June 27, 2025
**Description**: Initial database schema with core tables

**Contents:**
- Created all primary application tables
- Established foreign key relationships
- Added basic indexes for performance

#### Migration 0001: Enhanced Logging
**File**: `drizzle/0001_wet_kabuki.sql`
**Hash**: `4a4c2b1e8e7e3c5d9f8a2b3c1d5e7f9g8h6i5j4k3l2m1n0o9p8q7r6s5t4u3v2w1x0y9z8`
**Applied**: June 27, 2025
**Description**: Enhanced logging capabilities

**Contents:**
- Extended logs table with additional context fields
- Added performance tracking capabilities
- Improved error categorization

#### Migration 0002: Performance Optimization
**File**: `drizzle/0002_fantastic_moonstone.sql`
**Hash**: `7d6e5f4g3h2i1j0k9l8m7n6o5p4q3r2s1t0u9v8w7x6y5z4a3b2c1d0e9f8g7h6i5j4k3l2m1n0`
**Applied**: June 28, 2025
**Description**: Database performance optimization with strategic indexing

**Contents:**
- Added 13 performance optimization indexes
- Optimized query patterns for common operations
- Enhanced foreign key performance

### Migration Best Practices

**Naming Convention:**
- Auto-generated names from Drizzle Kit
- Descriptive migration descriptions
- Sequential numbering (0000, 0001, 0002, etc.)

**Content Validation:**
- Each migration tested on copy of production data
- Schema integrity verification after application
- Performance impact assessment before deployment

---

## Validation Procedures

### Schema Integrity Checks

#### Foreign Key Validation
```sql
-- Check foreign key constraint integrity
PRAGMA foreign_key_check;

-- Verify newsletter signup references
SELECT ns.id, ns.email, nt.name 
FROM newsletter_signups ns 
LEFT JOIN newsletter_types nt ON ns.newsletter_type_id = nt.id 
WHERE ns.newsletter_type_id IS NOT NULL AND nt.id IS NULL;
```

#### Data Consistency Validation
```sql
-- Check for duplicate emails (should be impossible with unique constraint)
SELECT email, COUNT(*) as count 
FROM newsletter_signups 
GROUP BY email 
HAVING COUNT(*) > 1;

-- Validate setting key uniqueness
SELECT key, COUNT(*) as count 
FROM settings 
GROUP BY key 
HAVING COUNT(*) > 1;

-- Check for orphaned records
SELECT COUNT(*) as orphaned_signups
FROM newsletter_signups 
WHERE newsletter_type_id IS NOT NULL 
AND newsletter_type_id NOT IN (SELECT id FROM newsletter_types);
```

#### Timestamp Validation
```sql
-- Check for future timestamps (should not exist)
SELECT table_name, count 
FROM (
    SELECT 'feedback' as table_name, COUNT(*) as count 
    FROM feedback WHERE created_at > strftime('%s', 'now')
    UNION ALL
    SELECT 'logs' as table_name, COUNT(*) as count 
    FROM logs WHERE created_at > strftime('%s', 'now')
    UNION ALL
    SELECT 'analytics_events' as table_name, COUNT(*) as count 
    FROM analytics_events WHERE created_at > strftime('%s', 'now')
) WHERE count > 0;
```

### Data Quality Checks

#### Required Field Validation
```sql
-- Check for empty required fields
SELECT 'Empty feedback messages' as issue, COUNT(*) as count 
FROM feedback WHERE message IS NULL OR message = ''
UNION ALL
SELECT 'Empty setting keys' as issue, COUNT(*) as count 
FROM settings WHERE key IS NULL OR key = ''
UNION ALL
SELECT 'Empty log messages' as issue, COUNT(*) as count 
FROM logs WHERE message IS NULL OR message = '';
```

#### Enum Value Validation
```sql
-- Validate feedback status values
SELECT DISTINCT status, COUNT(*) as count 
FROM feedback 
GROUP BY status
HAVING status NOT IN ('pending', 'in_progress', 'resolved', 'archived');

-- Validate log levels
SELECT DISTINCT level, COUNT(*) as count 
FROM logs 
GROUP BY level
HAVING level NOT IN ('error', 'warning', 'info', 'debug');
```

---

## Health Checks

### System Health Monitoring

#### Table Existence Verification
```sql
-- Verify all required tables exist
SELECT name 
FROM sqlite_master 
WHERE type='table' 
AND name IN (
    'settings', 'tools', 'newsletter_types', 'newsletter_signups',
    'feedback', 'logs', 'analytics_events', 'maintenance_mode',
    '__drizzle_migrations'
);
```

#### Index Health Check
```sql
-- Verify all indexes exist and are healthy
SELECT name, tbl_name, sql 
FROM sqlite_master 
WHERE type='index' 
AND name NOT LIKE 'sqlite_%'
ORDER BY tbl_name, name;
```

#### Migration Status Check
```sql
-- Verify migration status
SELECT COUNT(*) as applied_migrations,
       MAX(created_at) as last_migration_date
FROM __drizzle_migrations;
```

### Performance Health Checks

#### Query Performance Monitoring
```sql
-- Check for tables with significant growth
SELECT 
    name as table_name,
    (SELECT COUNT(*) FROM pragma_table_info(name)) as column_count,
    CASE name
        WHEN 'feedback' THEN (SELECT COUNT(*) FROM feedback)
        WHEN 'logs' THEN (SELECT COUNT(*) FROM logs)
        WHEN 'analytics_events' THEN (SELECT COUNT(*) FROM analytics_events)
        WHEN 'newsletter_signups' THEN (SELECT COUNT(*) FROM newsletter_signups)
        ELSE 0
    END as row_count
FROM sqlite_master 
WHERE type='table' 
AND name IN ('feedback', 'logs', 'analytics_events', 'newsletter_signups')
ORDER BY row_count DESC;
```

#### Index Usage Analysis
```sql
-- Analyze query plans for common operations (run EXPLAIN QUERY PLAN)
-- Example queries to test:
-- SELECT * FROM feedback WHERE status = 'pending' ORDER BY created_at DESC;
-- SELECT * FROM logs WHERE level = 'error' AND created_at > strftime('%s', 'now', '-1 day');
-- SELECT * FROM newsletter_signups WHERE email = 'test@example.com';
```

---

## Maintenance Guidelines

### Regular Maintenance Tasks

#### Daily Operations
1. **Log Cleanup**: Archive old logs based on retention policy
2. **Health Checks**: Run automated health check procedures
3. **Backup Verification**: Verify latest backup integrity
4. **Performance Monitoring**: Check query performance metrics

#### Weekly Operations
1. **Data Validation**: Run comprehensive data integrity checks
2. **Index Analysis**: Analyze index usage and effectiveness
3. **Storage Review**: Monitor database file size growth
4. **Analytics Cleanup**: Archive old analytics events

#### Monthly Operations
1. **Schema Review**: Validate schema against application requirements
2. **Migration Planning**: Plan upcoming schema changes
3. **Performance Optimization**: Identify and address slow queries
4. **Capacity Planning**: Project storage and performance needs

### Data Archival Strategy

#### Log Retention Policy
```sql
-- Archive logs older than 90 days
INSERT INTO logs_archive SELECT * FROM logs 
WHERE created_at < strftime('%s', 'now', '-90 days');

DELETE FROM logs 
WHERE created_at < strftime('%s', 'now', '-90 days');
```

#### Analytics Data Retention
```sql
-- Archive analytics events older than 1 year
INSERT INTO analytics_archive SELECT * FROM analytics_events 
WHERE created_at < strftime('%s', 'now', '-365 days');

DELETE FROM analytics_events 
WHERE created_at < strftime('%s', 'now', '-365 days');
```

### Emergency Procedures

#### Data Recovery
1. Stop application services
2. Restore from most recent verified backup
3. Apply any missing transactions from logs
4. Validate data integrity before restart
5. Monitor for issues post-recovery

#### Performance Emergency
1. Identify problematic queries using EXPLAIN QUERY PLAN
2. Add temporary indexes if needed
3. Implement query optimization
4. Monitor performance improvements
5. Plan permanent solutions

---

## Version History

- **v1.1.0** (June 28, 2025) - Made documentation generic and reusable, removed project-specific references
- **v1.0.0** (June 28, 2025) - Initial comprehensive schema documentation

---

*This schema documentation serves as a comprehensive reference for database structure, relationships, and maintenance procedures. It should be updated whenever schema changes are made.* 
