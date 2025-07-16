# Cube.dev Example Workflow: Real E-commerce Analytics

## The Business Problem

Using the **[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)** from Kaggle, we'll build analytics to track:

- **Order volume and revenue trends over time**
- **Product performance by category and seller**
- **Customer satisfaction and review patterns**
- **Delivery performance and logistics insights**
- **Geographic sales distribution across Brazil**

> **📝 Learning Note**: While Cube.dev provides a powerful web UI (Developer Playground) that can generate schemas, build queries visually, and manage configurations, this tutorial focuses on **file-based definitions and actual code content**. Understanding the underlying schema files, configuration syntax, and API structures is crucial for:
>
> - **Production deployments** where everything should be version-controlled
> - **Advanced customizations** that go beyond GUI capabilities
> - **Debugging and troubleshooting** when things don't work as expected
> - **Team collaboration** through code reviews and Git workflows
>
> I'll show you what's actually happening "under the hood" so you can take a glimpse into the Cube.dev architecture and how to effectively use it in real-world scenarios.

## Real Dataset: Brazilian E-Commerce (Olist)

### Download and Setup

```bash
# Download from Kaggle
kaggle datasets download -d olistbr/brazilian-ecommerce

# Extract the dataset
unzip brazilian-ecommerce.zip

# Files included:
# - olist_customers_dataset.csv
# - olist_orders_dataset.csv
# - olist_order_items_dataset.csv
# - olist_products_dataset.csv
# - olist_sellers_dataset.csv
# - olist_order_reviews_dataset.csv
# - olist_order_payments_dataset.csv
# - product_category_name_translation.csv
```

### Database Schema (PostgreSQL)

```sql
-- Customers table
CREATE TABLE customers (
    customer_id VARCHAR(32) PRIMARY KEY,
    customer_unique_id VARCHAR(32),
    customer_zip_code_prefix VARCHAR(5),
    customer_city VARCHAR(100),
    customer_state VARCHAR(2)
);

-- Orders table
CREATE TABLE orders (
    order_id VARCHAR(32) PRIMARY KEY,
    customer_id VARCHAR(32) REFERENCES customers(customer_id),
    order_status VARCHAR(20),
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP
);

-- Products table
CREATE TABLE products (
    product_id VARCHAR(32) PRIMARY KEY,
    product_category_name VARCHAR(100),
    product_name_lenght INTEGER,
    product_description_lenght INTEGER,
    product_photos_qty INTEGER,
    product_weight_g INTEGER,
    product_length_cm INTEGER,
    product_height_cm INTEGER,
    product_width_cm INTEGER
);

-- Order items table
CREATE TABLE order_items (
    order_id VARCHAR(32) REFERENCES orders(order_id),
    order_item_id INTEGER,
    product_id VARCHAR(32) REFERENCES products(product_id),
    seller_id VARCHAR(32),
    shipping_limit_date TIMESTAMP,
    price DECIMAL(10,2),
    freight_value DECIMAL(10,2),
    PRIMARY KEY (order_id, order_item_id)
);

-- Sellers table
CREATE TABLE sellers (
    seller_id VARCHAR(32) PRIMARY KEY,
    seller_zip_code_prefix VARCHAR(5),
    seller_city VARCHAR(100),
    seller_state VARCHAR(2)
);

-- Order reviews table
CREATE TABLE order_reviews (
    review_id VARCHAR(32) PRIMARY KEY,
    order_id VARCHAR(32) REFERENCES orders(order_id),
    review_score INTEGER,
    review_comment_title TEXT,
    review_comment_message TEXT,
    review_creation_date TIMESTAMP,
    review_answer_timestamp TIMESTAMP
);

-- Order payments table
CREATE TABLE order_payments (
    order_id VARCHAR(32) REFERENCES orders(order_id),
    payment_sequential INTEGER,
    payment_type VARCHAR(20),
    payment_installments INTEGER,
    payment_value DECIMAL(10,2),
    PRIMARY KEY (order_id, payment_sequential)
);

-- Product categories translation
CREATE TABLE product_categories (
    product_category_name VARCHAR(100) PRIMARY KEY,
    product_category_name_english VARCHAR(100)
);
```

### Data Loading Example

