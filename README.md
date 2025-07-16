# Cube.dev Cheatsheet

[See example usage](./example.md)

## Basic Concepts

### Core Components

- **Cubes**: Define your data model and business logic - like tables but with analytics superpowers
- **Measures**: Aggregated values (sum, count, avg, etc.) - the numbers you want to calculate
- **Dimensions**: Attributes for grouping and filtering - the "by what" in your analysis
- **Segments**: Reusable filters - predefined WHERE clauses you can apply to any query
- **Pre-aggregations**: Cached aggregated data for performance - pre-calculated results stored for fast queries

### File Structure

```
schema/
├── Users.js
├── Orders.js
└── Products.js
```

## Data Schema Definition

### Basic Cube Structure

```javascript
cube(`Users`, {
  sql: `SELECT * FROM users`,

  measures: {
    count: {
      type: `count`
    }
  },

  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primaryKey: true
    },

    name: {
      sql: `name`,
      type: `string`
    },

    createdAt: {
      sql: `created_at`,
      type: `time`
    }
  }
});
```

### SQL Table Reference

```javascript
cube(`Orders`, {
  sql: `SELECT * FROM orders WHERE status = 'completed'`,
  // Alternative with parameters
  sql: `SELECT * FROM ${CUBE.sqlTable()}`,
  sqlTable: `orders`
});
```

## Measures

*Numerical calculations performed on your data - think SUM, COUNT, AVG of columns*

### Basic Measure Types

```javascript
measures: {
  // Count
  count: {
    type: `count`
  },

  // Sum
  totalRevenue: {
    sql: `revenue`,
    type: `sum`
  },

  // Average
  avgOrderValue: {
    sql: `revenue`,
    type: `avg`
  },

  // Min/Max
  minOrderValue: {
    sql: `revenue`,
    type: `min`
  },

  // Count Distinct
  uniqueUsers: {
    sql: `user_id`,
    type: `countDistinct`
  }
}
```

### Advanced Measures

```javascript
measures: {
  // Number measure with custom SQL
  conversionRate: {
    sql: `${successfulOrders} / ${totalOrders} * 100`,
    type: `number`
  },

  // Calculated field
  averageOrderValue: {
    sql: `${totalRevenue} / ${count}`,
    type: `number`
  },

  // Rolling window
  rollingRevenue: {
    sql: `revenue`,
    type: `sum`,
    rollingWindow: {
      trailing: `30 day`
    }
  }
}
```

## Dimensions

*Attributes used to slice and dice your data - like categories, dates, or text fields*

### Basic Dimension Types

```javascript
dimensions: {
  // String
  status: {
    sql: `status`,
    type: `string`
  },

  // Number
  age: {
    sql: `age`,
    type: `number`
  },

  // Boolean
  isActive: {
    sql: `is_active`,
    type: `boolean`
  },

  // Time
  createdAt: {
    sql: `created_at`,
    type: `time`
  }
}
```

### Advanced Dimensions

```javascript
dimensions: {
  // Case statement
  ageGroup: {
    sql: `CASE
      WHEN age < 18 THEN 'Under 18'
      WHEN age < 65 THEN 'Adult'
      ELSE 'Senior'
    END`,
    type: `string`
  },

  // Formatted dimension
  fullName: {
    sql: `CONCAT(first_name, ' ', last_name)`,
    type: `string`
  },

  // Geography
  location: {
    sql: `CONCAT(city, ', ', state)`,
    type: `geo`
  }
}
```

## Time Dimensions

### Time Granularities

```javascript
dimensions: {
  createdAt: {
    sql: `created_at`,
    type: `time`
  }
}

// Available granularities in queries:
// second, minute, hour, day, week, month, quarter, year
```

### Custom Time Formats

```javascript
dimensions: {
  orderDate: {
    sql: `DATE(created_at)`,
    type: `time`
  },

  orderMonth: {
    sql: `DATE_TRUNC('month', created_at)`,
    type: `time`
  }
}
```

## Segments

*Reusable filter conditions that you can apply to any query - like "active users" or "premium customers"*

### Basic Segments

```javascript
segments: {
  activeUsers: {
    sql: `${CUBE}.status = 'active'`
  },

  recentOrders: {
    sql: `${CUBE}.created_at >= CURRENT_DATE - INTERVAL '30 days'`
  },

  highValueCustomers: {
    sql: `${totalLifetimeValue} > 1000`
  }
}
```

## Joins

*Connect related cubes together - like linking orders to customers or products*

### Basic Joins

```javascript
cube(`Orders`, {
  sql: `SELECT * FROM orders`,

  joins: {
    Users: {
      sql: `${CUBE}.user_id = ${Users}.id`,
      relationship: `belongsTo`
    },

    Products: {
      sql: `${CUBE}.product_id = ${Products}.id`,
      relationship: `belongsTo`
    }
  }
});
```

### Join Relationships

```javascript
joins: {
  // Many-to-One
  User: {
    sql: `${CUBE}.user_id = ${Users}.id`,
    relationship: `belongsTo`
  },

  // One-to-Many
  OrderItems: {
    sql: `${CUBE}.id = ${OrderItems}.order_id`,
    relationship: `hasMany`
  },

  // One-to-One
  UserProfile: {
    sql: `${CUBE}.id = ${UserProfiles}.user_id`,
    relationship: `hasOne`
  }
}
```

## Pre-aggregations

