# Write audit log filter definitions

Audit log filters control which database activities get logged, reducing log size and storage costs. Instead of logging everything (which creates huge files), you can log only what matters to you.

Why use filters?
* Reduce log size and storage costs

* Focus on security events and data changes

* Meet compliance requirements without noise

* Improve performance by reducing I/O overhead

Prerequisites:
* Percona Server for MySQL with audit log filter plugin installed

* `AUDIT_ADMIN` privilege

* Basic understanding of JSON syntax

## Quick start (5 minutes)

### Step 1: Create a filter
```sql
SET @my_filter = '{
  "filter": {
    "class": [
      { "name": "connection" }
    ]
  }
}';
SELECT audit_log_filter_set_filter('my_filter', @my_filter);
```

### Step 2: Assign to user
```sql
SELECT audit_log_filter_set_user('user1'@'%', 'my_filter');
```

### Step 3: Test
Connect as 'user1', run a query, and check `/var/lib/mysql/audit.log`.

## JSON filter structure

Every filter follows this pattern:

```json
{
  "filter": {
    "class": [
      {
        "name": "connection",
        "log": true
      }
    ]
  }
}
```

Event types you can filter:

* `"connection"` - User logins/logouts (track who accesses the database)

* `"general"` - SQL queries and commands (monitor all database activity)

* `"table_access"` - Table operations (SELECT, INSERT, UPDATE, DELETE)

* `"query"` - Query execution details (debug slow queries)

Optional fields:

* `"user"` - Array of usernames to track

* `"host"` - Array of client IPs to track  

* `"event"` - Array of specific event subclasses to track

* `"log"` - Set to `true` or `false` to enable/disable logging for this class

## Common patterns

Log everything (good for testing):
```json
{
  "filter": {
    "class": [
      { "name": "connection" },
      { "name": "general" },
      { "name": "table_access" }
    ]
  }
}
```

Track specific users only:
```json
{
  "filter": {
    "class": [
      {
        "name": "table_access",
        "event": [
          { "name": "insert" },
          { "name": "update" },
          { "name": "delete" }
        ]
      }
    ]
  }
}
```

Monitor data changes only:
```json
{
  "filter": {
    "class": [
      {
        "name": "table_access",
        "event": [
          { "name": "insert" },
          { "name": "update" },
          { "name": "delete" }
        ]
      }
    ]
  }
}
```

Track specific operations:
```json
{
  "filter": {
    "class": [
      {
        "name": "table_access",
        "event": ["insert", "update", "delete"]
      }
    ]
  }
}
```

## Create and assign a filter

Step 1: Create the filter
```sql
SET @my_filter = '{
  "filter": {
    "class": [
      {
        "name": "connection",
        "log": true
      }
    ]
  }
}';
SELECT audit_log_filter_set_filter('user_connections', @my_filter);
```

Step 2: Assign to users
```sql
SELECT audit_log_filter_set_user('user1'@'%', 'user_connections');
SELECT audit_log_filter_set_user('user2'@'localhost', 'user_connections');
```

Step 3: Test your filter

1. Connect as 'user1' and run: `SELECT 1;`

2. Check the audit log: `tail -f /var/lib/mysql/audit.log`

3. You should see JSON entries for the connection and query

What this does: Creates a filter that logs connection events for users `user1` and `user2`, then assigns the filter to user `user1`. When `user1` connects and runs queries, those activities will be logged according to the filter rules.

## Troubleshooting

Check your setup:
```sql
SELECT * FROM mysql.audit_log_filter;  -- See all filters
SELECT * FROM mysql.audit_log_user;    -- See user assignments
SELECT * FROM information_schema.plugins WHERE plugin_name = 'audit_log'; -- Check plugin status
```

Common issues and solutions:

| Problem | Cause | Solution |
|---------|-------|----------|
| No events in log | Filter not assigned to user | Run: `SELECT audit_log_filter_set_user('user@host', 'filter_name');` |
| Access denied | Missing privileges | Grant `AUDIT_ADMIN` privilege to your user |
| JSON syntax error | Invalid JSON format | Validate JSON with a JSON validator |
| Filter not working | Plugin not loaded | Check: `SELECT * FROM information_schema.plugins WHERE plugin_name = 'audit_log';` |

Test your filter:

1. Connect as the user you assigned the filter to

2. Run a simple query: `SELECT 1;`

3. Check the audit log: `tail -f /var/lib/mysql/audit.log`

4. You should see JSON entries for your activities

## Advanced: Default filters

Create a filter that applies to ALL users who don't have a specific filter:

```sql
SET @default_filter = '{
  "filter": {
    "class": { "name": "general" }
  }
}';
SELECT audit_log_filter_set_filter('default_filter', @default_filter);
SELECT audit_log_filter_set_user('%', 'default_filter');
```

This configuration creates a default filter that logs general events for any user not explicitly assigned a different filter.

## Reference

Core functions:
```sql
SELECT audit_log_filter_set_filter('name', 'json');
SELECT audit_log_filter_set_user('user@host', 'filter_name');
SELECT audit_log_filter_remove_filter('name');
SELECT audit_log_filter_remove_user('user@host');
```

Log management:
```sql
SET GLOBAL audit_log_filter_rotate_on_size = 1073741824;  -- 1GB per file
SET GLOBAL audit_log_filter_max_size = 2147483648;        -- 2GB total
SET GLOBAL audit_log_filter_prune_seconds = 604800;       -- 7 days
```

Quick workflow:

1. Create filter with `audit_log_filter_set_filter()`

2. Assign to users with `audit_log_filter_set_user()`

3. Test by connecting and checking `/var/lib/mysql/audit.log`

## Cheat sheet

### Quick setup (TL;DR)
```sql
-- 1. Create a filter
SET @filter = '{"filter":{"class":[{"name":"connection","log":true}]}}';
SELECT audit_log_filter_set_filter('my_filter', @filter);

-- 2. Assign to user
SELECT audit_log_filter_set_user('user1'@'%', 'my_filter');

-- 3. Test the filter
-- Connect as user1, run SELECT 1;, check /var/lib/mysql/audit.log
```

### All commands
```sql
-- Create/remove filters
SELECT audit_log_filter_set_filter('name', 'json');
SELECT audit_log_filter_remove_filter('name');

-- Assign/remove users
SELECT audit_log_filter_set_user('user@host', 'filter_name');
SELECT audit_log_filter_remove_user('user@host');

-- Check status
SELECT * FROM mysql.audit_log_filter;
SELECT * FROM mysql.audit_log_user;
SELECT * FROM information_schema.plugins WHERE plugin_name = 'audit_log';

-- Log management
SET GLOBAL audit_log_filter_rotate_on_size = 1073741824;  -- 1GB
SET GLOBAL audit_log_filter_max_size = 2147483648;        -- 2GB
SET GLOBAL audit_log_filter_prune_seconds = 604800;       -- 7 days
```

### Common JSON patterns
```json
-- Log everything
{"filter":{"class":[{"name":"connection"},{"name":"general"},{"name":"table_access"}]}}

-- Track specific operations
{"filter":{"class":[{"name":"table_access","event":[{"name":"insert"},{"name":"update"},{"name":"delete"}]}]}}

-- Log connection events only
{"filter":{"class":[{"name":"connection","event":[{"name":"connect"},{"name":"disconnect"}]}]}}
```

## Related topics

* [Audit Log Filter Overview](audit-log-filter-overview.md) - Introduction to audit log filtering concepts

* [Advanced Audit Log Filter Definitions](audit-log-filter-advanced.md) - Advanced features and complex configurations

* [Audit Log Filter Reference](audit-log-filter-reference.md) - Complete reference tables and technical details