```python
# load_data.py - Script to load CSV data into PostgreSQL
import pandas as pd
import psycopg2
from sqlalchemy import create_engine

# Database connection
engine = create_engine('postgresql://user:password@localhost/olist_ecommerce')

# Load CSV files
customers_df = pd.read_csv('olist_customers_dataset.csv')
orders_df = pd.read_csv('olist_orders_dataset.csv')
products_df = pd.read_csv('olist_products_dataset.csv')
order_items_df = pd.read_csv('olist_order_items_dataset.csv')
sellers_df = pd.read_csv('olist_sellers_dataset.csv')
reviews_df = pd.read_csv('olist_order_reviews_dataset.csv')
payments_df = pd.read_csv('olist_order_payments_dataset.csv')
categories_df = pd.read_csv('product_category_name_translation.csv')

# Load into database
customers_df.to_sql('customers', engine, if_exists='replace', index=False)
orders_df.to_sql('orders', engine, if_exists='replace', index=False)
products_df.to_sql('products', engine, if_exists='replace', index=False)
order_items_df.to_sql('order_items', engine, if_exists='replace', index=False)
sellers_df.to_sql('sellers', engine, if_exists='replace', index=False)
reviews_df.to_sql('order_reviews', engine, if_exists='replace', index=False)
payments_df.to_sql('order_payments', engine, if_exists='replace', index=False)
categories_df.to_sql('product_categories', engine, if_exists='replace', index=False)

print("Data loaded successfully!")
```

## Step 1: Project Setup

### Initialize Cube.dev Project

```bash
npx cubejs-cli create olist-analytics -d postgres
cd olist-analytics
```

### Environment Configuration

```bash
# .env file
CUBEJS_DB_TYPE=postgres
CUBEJS_DB_HOST=localhost
CUBEJS_DB_NAME=olist_ecommerce
CUBEJS_DB_USER=postgres
CUBEJS_DB_PASS=password
CUBEJS_DB_PORT=5432
CUBEJS_API_SECRET=your-secret-key-here
CUBEJS_DEV_MODE=true
```

## Step 2: Define Data Schema

### model/cubes/customers.yml

```yaml
cubes:
  - name: customers
    sql_table: customers

    measures:
      - name: count
        type: count
        title: Total Customers

      - name: unique_customers
        sql: customer_unique_id
        type: count_distinct
        title: Unique Customers

    dimensions:
      - name: customer_id
        sql: customer_id
        type: string
        primary_key: true

      - name: customer_unique_id
        sql: customer_unique_id
        type: string

      - name: city
        sql: customer_city
        type: string
        title: Customer City

      - name: state
        sql: customer_state
        type: string
        title: Customer State

      - name: zip_code
        sql: customer_zip_code_prefix
        type: string
        title: ZIP Code

    segments:
      - name: southeast_region
        sql: "{CUBE}.customer_state IN ('SP', 'RJ', 'MG', 'ES')"
        title: Southeast Brazil Customers

      - name: south_region
        sql: "{CUBE}.customer_state IN ('RS', 'SC', 'PR')"
        title: South Brazil Customers
```

### model/cubes/orders.yml

```yaml
cubes:
  - name: orders
    sql_table: orders

    joins:
      - name: customers
        sql: "{CUBE}.customer_id = {customers.customer_id}"
        relationship: many_to_one

      - name: order_items
        sql: "{CUBE}.order_id = {order_items.order_id}"
        relationship: one_to_many

      - name: order_reviews
        sql: "{CUBE}.order_id = {order_reviews.order_id}"
        relationship: one_to_many

      - name: order_payments
        sql: "{CUBE}.order_id = {order_payments.order_id}"
        relationship: one_to_many

    measures:
      - name: count
        type: count
        title: Total Orders

      - name: delivered_orders
        sql: order_id
        type: count
        title: Delivered Orders
        filters:
          - sql: "{CUBE}.order_status = 'delivered'"

      - name: cancelled_orders
        sql: order_id
        type: count
        title: Cancelled Orders
        filters:
          - sql: "{CUBE}.order_status = 'canceled'"

      # Advanced: Delivery time analysis
      - name: avg_delivery_time
        sql: "EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp)) / 86400"
        type: avg
        title: Average Delivery Time (Days)
        filters:
          - sql: "{CUBE}.order_delivered_customer_date IS NOT NULL"

      # Advanced: Delivery performance
      - name: on_time_deliveries
        sql: order_id
        type: count
        title: On-Time Deliveries
        filters:
          - sql: "{CUBE}.order_delivered_customer_date <= {CUBE}.order_estimated_delivery_date"

      - name: delivery_performance_rate
        sql: "{on_time_deliveries} / NULLIF({delivered_orders}, 0) * 100"
        type: number
        title: Delivery Performance Rate (%)
        format: percent

    dimensions:
      - name: order_id
        sql: order_id
        type: string
        primary_key: true

      - name: customer_id
        sql: customer_id
        type: string

      - name: status
        sql: order_status
        type: string
        title: Order Status

      - name: purchase_timestamp
        sql: order_purchase_timestamp
        type: time
        title: Purchase Date

      - name: approved_at
        sql: order_approved_at
        type: time
        title: Approved Date

      - name: delivered_date
        sql: order_delivered_customer_date
        type: time
        title: Delivered Date

      - name: estimated_delivery_date
        sql: order_estimated_delivery_date
        type: time
        title: Est. Delivery Date

      # Computed dimension for delivery status
      - name: delivery_status
        sql: >
          CASE
            WHEN order_delivered_customer_date IS NULL THEN 'Not Delivered'
            WHEN order_delivered_customer_date <= order_estimated_delivery_date THEN 'On Time'
            ELSE 'Late'
          END
        type: string
        title: Delivery Status

    segments:
      - name: delivered_orders
        sql: "{CUBE}.order_status = 'delivered'"
        title: Delivered Orders Only

      - name: recent_orders
        sql: "{CUBE}.order_purchase_timestamp >= '2018-01-01'"
        title: Recent Orders (2018+)
```

