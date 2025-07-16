# Cube.dev Cheatsheet

[See example usage](./example.md)

> ⚠️ Disclaimer: This cheatsheet is intended as a quick reference and learning walkthrough only. It is not an authoritative source and should not be relied upon for production decisions. Always refer to the official Cube.dev documentation for the most current, accurate, and comprehensive information. Features, syntax, and best practices may change between versions. Use this guide as a starting point, then validate everything against official sources.

## Basic Concepts

### Core Components

- **Cubes**: Define your data model and business logic - like tables but with analytics superpowers
- **Views**: Sit on top of cubes to create facades for data consumers - your final data products
- **Measures**: Aggregated values (sum, count, avg, etc.) - the numbers you want to calculate
- **Dimensions**: Attributes for grouping and filtering - the "by what" in your analysis
- **Segments**: Reusable filters - predefined WHERE clauses you can apply to any query
- **Pre-aggregations**: Cached aggregated data for performance - pre-calculated results stored for fast queries

### File Structure

```
model/
├── cubes/
│   ├── sales/
│   │   ├── orders.yml
│   │   └── order_items.yml
│   └── users/
│       └── customers.yml
└── views/
    ├── sales/
    │   └── revenue_analysis.yml
    └── users/
        └── customer_insights.yml
```

## Data Schema Definition

### Basic Cube Structure (YAML - Recommended)

```yaml
cubes:
  - name: users
    sql_table: users

    measures:
      - name: count
        type: count

    dimensions:
      - name: id
        sql: id
        type: number
        primary_key: true

      - name: name
        sql: name
        type: string

      - name: created_at
        sql: created_at
        type: time
```

### Alternative JavaScript Syntax (When Needed)

```javascript
cube(`users`, {
  sql_table: `users`,

  measures: {
    count: {
      type: `count`
    }
  },

  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primary_key: true
    },

    name: {
      sql: `name`,
      type: `string`
    },

    created_at: {
      sql: `created_at`,
      type: `time`
    }
  }
});
```

### SQL Table Reference

```yaml
cubes:
  - name: orders
    sql_table: orders
    # Alternative with custom SQL
    # sql: >
    #   SELECT * FROM orders
    #   WHERE status = 'completed'
```

## Measures

*Numerical calculations performed on your data - think SUM, COUNT, AVG of columns*

### Basic Measure Types (YAML)

```yaml
measures:
  # Count
  - name: count
    type: count

  # Sum
  - name: total_revenue
    sql: revenue
    type: sum

  # Average
  - name: avg_order_value
    sql: revenue
    type: avg

  # Min/Max
  - name: min_order_value
    sql: revenue
    type: min

  # Count Distinct
  - name: unique_users
    sql: user_id
    type: count_distinct
```

### Advanced Measures

```yaml
measures:
  # Number measure with custom SQL
  - name: conversion_rate
    sql: "{successful_orders} / {total_orders} * 100"
    type: number

  # Calculated field
  - name: average_order_value
    sql: "{total_revenue} / {count}"
    type: number

  # Rolling window
  - name: rolling_revenue
    sql: revenue
    type: sum
    rolling_window:
      trailing: 30 day
```

## Dimensions

*Attributes used to slice and dice your data - like categories, dates, or text fields*

### Basic Dimension Types (YAML)

```yaml
dimensions:
  # String
  - name: status
    sql: status
    type: string

  # Number
  - name: age
    sql: age
    type: number

  # Boolean
  - name: is_active
    sql: is_active
    type: boolean

  # Time
  - name: created_at
    sql: created_at
    type: time
```

### Advanced Dimensions

```yaml
dimensions:
  # Case statement
  - name: age_group
    sql: >
      CASE
        WHEN age < 18 THEN 'Under 18'
        WHEN age < 65 THEN 'Adult'
        ELSE 'Senior'
      END
    type: string

  # Formatted dimension
  - name: full_name
    sql: "CONCAT(first_name, ' ', last_name)"
    type: string

  # Geography
  - name: location
    sql: "CONCAT(city, ', ', state)"
    type: geo
```

## Time Dimensions

### Time Granularities

```yaml
dimensions:
  - name: created_at
    sql: created_at
    type: time

# Available granularities in queries:
# second, minute, hour, day, week, month, quarter, year
```

### Custom Time Formats

```yaml
dimensions:
  - name: order_date
    sql: "DATE(created_at)"
    type: time

  - name: order_month
    sql: "DATE_TRUNC('month', created_at)"
    type: time
```

## Segments

*Reusable filter conditions that you can apply to any query - like "active users" or "premium customers"*

### Basic Segments (YAML)

```yaml
segments:
  - name: active_users
    sql: "{CUBE}.status = 'active'"

  - name: recent_orders
    sql: "{CUBE}.created_at >= CURRENT_DATE - INTERVAL '30 days'"

  - name: high_value_customers
    sql: "{total_lifetime_value} > 1000"
```

## Joins

*Connect related cubes together - like linking orders to customers or products*

### Basic Joins (YAML)

