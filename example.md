# Cube.dev Workflow: E-commerce Analytics Example

## The Business Problem

You run an e-commerce store and want to track:

- **Daily sales revenue and order counts**
- **Top-selling products by category**
- **Customer acquisition and repeat purchase rates**
- **Performance metrics by region**

> **📝 Learning Note**: While Cube.dev provides a powerful web UI (Developer Playground) that can generate schemas, build queries visually, and manage configurations, this tutorial focuses on **file-based definitions and actual code content**. Understanding the underlying schema files, configuration syntax, and API structures is crucial for:
>
> - **Production deployments** where everything should be version-controlled
> - **Advanced customizations** that go beyond GUI capabilities
> - **Debugging and troubleshooting** when things don't work as expected
> - **Team collaboration** through code reviews and Git workflows
>
> We'll show you what's actually happening "under the hood" so you can fully leverage Cube.dev's capabilities!

## Sample Database Structure

### Tables

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255),
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  city VARCHAR(100),
  state VARCHAR(50),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- Products table
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  category VARCHAR(100),
  price DECIMAL(10,2),
  created_at TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  product_id INTEGER REFERENCES products(id),
  quantity INTEGER,
  total_amount DECIMAL(10,2),
  status VARCHAR(50), -- 'pending', 'completed', 'cancelled'
  order_date TIMESTAMP,
  created_at TIMESTAMP
);
```

### Sample Data

```sql
-- Sample users
INSERT INTO users VALUES
(1, 'john@email.com', 'John', 'Doe', 'New York', 'NY', '2023-01-15', '2023-01-15'),
(2, 'jane@email.com', 'Jane', 'Smith', 'Los Angeles', 'CA', '2023-02-01', '2023-02-01'),
(3, 'bob@email.com', 'Bob', 'Johnson', 'Chicago', 'IL', '2023-03-10', '2023-03-10');

-- Sample products
INSERT INTO products VALUES
(1, 'Wireless Headphones', 'Electronics', 99.99, '2023-01-01'),
(2, 'Coffee Mug', 'Home & Kitchen', 15.99, '2023-01-01'),
(3, 'Running Shoes', 'Sports', 129.99, '2023-01-01');