### model/cubes/products.yml

```yaml
cubes:
  - name: products
    sql: >
      SELECT
        p.*,
        pc.product_category_name_english as category_english
      FROM products p
      LEFT JOIN product_categories pc ON p.product_category_name = pc.product_category_name

    measures:
      - name: count
        type: count
        title: Total Products

      - name: avg_weight
        sql: product_weight_g
        type: avg
        title: Average Weight (g)

      - name: avg_photos
        sql: product_photos_qty
        type: avg
        title: Average Photos Count

    dimensions:
      - name: product_id
        sql: product_id
        type: string
        primary_key: true

      - name: category_name
        sql: product_category_name
        type: string
        title: Category (Portuguese)

      - name: category_english
        sql: category_english
        type: string
        title: Category (English)

      - name: name_length
        sql: product_name_lenght
        type: number
        title: Name Length

      - name: description_length
        sql: product_description_lenght
        type: number
        title: Description Length

      - name: photos_qty
        sql: product_photos_qty
        type: number
        title: Photos Count

      - name: weight_g
        sql: product_weight_g
        type: number
        title: Weight (g)

      # Computed dimensions
      - name: size_category
        sql: >
          CASE
            WHEN product_weight_g < 100 THEN 'Small'
            WHEN product_weight_g < 1000 THEN 'Medium'
            WHEN product_weight_g < 5000 THEN 'Large'
            ELSE 'Extra Large'
          END
        type: string
        title: Size Category

      - name: photo_quality
        sql: >
          CASE
            WHEN product_photos_qty = 0 THEN 'No Photos'
            WHEN product_photos_qty <= 2 THEN 'Few Photos'
            WHEN product_photos_qty <= 5 THEN 'Good Photos'
            ELSE 'Many Photos'
          END
        type: string
        title: Photo Quality
```

### model/cubes/order_items.yml

```yaml
cubes:
  - name: order_items
    sql_table: order_items

    joins:
      - name: orders
        sql: "{CUBE}.order_id = {orders.order_id}"
        relationship: many_to_one

      - name: products
        sql: "{CUBE}.product_id = {products.product_id}"
        relationship: many_to_one

      - name: sellers
        sql: "{CUBE}.seller_id = {sellers.seller_id}"
        relationship: many_to_one

    measures:
      - name: count
        type: count
        title: Total Order Items

      - name: total_revenue
        sql: price
        type: sum
        title: Total Revenue
        format: currency

      - name: total_freight
        sql: freight_value
        type: sum
        title: Total Freight
        format: currency

      - name: avg_price
        sql: price
        type: avg
        title: Average Price
        format: currency

      - name: avg_freight
        sql: freight_value
        type: avg
        title: Average Freight
        format: currency

      # Advanced: Profitability analysis
      - name: total_value
        sql: "price + freight_value"
        type: sum
        title: Total Value (Price + Freight)
        format: currency

      - name: freight_percentage
        sql: "{total_freight} / NULLIF({total_revenue}, 0) * 100"
        type: number
        title: Freight as % of Revenue
        format: percent

    dimensions:
      - name: order_id
        sql: order_id
        type: string

      - name: order_item_id
        sql: order_item_id
        type: number

      - name: product_id
        sql: product_id
        type: string

      - name: seller_id
        sql: seller_id
        type: string

      - name: price
        sql: price
        type: number
        format: currency

      - name: freight_value
        sql: freight_value
        type: number
        format: currency

      - name: shipping_limit_date
        sql: shipping_limit_date
        type: time
        title: Shipping Limit

      # Computed dimensions
      - name: price_range
        sql: >
          CASE
            WHEN price < 25 THEN 'Low ($0-25)'
            WHEN price < 100 THEN 'Medium ($25-100)'
            WHEN price < 300 THEN 'High ($100-300)'
            ELSE 'Premium ($300+)'
          END
        type: string
        title: Price Range

    segments:
      - name: high_value_items
        sql: "{CUBE}.price > 100"
        title: High Value Items ($100+)

      - name: low_freight_items
        sql: "{CUBE}.freight_value < 10"
        title: Low Freight Items (<$10)

    pre_aggregations:
      - name: daily_revenue
        measures:
          - total_revenue
          - count
        dimensions:
          - price_range
        time_dimension: orders.purchase_timestamp
        granularity: day
        refresh_key:
          every: 1 hour
```