*Pre-calculated and cached query results that make your dashboards lightning fast*

### Basic Pre-aggregations

```javascript
preAggregations: {
  main: {
    measures: [count, totalRevenue],
    dimensions: [status, createdAt],
    timeDimension: createdAt,
    granularity: `day`
  }
}
```

### Advanced Pre-aggregations

```javascript
preAggregations: {
  // Rollup with refresh
  dailyRollup: {
    measures: [count, totalRevenue],
    dimensions: [status],
    timeDimension: createdAt,
    granularity: `day`,
    refreshKey: {
      every: `1 hour`
    }
  },

  // Original SQL pre-aggregation
  monthlyStats: {
    type: `originalSql`,
    refreshKey: {
      every: `1 day`
    }
  },

  // Partitioned pre-aggregation
  partitioned: {
    measures: [count],
    dimensions: [status],
    timeDimension: createdAt,
    granularity: `day`,
    partitionGranularity: `month`
  }
}
```

## API Queries

### REST API Query Format

```javascript
{
  "measures": ["Orders.count", "Orders.totalRevenue"],
  "dimensions": ["Orders.status", "Users.city"],
  "timeDimensions": [{
    "dimension": "Orders.createdAt",
    "granularity": "day",
    "dateRange": ["2023-01-01", "2023-12-31"]
  }],
  "filters": [{
    "member": "Orders.status",
    "operator": "equals",
    "values": ["completed"]
  }],
  "segments": ["Orders.highValue"],
  "limit": 1000,
  "offset": 0,
  "order": {
    "Orders.createdAt": "desc"
  }
}
```

### GraphQL Query

```graphql
query {
  cube(
    measures: ["Orders.count"]
    dimensions: ["Orders.status"]
    timeDimensions: [{
      dimension: "Orders.createdAt"
      granularity: "month"
    }]
  ) {
    Orders {
      count
      status
      createdAt
    }
  }
}
```

## Filters and Operators

### Filter Operators

```javascript
// Comparison operators
{ "member": "Orders.amount", "operator": "gt", "values": ["100"] }
{ "member": "Orders.amount", "operator": "gte", "values": ["100"] }
{ "member": "Orders.amount", "operator": "lt", "values": ["1000"] }
{ "member": "Orders.amount", "operator": "lte", "values": ["1000"] }
{ "member": "Orders.status", "operator": "equals", "values": ["completed"] }
{ "member": "Orders.status", "operator": "notEquals", "values": ["cancelled"] }

// Array operators
{ "member": "Orders.status", "operator": "contains", "values": ["completed", "shipped"] }
{ "member": "Orders.status", "operator": "notContains", "values": ["cancelled"] }

// String operators
{ "member": "Users.name", "operator": "startsWith", "values": ["John"] }
{ "member": "Users.name", "operator": "endsWith", "values": ["Smith"] }

// Date operators
{ "member": "Orders.createdAt", "operator": "inDateRange", "values": ["2023-01-01", "2023-12-31"] }
{ "member": "Orders.createdAt", "operator": "beforeDate", "values": ["2023-06-01"] }
{ "member": "Orders.createdAt", "operator": "afterDate", "values": ["2023-01-01"] }

// Set operators
{ "member": "Orders.amount", "operator": "set" }
{ "member": "Orders.amount", "operator": "notSet" }
```

## Environment Configuration

### Database Connection

```javascript
// cube.js
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

```javascript
// cube.js
module.exports = {
  contextToAppId: ({ securityContext }) => {
    return `CUBEJS_APP_${securityContext.tenantId}`;
  },

  scheduledRefreshContexts: () => {
    return [
      { securityContext: { tenantId: 'tenant1' } },
      { securityContext: { tenantId: 'tenant2' } }
    ];
  }
};
```

## Common Patterns

### Multi-tenant Schema

```javascript
cube(`Orders`, {
  sql: `SELECT * FROM orders WHERE tenant_id = '${SECURITY_CONTEXT.tenantId}'`,

  measures: {
    count: {
      type: `count`
    }
  }
});
```

### Dynamic Schema

```javascript
// Dynamic table based on context
cube(`Events`, {
  sql: () => `SELECT * FROM events_${SECURITY_CONTEXT.source}`,

  // Context-based filtering
  sql: `SELECT * FROM events WHERE app_id = '${SECURITY_CONTEXT.appId}'`
});
```

### Derived Measures

```javascript
measures: {
  conversionRate: {
    sql: `CASE
      WHEN ${totalVisits} > 0
      THEN ${conversions}::float / ${totalVisits}::float * 100
      ELSE 0
    END`,
    type: `number`,
    format: `percent`
  }
}
```

### Complex Aggregations

```javascript
measures: {
  // Median calculation
  medianOrderValue: {
    sql: `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ${CUBE}.amount)`,
    type: `number`
  },

  // Cohort retention
  cohortRetention: {
    sql: `COUNT(DISTINCT CASE
      WHEN ${CUBE}.cohort_week = 0 THEN ${CUBE}.user_id
    END)::float / NULLIF(COUNT(DISTINCT ${CUBE}.user_id), 0) * 100`,
    type: `number`
  }
}
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

```javascript
refreshKey: {
  // Time-based refresh
  every: `1 hour`,

  // SQL-based refresh
  sql: `SELECT MAX(updated_at) FROM orders`,

  // Incremental refresh
  incremental: true,
  updateWindow: `7 day`
}
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