-- Sample orders
INSERT INTO orders VALUES
(1, 1, 1, 1, 99.99, 'completed', '2023-06-01', '2023-06-01'),
(2, 2, 2, 2, 31.98, 'completed', '2023-06-02', '2023-06-02'),
(3, 1, 3, 1, 129.99, 'completed', '2023-06-03', '2023-06-03'),
(4, 3, 1, 1, 99.99, 'pending', '2023-06-04', '2023-06-04');
```

## Step 1: Project Setup

### Initialize Cube.dev Project

```bash
npx cubejs-cli create ecommerce-analytics -d postgres
cd ecommerce-analytics
```

### Environment Configuration

```bash
# .env file
CUBEJS_DB_TYPE=postgres
CUBEJS_DB_HOST=localhost
CUBEJS_DB_NAME=ecommerce
CUBEJS_DB_USER=postgres
CUBEJS_DB_PASS=password
CUBEJS_DB_PORT=5432
CUBEJS_API_SECRET=your-secret-key-here
CUBEJS_DEV_MODE=true
```

## Step 2: Define Data Schema

### schema/Users.js

```javascript
cube(`Users`, {
  sql: `SELECT * FROM users`,

  measures: {
    count: {
      type: `count`,
      title: `Total Users`
    },

    newUsersCount: {
      sql: `id`,
      type: `countDistinct`,
      title: `New Users`,
      filters: [{
        sql: `${CUBE}.created_at >= CURRENT_DATE - INTERVAL '30 days'`
      }]
    }
  },

  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primaryKey: true
    },

    email: {
      sql: `email`,
      type: `string`
    },

    fullName: {
      sql: `CONCAT(first_name, ' ', last_name)`,
      type: `string`,
      title: `Full Name`
    },

    city: {
      sql: `city`,
      type: `string`
    },

    state: {
      sql: `state`,
      type: `string`
    },

    location: {
      sql: `CONCAT(city, ', ', state)`,
      type: `string`,
      title: `Location`
    },

    createdAt: {
      sql: `created_at`,
      type: `time`,
      title: `Registration Date`
    }
  },

  segments: {
    recentUsers: {
      sql: `${CUBE}.created_at >= CURRENT_DATE - INTERVAL '30 days'`,
      title: `Users registered in last 30 days`
    }
  }
});
```

### schema/Products.js

```javascript
cube(`Products`, {
  sql: `SELECT * FROM products`,

  measures: {
    count: {
      type: `count`,
      title: `Total Products`
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
      type: `string`,
      title: `Product Name`
    },

    category: {
      sql: `category`,
      type: `string`
    },

    price: {
      sql: `price`,
      type: `number`,
      format: `currency`
    },

    createdAt: {
      sql: `created_at`,
      type: `time`
    }
  }
});
```

### schema/Orders.js

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
  },

  measures: {
    count: {
      type: `count`,
      title: `Total Orders`
    },

    completedOrdersCount: {
      sql: `id`,
      type: `count`,
      title: `Completed Orders`,
      filters: [{
        sql: `${CUBE}.status = 'completed'`
      }]
    },

    totalRevenue: {
      sql: `total_amount`,
      type: `sum`,
      title: `Total Revenue`,
      format: `currency`
    },

    averageOrderValue: {
      sql: `total_amount`,
      type: `avg`,
      title: `Average Order Value`,
      format: `currency`
    },

    totalQuantity: {
      sql: `quantity`,
      type: `sum`,
      title: `Total Items Sold`
    }
  },

  dimensions: {
    id: {
      sql: `id`,
      type: `number`,
      primaryKey: true
    },

    userId: {
      sql: `user_id`,
      type: `number`
    },

    productId: {
      sql: `product_id`,
      type: `number`
    },

    status: {
      sql: `status`,
      type: `string`
    },

    totalAmount: {
      sql: `total_amount`,
      type: `number`,
      format: `currency`
    },

    quantity: {
      sql: `quantity`,
      type: `number`
    },

    orderDate: {
      sql: `order_date`,
      type: `time`,
      title: `Order Date`
    },

    createdAt: {
      sql: `created_at`,
      type: `time`
    }
  },

  segments: {
    completedOrders: {
      sql: `${CUBE}.status = 'completed'`,
      title: `Completed Orders Only`
    },

    highValueOrders: {
      sql: `${CUBE}.total_amount > 100`,
      title: `Orders over $100`
    },

    recentOrders: {
      sql: `${CUBE}.order_date >= CURRENT_DATE - INTERVAL '7 days'`,
      title: `Orders from last 7 days`
    }
  },

  preAggregations: {
    dailyRevenue: {
      measures: [totalRevenue, count],
      dimensions: [status],
      timeDimension: orderDate,
      granularity: `day`,
      refreshKey: {
        every: `1 hour`
      }
    }
  }
});
```

## Step 3: Start Development Server

```bash
npm run dev
```

This starts:

- **Cube.dev API** at <http://localhost:4000>
- **Developer Playground** at <http://localhost:4000/#/build>

## Step 4: Build Queries in Playground

### Query 1: Daily Sales Performance

```json
{
  "measures": [
    "Orders.totalRevenue",
    "Orders.completedOrdersCount",
    "Orders.averageOrderValue"
  ],
  "timeDimensions": [{
    "dimension": "Orders.orderDate",
    "granularity": "day",
    "dateRange": ["2023-06-01", "2023-06-30"]
  }],
  "segments": ["Orders.completedOrders"],
  "order": {
    "Orders.orderDate": "desc"
  }
}
```

**Expected Result:**

```json
[
  {
    "Orders.orderDate": "2023-06-03T00:00:00.000",
    "Orders.totalRevenue": "129.99",
    "Orders.completedOrdersCount": "1",
    "Orders.averageOrderValue": "129.99"
  },
  {
    "Orders.orderDate": "2023-06-02T00:00:00.000",
    "Orders.totalRevenue": "31.98",
    "Orders.completedOrdersCount": "1",
    "Orders.averageOrderValue": "31.98"
  },
  {
    "Orders.orderDate": "2023-06-01T00:00:00.000",
    "Orders.totalRevenue": "99.99",
    "Orders.completedOrdersCount": "1",
    "Orders.averageOrderValue": "99.99"
  }
]
```

### Query 2: Top Products by Category