### model/cubes/order_reviews.yml

```yaml
cubes:
  - name: order_reviews
    sql_table: order_reviews

    joins:
      - name: orders
        sql: "{CUBE}.order_id = {orders.order_id}"
        relationship: many_to_one

    measures:
      - name: count
        type: count
        title: Total Reviews

      - name: avg_review_score
        sql: review_score
        type: avg
        title: Average Review Score

      - name: positive_reviews
        sql: review_id
        type: count
        title: Positive Reviews (4-5 stars)
        filters:
          - sql: "{CUBE}.review_score >= 4"

      - name: negative_reviews
        sql: review_id
        type: count
        title: Negative Reviews (1-2 stars)
        filters:
          - sql: "{CUBE}.review_score <= 2"

      # Advanced: Customer satisfaction metrics
      - name: satisfaction_rate
        sql: "{positive_reviews} / NULLIF({count}, 0) * 100"
        type: number
        title: Customer Satisfaction Rate (%)
        format: percent

      - name: reviews_with_comments
        sql: review_id
        type: count
        title: Reviews with Comments
        filters:
          - sql: "{CUBE}.review_comment_message IS NOT NULL AND {CUBE}.review_comment_message != ''"

    dimensions:
      - name: review_id
        sql: review_id
        type: string
        primary_key: true

      - name: order_id
        sql: order_id
        type: string

      - name: review_score
        sql: review_score
        type: number
        title: Review Score

      - name: review_creation_date
        sql: review_creation_date
        type: time
        title: Review Date

      # Computed dimensions
      - name: review_category
        sql: >
          CASE
            WHEN review_score = 5 THEN 'Excellent'
            WHEN review_score = 4 THEN 'Good'
            WHEN review_score = 3 THEN 'Average'
            WHEN review_score = 2 THEN 'Poor'
            ELSE 'Very Poor'
          END
        type: string
        title: Review Category

      - name: has_comment
        sql: >
          CASE
            WHEN review_comment_message IS NOT NULL AND review_comment_message != '' THEN 'Yes'
            ELSE 'No'
          END
        type: string
        title: Has Comment

    segments:
      - name: recent_reviews
        sql: "{CUBE}.review_creation_date >= '2018-01-01'"
        title: Recent Reviews (2018+)
```

### model/cubes/sellers.yml

```yaml
cubes:
  - name: sellers
    sql_table: sellers

    measures:
      - name: count
        type: count
        title: Total Sellers

    dimensions:
      - name: seller_id
        sql: seller_id
        type: string
        primary_key: true

      - name: city
        sql: seller_city
        type: string
        title: Seller City

      - name: state
        sql: seller_state
        type: string
        title: Seller State

      - name: zip_code
        sql: seller_zip_code_prefix
        type: string
        title: ZIP Code

    segments:
      - name: southeast_sellers
        sql: "{CUBE}.seller_state IN ('SP', 'RJ', 'MG', 'ES')"
        title: Southeast Brazil Sellers
```

### model/views/ecommerce_overview.yml

```yaml
views:
  - name: ecommerce_overview
    title: E-commerce Overview Dashboard
    description: Complete Brazilian e-commerce analytics from Olist dataset

    cubes:
      # Orders metrics
      - join_path: orders
        includes:
          - count
          - delivered_orders
          - avg_delivery_time
          - delivery_performance_rate
          - status
          - purchase_timestamp
          - delivery_status

      # Order items for revenue
      - join_path: orders.order_items
        prefix: true
        includes:
          - total_revenue
          - total_freight
          - avg_price
          - count
          - price_range

      # Product information
      - join_path: orders.order_items.products
        prefix: true
        includes:
          - category_english
          - size_category
          - photo_quality

      # Customer data
      - join_path: orders.customers
        prefix: true
        includes:
          - city
          - state
          - unique_customers

      # Seller information
      - join_path: orders.order_items.sellers
        prefix: true
        includes:
          - city
          - state

      # Review data
      - join_path: orders.order_reviews
        prefix: true
        includes:
          - avg_review_score
          - satisfaction_rate
          - review_category
```

### model/views/product_analytics.yml

```yaml
views:
  - name: product_analytics
    title: Product Performance Analytics
    description: Deep dive into product categories and performance metrics

    cubes:
      # Product core data
      - join_path: products
        includes:
          - count
          - category_english
          - size_category
          - photo_quality
          - avg_weight

      # Sales performance via order items
      - join_path: order_items
        prefix: true
        includes:
          - total_revenue
          - count
          - avg_price

        # Only include delivered orders
        filters:
          - member: orders.status
            operator: equals
            values: ['delivered']

      # Review data for products
      - join_path: order_items.orders.order_reviews
        prefix: true
        includes:
          - avg_review_score
          - satisfaction_rate
```