```yaml
cubes:
  - name: orders
    sql_table: orders

    joins:
      - name: users
        sql: "{CUBE}.user_id = {users.id}"
        relationship: many_to_one

      - name: products
        sql: "{CUBE}.product_id = {products.id}"
        relationship: many_to_one
```

### Join Relationships

```yaml
joins:
  # Many-to-One
  - name: users
    sql: "{CUBE}.user_id = {users.id}"
    relationship: many_to_one

  # One-to-Many
  - name: order_items
    sql: "{CUBE}.id = {order_items.order_id}"
    relationship: one_to_many

  # One-to-One
  - name: user_profiles
    sql: "{CUBE}.id = {user_profiles.user_id}"
    relationship: one_to_one
```

## Views

*Create facades of your data model for data consumers - final data products for BI users and apps*

### Basic Views (YAML)

```yaml
views:
  - name: orders_view
    cubes:
      - join_path: orders
        includes:
          - status
          - created_at
          - count
          - total_revenue

      - join_path: orders.users
        prefix: true
        includes:
          - city
          - state
```

### Advanced Views

```yaml
views:
  - name: revenue_analysis
    title: Revenue Analysis Dashboard
    description: Complete revenue metrics for business intelligence

    cubes:
      - join_path: orders
        includes: "*"

      - join_path: orders.products
        prefix: true
        includes:
          - category
          - name

      - join_path: orders.users
        prefix: true
        includes:
          - city
          - state
```

## Pre-aggregations

*Pre-calculated and cached query results that make your dashboards lightning fast*

### Basic Pre-aggregations (YAML)

```yaml
pre_aggregations:
  - name: main
    measures:
      - count
      - total_revenue
    dimensions:
      - status
      - created_at
    time_dimension: created_at
    granularity: day
```

### Advanced Pre-aggregations

```yaml
pre_aggregations:
  # Rollup with refresh
  - name: daily_rollup
    measures:
      - count
      - total_revenue
    dimensions:
      - status
    time_dimension: created_at
    granularity: day
    refresh_key:
      every: 1 hour

  # Partitioned pre-aggregation
  - name: partitioned
    measures:
      - count
    dimensions:
      - status
    time_dimension: created_at
    granularity: day
    partition_granularity: month
```

## API Queries

### REST API Query Format

```javascript
{
  "measures": ["orders.count", "orders.total_revenue"],
  "dimensions": ["orders.status", "users.city"],
  "timeDimensions": [{
    "dimension": "orders.created_at",
    "granularity": "day",
    "dateRange": ["2023-01-01", "2023-12-31"]
  }],
  "filters": [{
    "member": "orders.status",
    "operator": "equals",
    "values": ["completed"]
  }],
  "segments": ["orders.high_value"],
  "limit": 1000,
  "offset": 0,
  "order": {
    "orders.created_at": "desc"
  }
}
```

### GraphQL Query

```graphql
query {
  cube(
    measures: ["orders.count"]
    dimensions: ["orders.status"]
    timeDimensions: [{
      dimension: "orders.created_at"
      granularity: "month"
    }]
  ) {
    orders {
      count
      status
      created_at
    }
  }
}
```

### SQL API

*Query Cube.dev using standard SQL syntax - great for BI tools and SQL-familiar developers*

```sql
-- Basic query
SELECT
  count,
  status,
  DATE_TRUNC('month', "orders.created_at") as month
FROM orders
WHERE status = 'completed'
ORDER BY month DESC;

-- Joins across cubes
SELECT
  "orders.count",
  "orders.total_revenue",
  "users.city",
  "products.category"
FROM orders
JOIN users ON TRUE
JOIN products ON TRUE
WHERE "orders.status" = 'completed';

-- Time dimensions with granularity
SELECT
  "orders.total_revenue",
  DATE_TRUNC('day', "orders.created_at") as order_date
FROM orders
WHERE "orders.created_at" >= '2023-01-01'
GROUP BY order_date;
```

**SQL API Transport Options:**

- **Postgres** (default): Same protocol as psql - works with any PostgreSQL-compatible tool
- **HTTP**: JSON-based protocol - use for embedded analytics when REST API isn't available

```bash
# Postgres transport example
psql -h localhost -p 5432 -d cubedb -U cube
```

**Benefits:** Use existing SQL knowledge, connect BI tools directly, leverage SQL functions

> **📝 Note**: Cube.dev also supports additional APIs and integrations including **WebSocket subscriptions** for real-time data, **embedded analytics SDKs**, **semantic layer integrations** with various BI tools, and **custom API extensions**. Check the Cube.dev documentation for the complete list of available integrations.

## Filters and Operators

### Filter Operators