```json
{
  "measures": [
    "Orders.totalRevenue",
    "Orders.totalQuantity"
  ],
  "dimensions": [
    "Products.category",
    "Products.name"
  ],
  "segments": ["Orders.completedOrders"],
  "order": {
    "Orders.totalRevenue": "desc"
  },
  "limit": 10
}
```

### Query 3: Customer Analysis by Location

```json
{
  "measures": [
    "Orders.totalRevenue",
    "Orders.count",
    "Users.count"
  ],
  "dimensions": [
    "Users.state",
    "Users.city"
  ],
  "segments": ["Orders.completedOrders"],
  "order": {
    "Orders.totalRevenue": "desc"
  }
}
```

## Step 5: API Integration

### REST API Example (Node.js)

```javascript
const axios = require('axios');

async function getDailySales() {
  const query = {
    measures: ['Orders.totalRevenue', 'Orders.completedOrdersCount'],
    timeDimensions: [{
      dimension: 'Orders.orderDate',
      granularity: 'day',
      dateRange: ['2023-06-01', '2023-06-30']
    }],
    segments: ['Orders.completedOrders']
  };

  try {
    const response = await axios.post('http://localhost:4000/cubejs-api/v1/query',
      { query },
      {
        headers: {
          'Authorization': 'Bearer your-api-token',
          'Content-Type': 'application/json'
        }
      }
    );

    console.log('Daily sales data:', response.data.data);
    return response.data.data;
  } catch (error) {
    console.error('Error fetching sales data:', error);
  }
}

getDailySales();
```

### Frontend Integration (React)

```jsx
import { useState, useEffect } from 'react';
import { CubeProvider, useCubeQuery } from '@cubejs-client/react';
import cubejs from '@cubejs-client/core';

const cubejsApi = cubejs('your-api-token', {
  apiUrl: 'http://localhost:4000/cubejs-api/v1'
});

function SalesDashboard() {
  const { resultSet, isLoading, error } = useCubeQuery({
    measures: ['Orders.totalRevenue', 'Orders.completedOrdersCount'],
    timeDimensions: [{
      dimension: 'Orders.orderDate',
      granularity: 'day',
      dateRange: ['2023-06-01', '2023-06-30']
    }],
    segments: ['Orders.completedOrders']
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.toString()}</div>;

  return (
    <div>
      <h2>Daily Sales Performance</h2>
      <table>
        <thead>
          <tr>
            <th>Date</th>
            <th>Revenue</th>
            <th>Orders</th>
          </tr>
        </thead>
        <tbody>
          {resultSet.tablePivot().map((row, index) => (
            <tr key={index}>
              <td>{row['Orders.orderDate']}</td>
              <td>${row['Orders.totalRevenue']}</td>
              <td>{row['Orders.completedOrdersCount']}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

function App() {
  return (
    <CubeProvider cubejsApi={cubejsApi}>
      <SalesDashboard />
    </CubeProvider>
  );
}
```

## Step 6: Production Deployment

### Build for Production

```bash
npm run build
```

### Environment Variables (Production)

```bash
CUBEJS_DB_TYPE=postgres
CUBEJS_DB_HOST=your-prod-db-host
CUBEJS_DB_NAME=ecommerce_prod
CUBEJS_DB_USER=cube_user
CUBEJS_DB_PASS=secure-password
CUBEJS_API_SECRET=production-secret-key
CUBEJS_DEV_MODE=false

# Optional: External database for pre-aggregations
CUBEJS_PRE_AGGREGATIONS_SCHEMA=pre_aggregations
CUBEJS_CACHE_AND_QUEUE_DRIVER=redis
CUBEJS_REDIS_URL=redis://localhost:6379
```

## Common Workflow Summary

1. **Setup**: Initialize project and configure database connection
2. **Schema Design**: Define cubes with measures, dimensions, and relationships
3. **Development**: Use playground to build and test queries
4. **Optimization**: Add pre-aggregations for frequently used query patterns
5. **Integration**: Connect frontend applications via REST or GraphQL APIs
6. **Production**: Deploy with proper environment configuration and caching

## Advanced Use Cases & Semantic Runtime Capabilities

### Multi-Warehouse Data Federation

#### Scenario: E-commerce with Separate Analytics and Operational Databases