## Step 3: Start Development Server

```bash
npm run dev
```

This starts:

- **Cube.dev API** at <http://localhost:4000>
- **Developer Playground** at <http://localhost:4000/#/build>

## Step 4: Build Queries in Playground

### Query 1: Daily Sales Performance & Trends

```json
{
  "measures": [
    "order_items.total_revenue",
    "orders.count",
    "order_items.avg_price"
  ],
  "timeDimensions": [{
    "dimension": "orders.purchase_timestamp",
    "granularity": "day",
    "dateRange": ["2017-01-01", "2018-08-31"]
  }],
  "segments": ["orders.delivered_orders"],
  "order": {
    "orders.purchase_timestamp": "desc"
  }
}
```

**Expected Business Insights:**

- Daily revenue trends over the dataset period
- Seasonal patterns in Brazilian e-commerce
- Order volume correlation with revenue

### Query 2: Product Category Performance Analysis

```json
{
  "measures": [
    "order_items.total_revenue",
    "order_items.count",
    "order_items.avg_price",
    "products.count"
  ],
  "dimensions": [
    "products.category_english"
  ],
  "segments": ["orders.delivered_orders"],
  "order": {
    "order_items.total_revenue": "desc"
  },
  "limit": 20
}
```

**Expected Result Structure:**

```json
[
  {
    "products.category_english": "health_beauty",
    "order_items.total_revenue": "1,151,732.95",
    "order_items.count": "7,624",
    "order_items.avg_price": "65.82",
    "products.count": "1,284"
  },
  {
    "products.category_english": "computers_accessories",
    "order_items.total_revenue": "1,058,298.43",
    "order_items.count": "4,827",
    "order_items.avg_price": "89.34",
    "products.count": "853"
  }
]
```

### Query 3: Geographic Sales Analysis (Brazilian States)

```json
{
  "measures": [
    "order_items.total_revenue",
    "orders.count",
    "customers.unique_customers",
    "order_reviews.avg_review_score"
  ],
  "dimensions": [
    "customers.state"
  ],
  "segments": ["orders.delivered_orders"],
  "order": {
    "order_items.total_revenue": "desc"
  }
}
```

### Query 4: Customer Satisfaction & Delivery Performance

```json
{
  "measures": [
    "order_reviews.satisfaction_rate",
    "orders.delivery_performance_rate",
    "orders.avg_delivery_time",
    "order_reviews.count"
  ],
  "dimensions": [
    "order_reviews.review_category",
    "orders.delivery_status"
  ],
  "segments": ["orders.delivered_orders", "order_reviews.recent_reviews"]
}
```

### Query 5: Seller Performance Analysis

```json
{
  "measures": [
    "order_items.total_revenue",
    "order_items.count",
    "order_reviews.avg_review_score"
  ],
  "dimensions": [
    "sellers.state",
    "sellers.city"
  ],
  "filters": [{
    "member": "order_items.total_revenue",
    "operator": "gt",
    "values": ["10000"]
  }],
  "segments": ["orders.delivered_orders"],
  "order": {
    "order_items.total_revenue": "desc"
  },
  "limit": 50
}
```

### Query 6: Advanced Cohort Analysis (Monthly)

```json
{
  "measures": [
    "orders.count",
    "order_items.total_revenue",
    "customers.unique_customers"
  ],
  "timeDimensions": [{
    "dimension": "orders.purchase_timestamp",
    "granularity": "month",
    "dateRange": ["2017-01-01", "2018-08-31"]
  }],
  "dimensions": [
    "customers.state"
  ],
  "segments": ["orders.delivered_orders"]
}
```

### Query 7: Using Views - E-commerce Overview

```json
{
  "measures": [
    "ecommerce_overview.order_items_total_revenue",
    "ecommerce_overview.count",
    "ecommerce_overview.order_reviews_satisfaction_rate"
  ],
  "dimensions": [
    "ecommerce_overview.customers_state",
    "ecommerce_overview.products_category_english"
  ],
  "timeDimensions": [{
    "dimension": "ecommerce_overview.purchase_timestamp",
    "granularity": "month",
    "dateRange": ["2018-01-01", "2018-08-31"]
  }],
  "order": {
    "ecommerce_overview.order_items_total_revenue": "desc"
  }
}
```

## Step 5: API Integration

### REST API Example (Node.js) - Brazilian E-commerce Analytics