```javascript
// Comparison operators
{ "member": "orders.amount", "operator": "gt", "values": ["100"] }
{ "member": "orders.amount", "operator": "gte", "values": ["100"] }
{ "member": "orders.amount", "operator": "lt", "values": ["1000"] }
{ "member": "orders.amount", "operator": "lte", "values": ["1000"] }
{ "member": "orders.status", "operator": "equals", "values": ["completed"] }
{ "member": "orders.status", "operator": "notEquals", "values": ["cancelled"] }

// Array operators
{ "member": "orders.status", "operator": "contains", "values": ["completed", "shipped"] }
{ "member": "orders.status", "operator": "notContains", "values": ["cancelled"] }

// String operators
{ "member": "users.name", "operator": "startsWith", "values": ["John"] }
{ "member": "users.name", "operator": "endsWith", "values": ["Smith"] }

// Date operators
{ "member": "orders.created_at", "operator": "inDateRange", "values": ["2023-01-01", "2023-12-31"] }
{ "member": "orders.created_at", "operator": "beforeDate", "values": ["2023-06-01"] }
{ "member": "orders.created_at", "operator": "afterDate", "values": ["2023-01-01"] }

// Set operators
{ "member": "orders.amount", "operator": "set" }
{ "member": "orders.amount", "operator": "notSet" }
```

## Environment Configuration

### Database Connection (cube.py or cube.js)

```python
# cube.py (Python configuration)
from cube import config

@config('db_type')
def db_type(ctx: RequestContext) -> str:
    return 'postgres'

@config('driver_factory')
def driver_factory(ctx: RequestContext):
    return {
        'type': 'postgres',
        'host': 'localhost',
        'database': 'analytics',
        'username': 'cube_user',
        'password': 'password',
        'port': 5432,
    }
```

```javascript
// cube.js (JavaScript configuration)
module.exports = {
  dbType: 'postgres',
  driverFactory: ({ dataSource }) => {
    return {
      type: 'postgres',
      host: process.env.CUBEJS_DB_HOST,
      database: process.env.CUBEJS_DB_NAME,
      username: process.env.CUBEJS_DB_USER,
      password: process.env.CUBEJS_DB_PASS,
      port: process.env.CUBEJS_DB_PORT,
    };
  }
};
```

### Security Context

```yaml
# In YAML data models with Jinja
cubes:
  - name: orders
    sql: >
      SELECT * FROM orders
      WHERE tenant_id = '{{ COMPILE_CONTEXT.security_context.tenant_id }}'

    measures:
      - name: revenue
        sql: total_amount
        type: sum
        # Field-level security
        public: "{{ 'admin' in COMPILE_CONTEXT.security_context.roles }}"
```

## Reference Syntax

### YAML Reference Patterns

```yaml
cubes:
  - name: orders
    sql_table: orders

    dimensions:
      # Column reference
      - name: status
        sql: status
        type: string

      # Member reference (same cube)
      - name: status_upper
        sql: "UPPER({status})"
        type: string

      # Cross-cube reference
      - name: user_name
        sql: "{users.name}"
        type: string

      # Current cube reference
      - name: order_id
        sql: "{CUBE}.id"
        type: number
        primary_key: true
```

### JavaScript Reference Patterns

```javascript
cube(`orders`, {
  sql_table: `orders`,

  dimensions: {
    // Column reference
    status: {
      sql: `status`,
      type: `string`
    },

    // Member reference (same cube)
    status_upper: {
      sql: `UPPER(${status})`,
      type: `string`
    },

    // Cross-cube reference
    user_name: {
      sql: `${users.name}`,
      type: `string`
    },

    // Current cube reference
    order_id: {
      sql: `${CUBE}.id`,
      type: `number`,
      primary_key: true
    }
  }
});
```

## Naming Conventions

### Best Practices

- Use **snake_case** for all names (cubes, measures, dimensions)
- Start with a letter, use only letters, numbers, and underscores
- Avoid Python reserved keywords (`from`, `return`, `yield`)

### Good Examples

```yaml
# Cubes
orders, stripe_invoices, base_payments

# Views
opportunities, cloud_accounts, arr

# Measures
count, avg_price, total_amount_shipped

# Dimensions
name, is_shipped, created_at

# Pre-aggregations
main, orders_by_status, lambda_invoices
```

## Performance Tips

### Optimization Best Practices

- Use pre-aggregations for frequently queried data combinations
- Partition large pre-aggregations by time
- Index dimensions used in filters and joins
- Use `sql` property efficiently to push down computations
- Leverage database-specific functions for better performance
- Set appropriate refresh keys for pre-aggregations
- Use segments for common filter patterns

### Pre-aggregation Refresh Strategy

```yaml
refresh_key:
  # Time-based refresh
  every: 1 hour

  # SQL-based refresh
  sql: "SELECT MAX(updated_at) FROM orders"

  # Incremental refresh
  incremental: true
  update_window: 7 day
```

## Debugging

### Common Commands

```bash
# Start development server
npm run dev

# Build production
npm run build

# Validate schema
npx cubejs-cli validate

# Generate schema from database
npx cubejs-cli generate
```

### Environment Variables

```bash
CUBEJS_DB_TYPE=postgres
CUBEJS_DB_HOST=localhost
CUBEJS_DB_NAME=analytics
CUBEJS_DB_USER=cube
CUBEJS_DB_PASS=password
CUBEJS_API_SECRET=your-secret-key
CUBEJS_DEV_MODE=true
```