```javascript
// cube.js - Multiple data source configuration
module.exports = {
  driverFactory: ({ dataSource }) => {
    if (dataSource === 'analytics') {
      return {
        type: 'bigquery',
        projectId: 'analytics-warehouse',
        keyFilename: '/path/to/analytics-key.json'
      };
    }

    if (dataSource === 'operational') {
      return {
        type: 'postgres',
        host: 'operational-db.company.com',
        database: 'ecommerce_live',
        username: process.env.OPERATIONAL_DB_USER,
        password: process.env.OPERATIONAL_DB_PASS
      };
    }

    if (dataSource === 'crm') {
      return {
        type: 'mysql',
        host: 'crm-db.company.com',
        database: 'salesforce_replica',
        username: process.env.CRM_DB_USER,
        password: process.env.CRM_DB_PASS
      };
    }
  }
};
```

#### Cross-Database Joins and Data Blending

```javascript
// schema/AnalyticsOrders.js - BigQuery analytics warehouse
cube(`AnalyticsOrders`, {
  dataSource: 'analytics',
  sql: `SELECT * FROM \`analytics-warehouse.ecommerce.orders_fact\``,

  measures: {
    totalRevenue: {
      sql: `revenue_amount`,
      type: `sum`,
      format: `currency`
    },

    // Advanced: ML-powered predictions from BigQuery ML
    predictedLTV: {
      sql: `ML.PREDICT(MODEL \`analytics-warehouse.ml.customer_ltv_model\`,
            (SELECT * FROM \`analytics-warehouse.ecommerce.customer_features\`
             WHERE customer_id = ${CUBE}.customer_id))`,
      type: `number`,
      format: `currency`
    }
  },

  dimensions: {
    orderId: {
      sql: `order_id`,
      type: `string`,
      primaryKey: true
    },

    customerId: {
      sql: `customer_id`,
      type: `string`
    },

    // Computed dimension using BigQuery functions
    revenueCategory: {
      sql: `CASE
        WHEN revenue_amount < 50 THEN 'Low Value'
        WHEN revenue_amount < 200 THEN 'Medium Value'
        ELSE 'High Value'
      END`,
      type: `string`
    }
  }
});

// schema/LiveInventory.js - Real-time operational database
cube(`LiveInventory`, {
  dataSource: 'operational',
  sql: `SELECT * FROM inventory WHERE updated_at > NOW() - INTERVAL '1 hour'`,

  measures: {
    currentStock: {
      sql: `quantity_available`,
      type: `sum`
    },

    lowStockItems: {
      sql: `product_id`,
      type: `countDistinct`,
      filters: [{
        sql: `${CUBE}.quantity_available < ${CUBE}.reorder_threshold`
      }]
    }
  },

  dimensions: {
    productId: {
      sql: `product_id`,
      type: `string`,
      primaryKey: true
    },

    warehouseLocation: {
      sql: `warehouse_code`,
      type: `string`
    }
  },

  // Real-time refresh for operational data
  refreshKey: {
    every: `30 seconds`
  }
});

// schema/CRMCustomers.js - Customer relationship data
cube(`CRMCustomers`, {
  dataSource: 'crm',
  sql: `SELECT * FROM customer_profiles`,

  measures: {
    customerCount: {
      type: `count`
    },

    avgCustomerScore: {
      sql: `customer_score`,
      type: `avg`
    }
  },

  dimensions: {
    customerId: {
      sql: `external_customer_id`,
      type: `string`,
      primaryKey: true
    },

    segment: {
      sql: `customer_segment`,
      type: `string`
    },

    acquisitionChannel: {
      sql: `acquisition_channel`,
      type: `string`
    }
  }
});