```javascript
const axios = require('axios');

async function getBrazilianEcommerceInsights() {
  const queries = {
    // Daily sales trends
    dailySales: {
      measures: ['order_items.total_revenue', 'orders.count'],
      timeDimensions: [{
        dimension: 'orders.purchase_timestamp',
        granularity: 'day',
        dateRange: ['2018-01-01', '2018-08-31']
      }],
      segments: ['orders.delivered_orders']
    },

    // Top categories by revenue
    topCategories: {
      measures: ['order_items.total_revenue', 'order_items.count'],
      dimensions: ['products.category_english'],
      segments: ['orders.delivered_orders'],
      order: { 'order_items.total_revenue': 'desc' },
      limit: 10
    },

    // Geographic performance
    statePerformance: {
      measures: [
        'order_items.total_revenue',
        'orders.count',
        'order_reviews.avg_review_score'
      ],
      dimensions: ['customers.state'],
      segments: ['orders.delivered_orders'],
      order: { 'order_items.total_revenue': 'desc' }
    },

    // Customer satisfaction metrics
    satisfaction: {
      measures: [
        'order_reviews.satisfaction_rate',
        'orders.delivery_performance_rate',
        'orders.avg_delivery_time'
      ],
      dimensions: ['order_reviews.review_category']
    }
  };

  try {
    const results = {};

    for (const [key, query] of Object.entries(queries)) {
      const response = await axios.post(
        'http://localhost:4000/cubejs-api/v1/query',
        { query },
        {
          headers: {
            'Authorization': 'Bearer your-api-token',
            'Content-Type': 'application/json'
          }
        }
      );
      results[key] = response.data.data;
    }

    // Business insights processing
    const insights = {
      totalRevenue: results.dailySales.reduce((sum, day) =>
        sum + parseFloat(day['order_items.total_revenue'] || 0), 0
      ),
      topCategory: results.topCategories[0]['products.category_english'],
      topState: results.statePerformance[0]['customers.state'],
      avgSatisfaction: results.satisfaction.find(r =>
        r['order_reviews.review_category'] === 'Excellent'
      )?.['order_reviews.satisfaction_rate'] || 0
    };

    console.log('🇧🇷 Brazilian E-commerce Insights:', insights);
    return { raw: results, insights };

  } catch (error) {
    console.error('Error fetching Brazilian e-commerce data:', error);
  }
}

// Advanced: Real-time dashboard data using views
async function getDashboardMetrics() {
  const query = {
    measures: [
      'ecommerce_overview.order_items_total_revenue',
      'ecommerce_overview.count',
      'ecommerce_overview.order_reviews_avg_review_score',
      'ecommerce_overview.delivery_performance_rate'
    ],
    timeDimensions: [{
      dimension: 'ecommerce_overview.purchase_timestamp',
      granularity: 'month',
      dateRange: 'last 12 months'
    }],
    segments: ['ecommerce_overview.delivered_orders']
  };

  const response = await axios.post('http://localhost:4000/cubejs-api/v1/query',
    { query },
    { headers: { 'Authorization': 'Bearer your-api-token' }}
  );

  return response.data.data;
}

getBrazilianEcommerceInsights();
```

### Frontend Integration (React) - Brazilian E-commerce Dashboard

