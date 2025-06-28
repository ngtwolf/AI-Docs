# SQLite + Drizzle ORM: Complete Implementation Guide

**v3.0.0 - Final Production Guide**

*A production-tested, comprehensive guide for implementing SQLite with Drizzle ORM in any project. Based on real-world experience and focused on essential functionality.*

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Command Reference Quick Card](#command-reference-quick-card)
3. [Project Structure](#project-structure)
4. [Core Configuration](#core-configuration)
5. [Database Schema Design](#database-schema-design)
6. [Migration System](#migration-system)
7. [Environment & Path Management](#environment--path-management)
8. [Docker Integration](#docker-integration)
9. [Backup & Restore System](#backup--restore-system)
10. [Critical Pitfalls & Solutions](#critical-pitfalls--solutions)
11. [Testing Checklist](#testing-checklist)
12. [Troubleshooting Guide](#troubleshooting-guide)
13. [Security & Production Considerations](#security--production-considerations)

---

## Quick Start

### Installation

```bash
npm install @libsql/client drizzle-orm drizzle-kit dotenv
npm install -D @types/better-sqlite3
```

### Essential Dependencies

```json
{
  "dependencies": {
    "@libsql/client": "^0.15.9",
    "drizzle-orm": "^0.44.2",
    "drizzle-kit": "^0.31.2",
    "dotenv": "^16.5.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.13"
  }
}
```

### NPM Scripts

```json
{
  "scripts": {
    "db:migrate": "node src/lib/migrate.js",
    "db:generate": "drizzle-kit generate",
    "db:studio": "drizzle-kit studio"
  }
}
```

---

## Command Reference Quick Card

### Development Workflow
```bash
# Schema changes workflow
npm run db:generate     # Generate migration after schema changes
npm run db:migrate      # Apply migrations safely
npm run db:studio       # Open database GUI

# Development testing
node validate-paths.js  # Verify path consistency
npm run build          # Test build after changes
```

### Production Deployment
```bash
# Pre-deployment validation
node validate-paths.js                    # Critical: verify paths
docker-compose config                     # Validate Docker config
docker-compose up --build -d             # Deploy with rebuild

# Post-deployment verification
docker logs container-name               # Check migration logs
docker exec -it container-name ls -la /app/data  # Verify data directory
```

### Backup Operations
```bash
# Manual backup creation
curl -X POST http://localhost:3000/api/admin/backups/create

# Backup management
ls -la data/backups/json/               # List local backups
docker exec container-name ls -la /app/data/backups/  # List container backups
```

### Troubleshooting Commands
```bash
# Database diagnostics
sqlite3 data/database.db ".tables"      # List tables
sqlite3 data/database.db ".schema"      # Show schema
sqlite3 data/database.db "PRAGMA integrity_check;"  # Check integrity

# Docker diagnostics
docker exec -it container-name sqlite3 /app/data/database.db ".tables"
docker inspect container-name | grep -A 10 Mounts  # Check volume mounts
```

---

## Project Structure

```
project-root/
├── src/
│   └── lib/
│       ├── db/
│       │   ├── index.ts          # Database connection
│       │   └── schema.ts         # Database schema
│       └── migrate.js           # Safe migration runner
├── drizzle/                     # Generated migration files
│   ├── 0000_initial.sql
│   ├── 0001_add_features.sql
│   └── meta/
│       ├── _journal.json
│       └── snapshots...
├── data/                        # Database storage
│   ├── database.db
│   └── backups/
├── drizzle.config.ts           # Development config
├── docker-compose.yml          # Container setup
├── Dockerfile
├── production.env
└── validate-paths.js
```

---

## Core Configuration

### 1. Development Configuration (`drizzle.config.ts`)

```typescript
import type { Config } from 'drizzle-kit';

export default {
    schema: './src/lib/db/schema.ts',
    out: './drizzle',
    dialect: 'sqlite',
    dbCredentials: {
        url: './data/database.db'
    },
    verbose: true,
    strict: true,
} satisfies Config;
```

### 2. Environment Variables (`production.env`)

```bash
# Database Configuration
DATABASE_URL=file:/app/data/database.db

# Application Configuration  
NODE_ENV=production
SESSION_SECRET=your_secure_jwt_secret_here
NEXT_PUBLIC_BASE_URL=https://your-domain.com
```

### 3. Database Connection (`src/lib/db/index.ts`)

**⚠️ CRITICAL: NO automatic migrations on import**

```typescript
import 'dotenv/config';
import { createClient } from '@libsql/client';
import { drizzle } from 'drizzle-orm/libsql';
import * as schema from './schema';

const client = createClient({
    url: process.env.DATABASE_URL || 'file:data/database.db',
    authToken: process.env.DATABASE_AUTH_TOKEN,
});

export const db = drizzle(client, { schema });

// Development-only initialization (optional)
export async function initializeDatabase() {
    try {
        console.log("Database initialization requested...");
        
        const { migrate } = await import('drizzle-orm/libsql/migrator');
        
        console.log("Running database migrations...");
        await migrate(db, { migrationsFolder: './drizzle' });
        console.log("Migrations completed.");
        
        // Add your application's seeding logic here if needed
        
        console.log("Database initialization complete.");
        
    } catch (error) {
        console.error("Error during database initialization:", error);
        throw error;
    }
}

// Note: Use separate migrate.js script for production deployments
```

---

## Database Schema Design

### Essential Schema Patterns

```typescript
// src/lib/db/schema.ts
import { sqliteTable, text, integer, index } from 'drizzle-orm/sqlite-core';
import { relations, sql } from 'drizzle-orm';

// Basic table with auto-increment ID and timestamps
export const users = sqliteTable('users', {
    id: integer('id').primaryKey({ autoIncrement: true }),
    email: text('email').notNull().unique(),
    name: text('name').notNull(),
    status: text('status', { enum: ['active', 'inactive'] }).notNull().default('active'),
    createdAt: integer('created_at', { mode: 'timestamp' }).notNull().default(sql`(strftime('%s', 'now'))`),
    updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull().default(sql`(strftime('%s', 'now'))`),
}, (table) => ({
    emailIdx: index('users_email_idx').on(table.email),
    statusIdx: index('users_status_idx').on(table.status),
}));

// Related table with foreign key
export const posts = sqliteTable('posts', {
    id: integer('id').primaryKey({ autoIncrement: true }),
    title: text('title').notNull(),
    content: text('content').notNull(),
    userId: integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
    status: text('status', { enum: ['draft', 'published'] }).notNull().default('draft'),
    createdAt: integer('created_at', { mode: 'timestamp' }).notNull().default(sql`(strftime('%s', 'now'))`),
}, (table) => ({
    userIdx: index('posts_user_id_idx').on(table.userId),
    statusIdx: index('posts_status_idx').on(table.status),
}));

// Relations for type safety
export const userRelations = relations(users, ({ many }) => ({
    posts: many(posts),
}));

export const postRelations = relations(posts, ({ one }) => ({
    author: one(users, {
        fields: [posts.userId],
        references: [users.id],
    }),
}));
```

### Design Principles

1. **Use `integer` for IDs** - More efficient than text UUIDs
2. **Always use auto-increment** for primary keys
3. **Use Unix timestamps** - `integer('created_at', { mode: 'timestamp' })`
4. **Index foreign keys** - Essential for performance
5. **Use enums for status fields** - Ensures data consistency
6. **Set `onDelete` behavior** - Usually `'cascade'` for dependent data

---

## Migration System

### Why Transaction Safety Matters

**The Critical Problem:** Without transactions, a failed migration can leave your database in an inconsistent state - some changes applied, others not. This can corrupt data relationships and make recovery extremely difficult.

**The Solution:** Wrap all migrations in transactions with `behavior: 'immediate'` to prevent concurrent access and ensure atomic operations (all changes succeed or all rollback).

### Transaction-Safe Migration Runner (`src/lib/migrate.js`)

**⚠️ This is the ONLY safe way to run migrations in production**

```javascript
import 'dotenv/config';
import { createClient } from '@libsql/client';
import { drizzle } from 'drizzle-orm/libsql';
import { migrate } from 'drizzle-orm/libsql/migrator';
import { existsSync, mkdirSync, writeFileSync } from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import { sql } from 'drizzle-orm';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Path configuration
const DATA_DIR = process.env.NODE_ENV === 'production' ? '/app/data' : './data';
const DB_PATH = path.join(DATA_DIR, 'database.db');
const BACKUP_DIR = path.join(DATA_DIR, 'backups');
const MIGRATIONS_DIR = path.resolve(__dirname, '../../drizzle');

async function validateSchemaIntegrity(db) {
    try {
        console.log('Validating database schema integrity...');
        
        // Check Drizzle migration table exists
        const migrationTable = await db.run(sql`
            SELECT name FROM sqlite_master 
            WHERE type='table' AND name='__drizzle_migrations'
        `);
        
        if (!migrationTable.rows?.length) {
            throw new Error('Drizzle migration table missing');
        }
        
        // Validate foreign key constraints
        const fkCheck = await db.run(sql`PRAGMA foreign_key_check`);
        if (fkCheck.rows && fkCheck.rows.length > 0) {
            throw new Error('Foreign key constraint violations detected');
        }
        
        console.log('Schema validation passed.');
        return true;
    } catch (error) {
        console.error('Schema validation failed:', error);
        return false;
    }
}

async function createPreMigrationBackup(db) {
    try {
        if (process.env.NODE_ENV === 'production') {
            const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
            const backupPath = path.join(BACKUP_DIR, 'pre-migration', `backup-${timestamp}.sql`);
            
            mkdirSync(path.dirname(backupPath), { recursive: true });
            console.log(`Creating pre-migration backup: ${backupPath}`);
            
            // Get all application tables
            const tables = await db.run(sql`
                SELECT name FROM sqlite_master 
                WHERE type='table' 
                AND name NOT LIKE 'sqlite_%' 
                AND name != '__drizzle_migrations'
            `);
            
            let sqlDump = `-- Pre-migration backup\n-- Created: ${new Date().toISOString()}\n\n`;
            
            // Export schema and data for each table
            for (const table of tables.rows) {
                const tableName = table.name;
                
                // Export table schema
                const schema = await db.run(sql`
                    SELECT sql FROM sqlite_master 
                    WHERE type='table' AND name = ${tableName}
                `);
                sqlDump += `${schema.rows[0].sql};\n\n`;
                
                // Export table data
                const rows = await db.run(sql`SELECT * FROM ${sql.raw(tableName)}`);
                for (const row of rows.rows) {
                    const values = Object.values(row).map(v => 
                        v === null ? 'NULL' : 
                        typeof v === 'string' ? `'${v.replace(/'/g, "''")}'` : v
                    );
                    sqlDump += `INSERT INTO ${tableName} VALUES (${values.join(', ')});\n`;
                }
                sqlDump += '\n';
            }
            
            writeFileSync(backupPath, sqlDump);
            console.log('Pre-migration backup created successfully.');
            return backupPath;
        }
        return null;
    } catch (error) {
        console.error('Failed to create pre-migration backup:', error);
        return null;
    }
}

async function runMigrations() {
    let db;
    let backupPath = null;
    
    try {
        // Ensure directories exist
        if (!existsSync(DATA_DIR)) mkdirSync(DATA_DIR, { recursive: true });
        if (!existsSync(BACKUP_DIR)) mkdirSync(BACKUP_DIR, { recursive: true });
        
        // Setup database connection
        const client = createClient({ url: process.env.DATABASE_URL });
        db = drizzle(client);
        
        // Create pre-migration backup
        if (existsSync(DB_PATH)) {
            backupPath = await createPreMigrationBackup(db);
        }
        
        // Run migrations within transaction for safety
        console.log('Running migrations with transaction safety...');
        await db.transaction(async (tx) => {
            await migrate(tx, { migrationsFolder: MIGRATIONS_DIR });
            
            // Validate schema within same transaction
            const isValid = await validateSchemaIntegrity(tx);
            if (!isValid) {
                throw new Error('Schema validation failed - rolling back');
            }
            
            console.log('Migrations completed within transaction');
        }, {
            behavior: 'immediate' // Prevents lock conflicts
        });
        
        console.log('Migration completed successfully');
        process.exit(0);
        
    } catch (error) {
        console.error('Migration failed:', error);
        process.exit(1);
    }
}

runMigrations();
```

### Migration Workflow

```bash
# 1. Modify schema.ts with your changes
# 2. Generate migration
npm run db:generate

# 3. Review generated SQL in ./drizzle/
# 4. Test migration in development
npm run db:migrate

# 5. Deploy to production (runs automatically via Docker)
```

---

## Environment & Path Management

### Critical Path Consistency

**⚠️ All database paths must be perfectly aligned to prevent data loss**

#### Development Environment
```
Database File: ./data/database.db
Drizzle Config: './data/database.db'
Environment Variable: 'file:data/database.db'
```

#### Production Environment (Docker)
```
Host Location: ./data/database.db
Container Mount: /app/data/database.db  
Volume Mount: ./data:/app/data
DATABASE_URL: file:/app/data/database.db
```

### Path Validation Script (`validate-paths.js`)

```javascript
const fs = require('fs');

const checks = [
    {
        file: 'production.env',
        pattern: 'DATABASE_URL=file:/app/data/database.db',
        description: 'Production environment database path'
    },
    {
        file: 'docker-compose.yml',
        pattern: './data:/app/data',
        description: 'Docker volume mount configuration'
    },
    {
        file: 'src/lib/migrate.js',
        pattern: "const DATA_DIR = process.env.NODE_ENV === 'production' ? '/app/data' : './data';",
        description: 'Migration script data directory'
    }
];

console.log('Validating database path consistency...\n');

let allValid = true;

checks.forEach(check => {
    try {
        const content = fs.readFileSync(check.file, 'utf8');
        if (content.includes(check.pattern)) {
            console.log(`✅ ${check.description}`);
        } else {
            console.log(`❌ ${check.description} - MISMATCH!`);
            allValid = false;
        }
    } catch (error) {
        console.log(`❌ ${check.description} - FILE NOT FOUND!`);
        allValid = false;
    }
});

if (allValid) {
    console.log('\n✅ All database paths are consistent - deployment safe!');
    process.exit(0);
} else {
    console.log('\n❌ Path inconsistencies detected - DO NOT DEPLOY!');
    process.exit(1);
}
```

---

## Docker Integration

### Dockerfile

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --prefer-offline --no-audit

COPY . .
COPY production.env ./
ENV DATABASE_URL="file:/app/data/database.db"
RUN npm run build

FROM node:20-alpine AS runner

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup

# Copy application
COPY --from=builder /app/package.json ./
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/src ./src
COPY --from=builder /app/drizzle ./drizzle

RUN npm ci --omit=dev --prefer-offline --no-audit

ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000
USER appuser

# Run migrations before starting app
CMD node src/lib/migrate.js && npm start
```

### Docker Compose

```yaml
services:
  app:
    image: your-app:latest
    container_name: your-app-container
    restart: unless-stopped
    env_file:
      - ./production.env
    environment:
      - NODE_ENV=production
    volumes:
      - ./data:/app/data
    ports:
      - "3000:3000"
```

### Docker Permission Troubleshooting

**Common Issue:** Database file permission errors in container

**Symptoms:**
```
Error: SQLITE_CANTOPEN: unable to open database file
Permission denied: '/app/data/database.db'
```

**Solutions:**
```bash
# Fix host directory permissions
sudo chown -R 1001:1001 ./data
chmod -R 755 ./data

# Alternative: Use Docker user mapping
docker run --user 1001:1001 your-app

# Verify permissions in container
docker exec -it container-name ls -la /app/data
docker exec -it container-name id  # Check user ID
```

**Prevention:**
- Always create data directory with correct ownership before first run
- Use non-root user in Dockerfile (already included above)
- Test container startup with fresh data directory

---

## Backup & Restore System

### JSON Backup Service

```typescript
// src/services/backup.ts
import fs from 'fs/promises';
import path from 'path';
import { db } from '@/lib/db';
// Import your schema tables here
import { users, posts } from '@/lib/db/schema';

export async function createBackup(): Promise<string> {
    try {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const backupDir = process.env.NODE_ENV === 'production' 
            ? '/app/data/backups/json' 
            : './data/backups/json';
        const filename = `backup-${timestamp}.json`;
        const filePath = path.join(backupDir, filename);
        
        await fs.mkdir(backupDir, { recursive: true });
        
        // Export all data (customize tables for your app)
        const backup = {
            metadata: {
                version: '1.0',
                created: new Date().toISOString(),
                tables: ['users', 'posts'], // Update with your tables
            },
            users: await db.select().from(users),
            posts: await db.select().from(posts),
            // Add other tables as needed
        };
        
        await fs.writeFile(filePath, JSON.stringify(backup, null, 2));
        console.log(`Backup created: ${filePath}`);
        return filename;
        
    } catch (error) {
        console.error('Backup creation failed:', error);
        throw error;
    }
}

export async function restoreFromBackup(filename: string): Promise<void> {
    try {
        const backupDir = process.env.NODE_ENV === 'production' 
            ? '/app/data/backups/json' 
            : './data/backups/json';
        const filePath = path.join(backupDir, filename);
        
        const backupContent = await fs.readFile(filePath, 'utf8');
        const backup = JSON.parse(backupContent);
        
        if (!backup.metadata || !backup.metadata.version) {
            throw new Error('Invalid backup file format');
        }
        
        // Restore data within transaction for safety
        await db.transaction(async (tx) => {
            console.log('Restoring from backup...');
            
            // Clear existing data in dependency order
            await tx.delete(posts);
            await tx.delete(users);
            
            // Restore data in dependency order
            if (backup.users?.length > 0) {
                await tx.insert(users).values(backup.users);
            }
            if (backup.posts?.length > 0) {
                await tx.insert(posts).values(backup.posts);
            }
            
            // Reset auto-increment sequences
            const resetSequence = async (tableName: string, data: any[]) => {
                if (data && data.length > 0) {
                    const maxId = Math.max(...data.map((row: any) => row.id || 0));
                    if (maxId > 0) {
                        await tx.run(sql.raw(`UPDATE sqlite_sequence SET seq = ${maxId} WHERE name = '${tableName}'`));
                    }
                }
            };
            
            await resetSequence('users', backup.users || []);
            await resetSequence('posts', backup.posts || []);
        });
        
        console.log('Backup restored successfully');
        
    } catch (error) {
        console.error('Backup restore failed:', error);
        throw error;
    }
}
```

---

## Critical Pitfalls & Solutions

### 1. Migration Strategy Anti-Pattern

**❌ DANGEROUS (Never Do This):**
```typescript
// DON'T DO THIS - Runs on every import!
(async () => {
    await migrate(db, { migrationsFolder: './drizzle' });
})();
```

**✅ SAFE Solution:**
```typescript
// Use separate migration script
// Only run migrations via: node src/lib/migrate.js
```

**Why This Matters:** Automatic migrations on import create race conditions, can run multiple times simultaneously, and make debugging impossible. Always use explicit migration scripts.

### 2. Path Consistency Issues

**❌ Wrong:**
```
Development: ./data/database.db
Production: /data/database.db    # Missing /app prefix
```

**✅ Correct:**
```
Development: ./data/database.db
Production: /app/data/database.db
Docker Mount: ./data:/app/data
```

### 3. Missing Transaction Safety

**❌ Unsafe:**
```javascript
await migrate(db, { migrationsFolder: './drizzle' });
```

**✅ Safe:**
```javascript
await db.transaction(async (tx) => {
    await migrate(tx, { migrationsFolder: './drizzle' });
}, { behavior: 'immediate' });
```

**Why This Matters:** Without transactions, failed migrations leave your database in an inconsistent state. Transaction rollback ensures all-or-nothing migration execution.

### 4. Auto-Increment Sequence Issues

**Problem:** After backup restore, new records get conflicting IDs.

**Solution:**
```sql
UPDATE sqlite_sequence SET seq = (SELECT MAX(id) FROM users) WHERE name = 'users';
```

---

## Testing Checklist

### Pre-Production Migration Testing

- [ ] Create copy of production database for testing
- [ ] Run migration against production copy
- [ ] Verify all schema changes applied correctly
- [ ] Test application functionality with new schema
- [ ] Verify rollback procedure works
- [ ] Measure migration execution time

### Backup & Restore Testing

- [ ] Create backup of test database
- [ ] Verify backup file integrity
- [ ] Restore backup to fresh database
- [ ] Compare original vs restored data
- [ ] Test application against restored database
- [ ] Check auto-increment sequences are correct

### Environment Validation

- [ ] Verify database paths match across environments
- [ ] Test Docker volume mounts persist data
- [ ] Validate backup directory permissions
- [ ] Test migration script execution
- [ ] Run path validation script

---

## Troubleshooting Guide

### "Database file not found" Error

**Diagnosis:**
```bash
ls -la data/database.db
echo $DATABASE_URL
docker inspect container-name | grep Mounts
```

**Solutions:**
- Verify DATABASE_URL matches file location
- Check Docker volume mount configuration
- Ensure data directory exists with proper permissions

### Migration Fails with Foreign Key Error

**Diagnosis:**
```sql
PRAGMA foreign_key_list(table_name);
SELECT sql FROM sqlite_master WHERE type='table';
```

**Solutions:**
- Ensure parent tables created before child tables
- Check for orphaned records violating constraints
- Review migration file order

### Database Locked Error (SQLITE_BUSY)

**Diagnosis:**
```bash
lsof data/database.db  # Check open connections
ps aux | grep node      # Check running processes
```

**Solutions:**
```typescript
// Set busy timeout
await client.execute('PRAGMA busy_timeout = 5000;');

// Use immediate transactions
await db.transaction(async (tx) => {
    // operations
}, { behavior: 'immediate' });
```

### Performance Issues

**Diagnosis:**
```sql
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = ?;
SELECT page_count * page_size / 1024 / 1024 as 'DB Size MB' 
FROM pragma_page_count(), pragma_page_size();
```

**Solutions:**
- Add missing indexes for frequently queried columns
- Run `VACUUM` to reclaim space
- Consider data archiving for old records

---

## Security & Production Considerations

### Database Security

**File Permissions:**
```bash
# Secure database file
chmod 640 data/database.db
chown app:app data/database.db

# Secure backup directory
chmod 750 data/backups
```

**Connection Security:**
```typescript
// Use environment variables for sensitive config
const client = createClient({
    url: process.env.DATABASE_URL,
    authToken: process.env.DATABASE_AUTH_TOKEN, // For remote SQLite
});
```

### Production Monitoring

**Essential Monitoring:**
- Database file size growth
- Query performance metrics
- Migration execution times
- Backup creation success/failure
- Disk space utilization

**Health Check Endpoint:**
```typescript
// Add to your API routes
export async function GET() {
    try {
        await db.run(sql`SELECT 1`);
        return Response.json({ status: 'healthy', timestamp: new Date().toISOString() });
    } catch (error) {
        return Response.json({ status: 'unhealthy', error: error.message }, { status: 500 });
    }
}
```

### Data Validation

**Schema Integrity Checks:**
```sql
-- Run periodically to ensure data integrity
PRAGMA integrity_check;
PRAGMA foreign_key_check;
PRAGMA quick_check;
```

---

## Best Practices

### Database Design
- Use `integer` for primary keys with auto-increment
- Add indexes for all foreign keys and frequently queried columns
- Use enums for status fields
- Set appropriate `onDelete` behavior for foreign keys

### Migration Strategy
- Never run migrations automatically on app startup
- Always backup before migrations in production
- Test migrations on production data copy
- Keep migrations small and focused
- Use transactions for all migration operations

### Production Deployment
- Validate paths before deployment
- Monitor migration performance
- Have rollback plan ready
- Use proper logging and monitoring
- Test backup/restore procedures regularly

### Performance
- Add indexes strategically based on query patterns
- Use transactions for bulk operations
- Monitor query performance
- Consider pagination for large result sets

---

*This guide is based on real production implementation experience. Always test changes in development before applying to production.*