// schema/UnifiedCustomerView.js - Cross-database joined cube
cube(`UnifiedCustomerView`, {
  sql: `
    SELECT
      c.external_customer_id as customer_id,
      c.customer_segment,
      c.acquisition_channel,
      o.total_orders,
      o.lifetime_revenue
    FROM ${CRMCustomers.sql()} c
    LEFT JOIN (
      SELECT
        customer_id,
        COUNT(*) as total_orders,
        SUM(revenue_amount) as lifetime_revenue
      FROM ${AnalyticsOrders.sql()}
      GROUP BY customer_id
    ) o ON c.external_customer_id = o.customer_id
  `,

  joins: {
    AnalyticsOrders: {
      sql: `${CUBE}.customer_id = ${AnalyticsOrders}.customerId`,
      relationship: `hasMany`
    },

    CRMCustomers: {
      sql: `${CUBE}.customer_id = ${CRMCustomers}.customerId`,
      relationship: `belongsTo`
    }
  },

  measures: {
    customerLTV: {
      sql: `lifetime_revenue`,
      type: `avg`,
      format: `currency`
    },

    crossSellOpportunities: {
      sql: `CASE WHEN total_orders > 5 AND customer_segment = 'Premium' THEN 1 ELSE 0 END`,
      type: `sum`
    }
  }
});
```

### Advanced Semantic Layer Patterns

#### Schema Versioning and Governance

```javascript
// schema/versioned/Orders_v2.js
cube(`Orders`, {
  sql: `SELECT * FROM orders`,

  // Schema metadata for governance
  meta: {
    version: '2.0.0',
    owner: 'analytics-team@company.com',
    description: 'Core orders cube with enhanced revenue tracking',
    deprecated: false,
    tags: ['core', 'revenue', 'transactions']
  },

  measures: {
    // Legacy measure with deprecation warning
    revenue: {
      sql: `total_amount`,
      type: `sum`,
      meta: {
        deprecated: true,
        deprecationReason: 'Use totalRevenue instead for consistency',
        migrationPath: 'Replace Orders.revenue with Orders.totalRevenue'
      }
    },

    // New standardized measure
    totalRevenue: {
      sql: `total_amount`,
      type: `sum`,
      format: `currency`,
      meta: {
        description: 'Total revenue including taxes and fees',
        businessDefinition: 'Sum of all completed order amounts in USD'
      }
    }
  }
});

// schema/metrics/MetricRegistry.js - Centralized metric definitions
const CoreMetrics = {
  REVENUE_METRICS: {
    totalRevenue: {
      sql: `total_amount`,
      type: `sum`,
      format: `currency`
    },

    revenueGrowthRate: {
      sql: `
        (${totalRevenue} - LAG(${totalRevenue}) OVER (ORDER BY ${timeDimension}))
        / LAG(${totalRevenue}) OVER (ORDER BY ${timeDimension}) * 100
      `,
      type: `number`,
      format: `percent`
    }
  },

  CUSTOMER_METRICS: {
    customerAcquisitionCost: {
      sql: `${marketingSpend} / ${newCustomers}`,
      type: `number`,
      format: `currency`
    },

    customerLifetimeValue: {
      sql: `${totalRevenue} / ${uniqueCustomers}`,
      type: `number`,
      format: `currency`
    }
  }
};
```

#### Advanced Data Transformations and Custom SQL

```javascript
// schema/advanced/CustomerCohorts.js
cube(`CustomerCohorts`, {
  sql: `
    WITH customer_cohorts AS (
      SELECT
        user_id,
        DATE_TRUNC('month', MIN(order_date)) as cohort_month,
        DATE_TRUNC('month', order_date) as order_month,
        SUM(total_amount) as revenue
      FROM orders
      WHERE status = 'completed'
      GROUP BY user_id, DATE_TRUNC('month', order_date)
    ),
    cohort_analysis AS (
      SELECT
        cohort_month,
        order_month,
        EXTRACT(MONTH FROM AGE(order_month, cohort_month)) as period_number,
        COUNT(DISTINCT user_id) as customers,
        SUM(revenue) as revenue
      FROM customer_cohorts
      GROUP BY cohort_month, order_month
    )
    SELECT * FROM cohort_analysis
  `,

  measures: {
    cohortSize: {
      sql: `customers`,
      type: `sum`
    },

    retentionRate: {
      sql: `
        ${cohortSize} / FIRST_VALUE(${cohortSize})
        OVER (PARTITION BY cohort_month ORDER BY period_number) * 100
      `,
      type: `number`,
      format: `percent`
    },

    cohortRevenue: {
      sql: `revenue`,
      type: `sum`,
      format: `currency`
    }
  },

  dimensions: {
    cohortMonth: {
      sql: `cohort_month`,
      type: `time`
    },

    periodNumber: {
      sql: `period_number`,
      type: `number`
    }
  }
});