```jsx
import { useState, useEffect } from 'react';
import { CubeProvider, useCubeQuery } from '@cubejs-client/react';
import cubejs from '@cubejs-client/core';

const cubejsApi = cubejs('your-api-token', {
  apiUrl: 'http://localhost:4000/cubejs-api/v1'
});

function BrazilianEcommerceDashboard() {
  // Daily sales performance using snake_case
  const { resultSet: salesData, isLoading: salesLoading } = useCubeQuery({
    measures: ['order_items.total_revenue', 'orders.count'],
    timeDimensions: [{
      dimension: 'orders.purchase_timestamp',
      granularity: 'month',
      dateRange: ['2017-01-01', '2018-08-31']
    }],
    segments: ['orders.delivered_orders']
  });

  // Category performance using corrected naming
  const { resultSet: categoryData, isLoading: categoryLoading } = useCubeQuery({
    measures: ['order_items.total_revenue', 'order_items.count'],
    dimensions: ['products.category_english'],
    segments: ['orders.delivered_orders'],
    order: { 'order_items.total_revenue': 'desc' },
    limit: 10
  });

  // Customer satisfaction by state using proper references
  const { resultSet: satisfactionData, isLoading: satisfactionLoading } = useCubeQuery({
    measures: [
      'order_reviews.avg_review_score',
      'orders.delivery_performance_rate',
      'order_items.total_revenue'
    ],
    dimensions: ['customers.state'],
    segments: ['orders.delivered_orders'],
    order: { 'order_items.total_revenue': 'desc' },
    limit: 10
  });

  if (salesLoading || categoryLoading || satisfactionLoading) {
    return <div className="loading">Loading Brazilian E-commerce Analytics...</div>;
  }

  return (
    <div className="dashboard">
      <header>
        <h1>🇧🇷 Brazilian E-commerce Analytics Dashboard</h1>
        <p>Insights from Olist Dataset (2017-2018)</p>
      </header>

      {/* Monthly Sales Trends */}
      <section className="sales-trends">
        <h2>📈 Monthly Sales Performance</h2>
        <div className="chart-container">
          <table>
            <thead>
              <tr>
                <th>Month</th>
                <th>Revenue (R$)</th>
                <th>Orders</th>
                <th>Avg Order Value</th>
              </tr>
            </thead>
            <tbody>
              {salesData?.tablePivot().map((row, index) => {
                const revenue = parseFloat(row['order_items.total_revenue'] || 0);
                const orders = parseInt(row['orders.count'] || 0);
                const avgOrderValue = orders > 0 ? revenue / orders : 0;

                return (
                  <tr key={index}>
                    <td>{new Date(row['orders.purchase_timestamp']).toLocaleDateString('pt-BR', { year: 'numeric', month: 'short' })}</td>
                    <td>R$ {revenue.toLocaleString('pt-BR', { minimumFractionDigits: 2 })}</td>
                    <td>{orders.toLocaleString()}</td>
                    <td>R$ {avgOrderValue.toLocaleString('pt-BR', { minimumFractionDigits: 2 })}</td>
                  </tr>
                );
              })}
            </tbody>
          </table>
        </div>
      </section>

      {/* Top Categories */}
      <section className="category-performance">
        <h2>🛍️ Top Product Categories</h2>
        <div className="category-grid">
          {categoryData?.tablePivot().map((category, index) => (
            <div key={index} className="category-card">
              <h3>#{index + 1} {category['products.category_english']?.replace(/_/g, ' ').toUpperCase()}</h3>
              <div className="metrics">
                <p><strong>Revenue:</strong> R$ {parseFloat(category['order_items.total_revenue'] || 0).toLocaleString('pt-BR')}</p>
                <p><strong>Items Sold:</strong> {parseInt(category['order_items.count'] || 0).toLocaleString()}</p>
                <p><strong>Avg Price:</strong> R$ {(parseFloat(category['order_items.total_revenue'] || 0) / parseInt(category['order_items.count'] || 1)).toFixed(2)}</p>
              </div>
            </div>
          ))}
        </div>
      </section>

      {/* Geographic Performance */}
      <section className="geographic-analysis">
        <h2>🗺️ Performance by Brazilian State</h2>
        <table>
          <thead>
            <tr>
              <th>State</th>
              <th>Revenue (R$)</th>
              <th>Avg Review Score</th>
              <th>Delivery Performance</th>
            </tr>
          </thead>
          <tbody>
            {satisfactionData?.tablePivot().map((state, index) => (
              <tr key={index}>
                <td><strong>{state['customers.state']}</strong></td>
                <td>R$ {parseFloat(state['order_items.total_revenue'] || 0).toLocaleString('pt-BR')}</td>
                <td>⭐ {parseFloat(state['order_reviews.avg_review_score'] || 0).toFixed(1)}/5</td>
                <td>{parseFloat(state['orders.delivery_performance_rate'] || 0).toFixed(1)}%</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
}

// Component using Views for cleaner queries
function EcommerceOverviewDashboard() {
  const { resultSet, isLoading, error } = useCubeQuery({
    measures: [
      'ecommerce_overview.order_items_total_revenue',
      'ecommerce_overview.count',
      'ecommerce_overview.order_reviews_satisfaction_rate'
    ],
    dimensions: [
      'ecommerce_overview.customers_state',
      'ecommerce_overview.products_category_english'
    ],
    timeDimensions: [{
      dimension: 'ecommerce_overview.purchase_timestamp',
      granularity: 'month',
      dateRange: ['2018-01-01', '2018-08-31']
    }],
    order: {
      'ecommerce_overview.order_items_total_revenue': 'desc'
    },
    limit: 20
  });

  if (isLoading) return <div>Loading overview...</div>;
  if (error) return <div>Error: {error.toString()}</div>;

  return (
    <div className="overview-dashboard">
      <h2>📊 E-commerce Overview (Using Views)</h2>
      <table>
        <thead>
          <tr>
            <th>State</th>
            <th>Category</th>
            <th>Revenue</th>
            <th>Satisfaction</th>
          </tr>
        </thead>
        <tbody>
          {resultSet?.tablePivot().map((row, index) => (
            <tr key={index}>
              <td>{row['ecommerce_overview.customers_state']}</td>
              <td>{row['ecommerce_overview.products_category_english']}</td>
              <td>R$ {parseFloat(row['ecommerce_overview.order_items_total_revenue'] || 0).toLocaleString('pt-BR')}</td>
              <td>{parseFloat(row['ecommerce_overview.order_reviews_satisfaction_rate'] || 0).toFixed(1)}%</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

// Advanced component: Real-time KPI widgets
function KPIWidget({ title, value, format = 'number', trend }) {
  const formatValue = (val) => {
    switch (format) {
      case 'currency':
        return `R$ ${val.toLocaleString('pt-BR')}`;
      case 'percent':
        return `${val.toFixed(1)}%`;
      case 'rating':
        return `${val.toFixed(1)}/5 ⭐`;
      default:
        return val.toLocaleString();
    }
  };

  return (
    <div className="kpi-widget">
      <h3>{title}</h3>
      <div className="kpi-value">{formatValue(value)}</div>
      {trend && <div className={`trend ${trend > 0 ? 'positive' : 'negative'}`}>
        {trend > 0 ? '↗️' : '↘️'} {Math.abs(trend).toFixed(1)}%
      </div>}
    </div>
  );
}

function App() {
  return (
    <CubeProvider cubejsApi={cubejsApi}>
      <div className="app">
        <BrazilianEcommerceDashboard />
        <EcommerceOverviewDashboard />
      </div>
    </CubeProvider>
  );
}

export default App;
```