// schema/advanced/RealTimeMetrics.js - Streaming data integration
cube(`RealTimeMetrics`, {
  sql: `
    SELECT * FROM (
      -- Batch data from warehouse
      SELECT
        'batch' as data_source,
        order_id,
        revenue,
        order_timestamp
      FROM warehouse.orders
      WHERE order_timestamp < CURRENT_TIMESTAMP - INTERVAL '5 minutes'

      UNION ALL

      -- Real-time data from Kafka/streaming
      SELECT
        'realtime' as data_source,
        order_id,
        revenue,
        order_timestamp
      FROM streaming.live_orders
      WHERE order_timestamp >= CURRENT_TIMESTAMP - INTERVAL '5 minutes'
    )
  `,

  measures: {
    realtimeRevenue: {
      sql: `revenue`,
      type: `sum`,
      format: `currency`,
      filters: [{
        sql: `${CUBE}.data_source = 'realtime'`
      }]
    },

    totalRevenue: {
      sql: `revenue`,
      type: `sum`,
      format: `currency`
    }
  },

  // Extremely frequent refresh for real-time data
  refreshKey: {
    every: `10 seconds`
  }
});
```

### Enterprise Security and Multi-Tenancy

#### Row-Level Security (RLS)

```javascript
// cube.js - Security context configuration
module.exports = {
  contextToAppId: ({ securityContext }) => {
    return `CUBEJS_APP_${securityContext.tenantId}_${securityContext.userRole}`;
  },

  // Dynamic schema based on user context
  repositoryFactory: ({ securityContext }) => {
    return {
      dataSchemaFiles: () => {
        const baseSchemas = ['Users.js', 'Orders.js'];

        // Add admin-only schemas
        if (securityContext.userRole === 'admin') {
          baseSchemas.push('AdminMetrics.js', 'FinancialReports.js');
        }

        // Add tenant-specific schemas
        if (securityContext.tenantId) {
          baseSchemas.push(`tenant/${securityContext.tenantId}/CustomMetrics.js`);
        }

        return baseSchemas;
      }
    };
  }
};