### Advanced API Usage: Custom Analytics Endpoints

```javascript
// server/custom-analytics.js
const express = require('express');
const CubejsServer = require('@cubejs-backend/server');

const server = new CubejsServer();

// Custom endpoint for Brazilian market insights
server.addExtension('POST', '/api/brazil-insights', async (req, res) => {
  const { dateRange, topN = 10 } = req.body;

  try {
    const cubejsApi = req.cubejsApi;

    // Parallel queries for comprehensive insights using corrected naming
    const [
      salesTrends,
      categoryPerformance,
      stateAnalysis,
      satisfactionMetrics,
      logisticsKPIs
    ] = await Promise.all([
      // Sales trend analysis
      cubejsApi.load({
        measures: ['order_items.total_revenue', 'orders.count'],
        timeDimensions: [{
          dimension: 'orders.purchase_timestamp',
          granularity: 'month',
          dateRange: dateRange || ['2017-01-01', '2018-08-31']
        }],
        segments: ['orders.delivered_orders']
      }),

      // Category performance
      cubejsApi.load({
        measures: ['order_items.total_revenue', 'order_items.count', 'order_items.avg_price'],
        dimensions: ['products.category_english'],
        segments: ['orders.delivered_orders'],
        order: { 'order_items.total_revenue': 'desc' },
        limit: topN
      }),

      // Geographic analysis
      cubejsApi.load({
        measures: ['order_items.total_revenue', 'customers.unique_customers'],
        dimensions: ['customers.state'],
        segments: ['orders.delivered_orders'],
        order: { 'order_items.total_revenue': 'desc' }
      }),

      // Customer satisfaction
      cubejsApi.load({
        measures: [
          'order_reviews.avg_review_score',
          'order_reviews.satisfaction_rate',
          'order_reviews.count'
        ],
        dimensions: ['order_reviews.review_category']
      }),

      // Logistics performance
      cubejsApi.load({
        measures: [
          'orders.avg_delivery_time',
          'orders.delivery_performance_rate'
        ],
        dimensions: ['orders.delivery_status', 'customers.state'],
        segments: ['orders.delivered_orders']
      })
    ]);

    // Process and combine insights
    const insights = {
      overview: {
        totalRevenue: salesTrends.tablePivot().reduce((sum, row) =>
          sum + parseFloat(row['order_items.total_revenue'] || 0), 0
        ),
        totalOrders: salesTrends.tablePivot().reduce((sum, row) =>
          sum + parseInt(row['orders.count'] || 0), 0
        ),
        avgReviewScore: satisfactionMetrics.tablePivot().find(r =>
          r['order_reviews.review_category'] === 'Overall'
        )?.['order_reviews.avg_review_score'] || 0
      },

      topCategories: categoryPerformance.tablePivot().slice(0, 5),

      statePerformance: stateAnalysis.tablePivot(),

      satisfaction: {
        overall: satisfactionMetrics.tablePivot(),
        trends: salesTrends.tablePivot()
      },

      logistics: {
        avgDeliveryTime: logisticsKPIs.tablePivot()[0]?.['orders.avg_delivery_time'] || 0,
        onTimeRate: logisticsKPIs.tablePivot()[0]?.['orders.delivery_performance_rate'] || 0
      }
    };

    res.json({
      success: true,
      insights,
      metadata: {
        dataSource: 'Brazilian E-Commerce Public Dataset (Olist)',
        period: dateRange || ['2017-01-01', '2018-08-31'],
        generatedAt: new Date().toISOString()
      }
    });

  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Usage example
// POST /api/brazil-insights
// {
//   "dateRange": ["2018-01-01", "2018-06-30"],
//   "topN": 15
// }
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