// schema/SecureOrders.js
cube(`SecureOrders`, {
  sql: `
    SELECT * FROM orders
    WHERE tenant_id = '${SECURITY_CONTEXT.tenantId}'
    ${SECURITY_CONTEXT.userRole !== 'admin' ?
      "AND user_id = '" + SECURITY_CONTEXT.userId + "'" : ''
    }
  `,

  measures: {
    revenue: {
      sql: `total_amount`,
      type: `sum`,
      // Field-level security
      shown: ({ securityContext }) => {
        return ['admin', 'finance'].includes(securityContext.userRole);
      }
    }
  }
});
```

#### Advanced API Capabilities

#### Custom REST Endpoints

```javascript
// cube.js - Custom API endpoints
module.exports = {
  extendContext: (req) => {
    return {
      securityContext: {
        userId: req.headers['x-user-id'],
        tenantId: req.headers['x-tenant-id'],
        userRole: req.headers['x-user-role']
      }
    };
  },

  // Custom REST endpoints
  apiExtensions: {
    '/health': async (req, res) => {
      const health = await checkSystemHealth();
      res.json(health);
    },

    '/metrics/kpis': async (req, res, { cubejsApi }) => {
      const kpis = await cubejsApi.load({
        measures: [
          'Orders.totalRevenue',
          'Users.count',
          'Products.count'
        ],
        timeDimensions: [{
          dimension: 'Orders.orderDate',
          granularity: 'day',
          dateRange: 'last 30 days'
        }]
      });

      res.json({
        totalRevenue: kpis.annotation.measures['Orders.totalRevenue'],
        totalUsers: kpis.annotation.measures['Users.count'],
        totalProducts: kpis.annotation.measures['Products.count']
      });
    }
  }
};
```

#### GraphQL Schema Extensions

```javascript
// GraphQL custom resolvers
const resolvers = {
  Query: {
    customerInsights: async (parent, { customerId }, { cubejsApi }) => {
      const [orderHistory, cohortData, predictions] = await Promise.all([
        cubejsApi.load({
          measures: ['Orders.count', 'Orders.totalRevenue'],
          dimensions: ['Orders.status'],
          filters: [{ member: 'Orders.customerId', operator: 'equals', values: [customerId] }]
        }),

        cubejsApi.load({
          measures: ['CustomerCohorts.retentionRate'],
          dimensions: ['CustomerCohorts.periodNumber'],
          filters: [{ member: 'CustomerCohorts.customerId', operator: 'equals', values: [customerId] }]
        }),

        // External ML prediction service
        getPredictions(customerId)
      ]);

      return {
        orderHistory: orderHistory.tablePivot(),
        retentionProfile: cohortData.tablePivot(),
        predictions
      };
    }
  }
};
```

### Performance and Monitoring

#### Advanced Pre-aggregation Strategies

```javascript
// schema/OptimizedOrders.js
cube(`OptimizedOrders`, {
  sql: `SELECT * FROM orders`,

  preAggregations: {
    // Partitioned by time for large datasets
    monthlyRollup: {
      type: `rollup`,
      measures: [totalRevenue, count],
      dimensions: [status, productCategory],
      timeDimension: orderDate,
      granularity: `day`,
      partitionGranularity: `month`,
      refreshKey: {
        every: `1 hour`,
        incremental: true,
        updateWindow: `7 day`
      },
      // Use external database for pre-aggregations
      external: true
    },

    // Lambda architecture: real-time + batch
    realtimeSummary: {
      type: `rollup`,
      measures: [totalRevenue, count],
      timeDimension: orderDate,
      granularity: `hour`,
      refreshKey: {
        every: `1 minute`
      },
      // Only last 24 hours for real-time
      buildRangeStart: {
        sql: `CURRENT_TIMESTAMP - INTERVAL '24 hours'`
      }
    },

    // Machine learning features
    customerFeatures: {
      type: `originalSql`,
      sql: `
        SELECT
          customer_id,
          COUNT(*) as order_count,
          AVG(total_amount) as avg_order_value,
          STDDEV(total_amount) as order_value_variance,
          DATE_DIFF('day', MIN(order_date), MAX(order_date)) as customer_lifespan_days
        FROM ${CUBE.sql()}
        GROUP BY customer_id
      `,
      refreshKey: {
        every: `1 day`
      }
    }
  }
});
```

#### Monitoring and Observability

```javascript
// cube.js - Built-in monitoring
module.exports = {
  // Query logging and metrics
  logger: (msg, params) => {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: params.level,
      message: msg,
      duration: params.duration,
      query: params.query,
      userId: params.securityContext?.userId,
      tenantId: params.securityContext?.tenantId
    }));
  },

  // Performance monitoring
  telemetry: {
    enabled: true,
    endpoint: 'https://your-monitoring-endpoint.com/metrics'
  },

  // Query result caching strategy
  cacheAndQueueDriver: 'redis',
  redisPool: {
    poolMin: 2,
    poolMax: 10,
    acquireTimeoutMillis: 30000,
    idleTimeoutMillis: 30000
  }
};
```

## Semantic Runtime Benefits

### 1. **Unified Data Model**

- Single source of truth across all data sources
- Consistent metric definitions and business logic
- Version control and governance for schemas

### 2. **API-First Architecture**

- REST, GraphQL, and SQL endpoints from same schema
- Auto-generated documentation and type definitions
- Client SDKs for multiple languages

### 3. **Performance Optimization**

- Intelligent caching and pre-aggregations
- Query optimization and push-down
- Real-time and batch data federation

### 4. **Enterprise Security**

- Row-level and column-level security
- Multi-tenant data isolation
- RBAC and custom authorization

### 5. **Developer Experience**

- Type-safe queries and results
- Real-time schema validation
- Built-in testing and debugging tools

### 6. **Operational Excellence**

- Monitoring and alerting
- Performance analytics
- Schema impact analysis

## Key Benefits Realized

- **Performance**: Pre-aggregations make complex queries sub-second
- **Consistency**: Single source of truth for business metrics
- **Flexibility**: Easy to add new metrics without changing frontend code
- **Scalability**: Handles growing data volumes with smart caching
- **Developer Experience**: Type-safe APIs and intuitive query building
- **Enterprise Ready**: Multi-tenancy, security, and governance built-in
- **Data Federation**: Query across multiple databases as single logical model
- **Real-time Analytics**: Mix streaming and batch data seamlessly
