# Shopify Interview Preparation Guide

## Table of Contents
- [General Shopify API Concepts](#general-shopify-api-concepts)
- [Authentication & Security](#authentication--security)
- [Working with Shopify APIs](#working-with-shopify-apis)
- [Webhooks & Event Handling](#webhooks--event-handling)
- [Rate Limiting & Performance](#rate-limiting--performance)
- [App Development Best Practices](#app-development-best-practices)
- [Checkout & Order Processing](#checkout--order-processing)
- [Theme & Storefront Customization](#theme--storefront-customization)
- [Advanced Implementations](#advanced-implementations)
- [Testing & Deployment Strategies](#testing--deployment-strategies)
- [Case Studies & Project Experience](#case-studies--project-experience)

---

## General Shopify API Concepts

### REST vs GraphQL API

**Key Differences:**
- **REST API**: Resource-based with multiple endpoints. Each request typically returns a complete resource.
  - Advantages: Simpler to understand, better caching, more familiar to many developers
  - Disadvantages: May require multiple requests for related data, can lead to over-fetching

- **GraphQL API**: Single endpoint where you specify exactly what data you need.
  - Advantages: Precise data selection, fewer requests, strongly typed schema
  - Disadvantages: More complex queries, potential for expensive operations

**Example REST API Call:**
```javascript
// Fetch a specific product
fetch('https://store.myshopify.com/admin/api/2023-07/products/12345.json', {
  headers: { 'X-Shopify-Access-Token': 'access_token' }
})
```

**Equivalent GraphQL Query:**
```graphql
{
  product(id: "gid://shopify/Product/12345") {
    id
    title
    description
    variants(first: 10) {
      edges {
        node {
          id
          price
          availableForSale
        }
      }
    }
  }
}
```

### API Selection for Different Scenarios

1. **Admin API** (REST/GraphQL): For building admin tools, dashboards, and operations that manage the store
   - Use for: Order management, product updates, customer data processing

2. **Storefront API** (GraphQL): For custom storefronts and headless commerce
   - Use for: Building custom shopping experiences, headless commerce solutions

3. **Checkout API**: For customizing the checkout experience
   - Use for: Custom checkout fields, post-purchase upsells, checkout extensions

### API Scopes & Configuration

API scopes determine what actions your app can perform. Proper scope selection follows the principle of least privilege.

**Common Scopes:**
- `read_products`, `write_products`: Product management
- `read_orders`, `write_orders`: Order management
- `read_customers`, `write_customers`: Customer data

**Configuration Example:**
```javascript
// When redirecting to OAuth
const scopes = 'read_products,write_products,read_orders';
const redirectUrl = `https://store.myshopify.com/admin/oauth/authorize?client_id=${apiKey}&scope=${scopes}&redirect_uri=${redirectUri}`;
```

### API Versioning

Shopify releases quarterly API versions (January, April, July, October).

**Best Practices:**
- Always specify an API version in your requests
- Stay current with versions, but don't upgrade on day one
- Plan for quarterly updates
- Test against release candidates

**Version Handling:**
```javascript
// REST API with version
const apiVersion = '2023-07';
fetch(`https://store.myshopify.com/admin/api/${apiVersion}/products.json`);

// GraphQL with version
const apiUrl = `https://store.myshopify.com/admin/api/${apiVersion}/graphql.json`;
```

---

## Authentication & Security

### OAuth 2.0 Flow Implementation

**Step-by-Step Process:**
1. App redirects merchant to Shopify authorization page with API key and required scopes
2. Merchant approves and Shopify redirects back with a temporary code
3. App exchanges code for permanent access token using API secret
4. App securely stores the token for future API calls

**Server-Side Code:**
```javascript
// Step 1: Redirect to Shopify
app.get('/auth', (req, res) => {
  const shop = req.query.shop;
  const redirectUri = `${appUrl}/auth/callback`;
  const scopes = 'read_products,write_orders';
  res.redirect(`https://${shop}/admin/oauth/authorize?client_id=${apiKey}&scope=${scopes}&redirect_uri=${redirectUri}`);
});

// Step 2 & 3: Handle callback and exchange for token
app.get('/auth/callback', async (req, res) => {
  const { shop, code, hmac } = req.query;
  
  // Verify hmac
  if (!verifyHmac(req.query, apiSecret)) {
    return res.status(403).send('Invalid signature');
  }
  
  // Exchange for token
  const response = await fetch(`https://${shop}/admin/oauth/access_token`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: apiKey,
      client_secret: apiSecret,
      code
    })
  });
  
  const { access_token } = await response.json();
  // Store token securely
  await storeTokenSecurely(shop, access_token);
  
  res.redirect('/dashboard');
});
```

### Securing Access Tokens

**Storage Options:**
- Environment variables (for development)
- Encrypted database fields
- Secret management services (AWS Secrets Manager, HashiCorp Vault)

**Never:**
- Commit tokens to source control
- Store in client-side code
- Log tokens in application logs

### Online vs Offline Tokens

- **Online Tokens**: Short-lived, require merchant to be logged in, used for embedded apps
- **Offline Tokens**: Long-lived, work without merchant presence, used for background operations

### HMAC Verification

The HMAC key verifies that requests are authentic and come from Shopify.

**Verification Code:**
```javascript
function verifyHmac(query, secret) {
  const hmac = query.hmac;
  const message = Object.entries(query)
    .filter(([key]) => key !== 'hmac')
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([key, value]) => `${key}=${value}`)
    .join('&');
  
  const generatedHash = crypto
    .createHmac('sha256', secret)
    .update(message)
    .digest('hex');
    
  return crypto.timingSafeEqual(
    Buffer.from(hmac),
    Buffer.from(generatedHash)
  );
}
```

### Securing Webhook Endpoints

**Best Practices:**
- Verify HMAC signatures on all webhook requests
- Use HTTPS for all endpoints
- Implement proper error handling
- Set up monitoring and alerts

**Webhook Verification:**
```javascript
app.post('/webhooks/orders/create', express.raw({ type: 'application/json' }), (req, res) => {
  const hmacHeader = req.headers['x-shopify-hmac-sha256'];
  const body = req.body;
  
  const generatedHash = crypto
    .createHmac('sha256', webhookSecret)
    .update(body)
    .digest('base64');
    
  const hashesMatch = crypto.timingSafeEqual(
    Buffer.from(hmacHeader),
    Buffer.from(generatedHash)
  );
  
  if (!hashesMatch) {
    return res.status(403).send('Invalid webhook signature');
  }
  
  // Process webhook
  const data = JSON.parse(body);
  // Handle order creation
  
  res.status(200).send('Webhook processed');
});
```

---

## Working with Shopify APIs

### Retrieving & Filtering Orders

**REST Example:**
```javascript
// Get all orders
fetch(`https://${shop}/admin/api/2023-07/orders.json?status=any`, {
  headers: { 'X-Shopify-Access-Token': accessToken }
});

// Get only unfulfilled orders
fetch(`https://${shop}/admin/api/2023-07/orders.json?fulfillment_status=unfulfilled`, {
  headers: { 'X-Shopify-Access-Token': accessToken }
});
```

**GraphQL Example:**
```graphql
{
  orders(first: 50, query: "fulfillment_status:unfulfilled") {
    edges {
      node {
        id
        name
        totalPriceSet {
          shopMoney { amount, currencyCode }
        }
        lineItems(first: 10) {
          edges {
            node {
              title
              quantity
            }
          }
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### Handling Pagination

**Cursor-Based Pagination (GraphQL):**
```javascript
async function fetchAllOrders(client, cursor = null) {
  let hasNextPage = true;
  let allOrders = [];
  
  while (hasNextPage) {
    const query = `{
      orders(first: 50, after: ${cursor ? `"${cursor}"` : null}) {
        edges {
          node {
            id
            name
          }
        }
        pageInfo {
          hasNextPage
          endCursor
        }
      }
    }`;
    
    const response = await client.query({ query });
    const { edges, pageInfo } = response.data.orders;
    
    allOrders = [...allOrders, ...edges.map(edge => edge.node)];
    hasNextPage = pageInfo.hasNextPage;
    cursor = pageInfo.endCursor;
    
    // Respect rate limits
    if (hasNextPage) {
      await new Promise(resolve => setTimeout(resolve, 500));
    }
  }
  
  return allOrders;
}
```

**Link-Based Pagination (REST):**
```javascript
async function fetchAllProducts(shop, accessToken) {
  let url = `https://${shop}/admin/api/2023-07/products.json?limit=250`;
  let allProducts = [];
  
  while (url) {
    const response = await fetch(url, {
      headers: { 'X-Shopify-Access-Token': accessToken }
    });
    
    const products = await response.json();
    allProducts = [...allProducts, ...products.products];
    
    // Check for Link header with 'next' rel
    const link = response.headers.get('Link');
    url = link ? extractNextUrl(link) : null;
    
    // Respect rate limits
    if (url) {
      await new Promise(resolve => setTimeout(resolve, 500));
    }
  }
  
  return allProducts;
}

function extractNextUrl(linkHeader) {
  const links = linkHeader.split(',');
  const nextLink = links.find(link => link.includes('rel="next"'));
  
  if (!nextLink) return null;
  
  const matches = nextLink.match(/<([^>]+)>/);
  return matches ? matches[1] : null;
}
```

### Minimizing API Calls

**Best Practices:**
1. Use GraphQL to fetch exactly what you need in one request
2. Implement efficient caching (Redis, Memcached)
3. Batch operations where possible
4. Use webhooks for real-time updates instead of polling
5. Implement backoff strategies for retries

**Caching Example:**
```javascript
const cache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

async function getProductWithCache(productId, shop, accessToken) {
  const cacheKey = `product_${productId}`;
  
  if (cache.has(cacheKey)) {
    const { data, timestamp } = cache.get(cacheKey);
    if (Date.now() - timestamp < CACHE_TTL) {
      return data;
    }
  }
  
  const response = await fetch(`https://${shop}/admin/api/2023-07/products/${productId}.json`, {
    headers: { 'X-Shopify-Access-Token': accessToken }
  });
  
  const data = await response.json();
  cache.set(cacheKey, { data, timestamp: Date.now() });
  
  return data;
}
```

### Common GraphQL Queries

**Product Query with Variants and Images:**
```graphql
{
  product(id: "gid://shopify/Product/12345") {
    id
    title
    description
    onlineStoreUrl
    images(first: 10) {
      edges {
        node {
          id
          url
          altText
        }
      }
    }
    variants(first: 25) {
      edges {
        node {
          id
          title
          price
          inventoryQuantity
          selectedOptions {
            name
            value
          }
        }
      }
    }
  }
}
```

**Customer with Orders:**
```graphql
{
  customer(id: "gid://shopify/Customer/12345") {
    id
    firstName
    lastName
    email
    orders(first: 10) {
      edges {
        node {
          id
          name
          totalPriceSet {
            shopMoney { amount, currencyCode }
          }
          createdAt
        }
      }
    }
  }
}
```

---

## Webhooks & Event Handling

### Webhook Registration

**REST API Registration:**
```javascript
async function createWebhook(shop, accessToken, topic, address) {
  const response = await fetch(`https://${shop}/admin/api/2023-07/webhooks.json`, {
    method: 'POST',
    headers: {
      'X-Shopify-Access-Token': accessToken,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      webhook: {
        topic,
        address,
        format: 'json'
      }
    })
  });
  
  return response.json();
}

// Example usage
createWebhook(shop, accessToken, 'orders/create', 'https://myapp.com/webhooks/orders/create');
```

**GraphQL Registration:**
```graphql
mutation {
  webhookSubscriptionCreate(
    topic: ORDERS_CREATE,
    webhookSubscription: {
      callbackUrl: "https://myapp.com/webhooks/orders/create",
      format: JSON
    }
  ) {
    webhookSubscription {
      id
    }
    userErrors {
      field
      message
    }
  }
}
```

### Common Webhook Topics

- **Orders**: `orders/create`, `orders/updated`, `orders/fulfilled`, `orders/cancelled`
- **Products**: `products/create`, `products/update`, `products/delete`
- **Customers**: `customers/create`, `customers/update`
- **App**: `app/uninstalled`

### Handling Webhook Failures

**Best Practices:**
1. Implement idempotent handlers (can process the same webhook multiple times without side effects)
2. Use a queuing system (RabbitMQ, SQS) to decouple receiving from processing
3. Implement retry logic with exponential backoff
4. Set up monitoring and alerting for webhook failures
5. Log detailed information for debugging

**Example Webhook Handler with Queue:**
```javascript
app.post('/webhooks/orders/create', express.raw({ type: 'application/json' }), async (req, res) => {
  // Verify webhook (as shown earlier)
  
  try {
    // Queue for processing instead of processing inline
    await queue.add('process-order', {
      orderId: order.id,
      payload: order
    }, {
      attempts: 5,
      backoff: {
        type: 'exponential',
        delay: 10000
      }
    });
    
    // Respond quickly to Shopify
    res.status(200).send('Webhook received');
  } catch (error) {
    console.error('Failed to queue webhook', error);
    // Still return 200 to avoid Shopify retries (we'll handle retry logic ourselves)
    res.status(200).send('Webhook received with processing error');
  }
});

// Process from queue
queue.process('process-order', async (job) => {
  const { orderId, payload } = job.data;
  
  // Process order here
  await updateOrderStatus(orderId);
  
  return { success: true };
});
```

### Order Fulfillment Detection

**Using Webhooks:**
```javascript
app.post('/webhooks/orders/fulfilled', express.raw({ type: 'application/json' }), async (req, res) => {
  // Verify webhook
  
  const order = JSON.parse(req.body);
  
  // Update database
  await db.orders.update({
    where: { shopifyOrderId: order.id },
    data: { status: 'fulfilled', fulfilledAt: new Date() }
  });
  
  // Send notification to customer
  await sendFulfillmentNotification(order);
  
  res.status(200).send('Order fulfillment processed');
});
```

### Real-time Dashboard Updates

**Options:**
1. **WebSockets**: Push updates to connected clients when webhooks are received
2. **Server-Sent Events**: One-way channel from server to browser
3. **Polling with Efficient Caching**: For simpler implementations

**WebSocket Example:**
```javascript
// Server-side
const WebSocket = require('ws');
const wss = new WebSocket.Server({ server });

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
});

// When webhook is received
app.post('/webhooks/orders/create', express.raw({ type: 'application/json' }), (req, res) => {
  // Verify and process webhook
  
  // Broadcast to all connected clients
  wss.clients.forEach((client) => {
    if (client.isAlive && client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify({
        type: 'ORDER_CREATED',
        order: orderSummary
      }));
    }
  });
  
  res.status(200).send('Webhook processed');
});
```

---

## Rate Limiting & Performance

### Shopify Rate Limits

**REST API Limits:**
- 40 requests per app per store per second (Higher for Plus)
- Leaky bucket algorithm (2 request bucket refill per second)

**GraphQL API Limits:**
- Cost-based system (1000 points per app per store per second)
- Query complexity determines cost

### Monitoring Rate Limits

**REST Headers:**
- `X-Shopify-Shop-Api-Call-Limit`: Current usage (e.g., "39/40")

**GraphQL Headers:**
- `X-Shopify-Shop-Api-Call-Limit`: Points used (e.g., "725/1000")

**Code Example:**
```javascript
async function shopifyRequest(url, options, retries = 3) {
  const response = await fetch(url, options);
  
  // Check rate limit
  const rateLimitHeader = response.headers.get('X-Shopify-Shop-Api-Call-Limit');
  if (rateLimitHeader) {
    const [current, limit] = rateLimitHeader.split('/').map(Number);
    console.log(`API Rate Limit: ${current}/${limit}`);
    
    // If close to limit, add delay
    if (current > limit * 0.8) {
      console.log('Approaching rate limit, adding delay');
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
  
  // Handle 429 Too Many Requests
  if (response.status === 429 && retries > 0) {
    console.log('Rate limited, retrying after delay');
    await new Promise(resolve => setTimeout(resolve, 2000));
    return shopifyRequest(url, options, retries - 1);
  }
  
  return response;
}
```

### Throttling Implementation

**Leaky Bucket Implementation:**
```javascript
class LeakyBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.refillRate = refillRate; // tokens per ms
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }
  
  async take(cost = 1) {
    // Refill tokens based on time elapsed
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + (elapsed * this.refillRate)
    );
    this.lastRefill = now;
    
    if (this.tokens < cost) {
      // Not enough tokens, calculate wait time
      const waitTime = (cost - this.tokens) / this.refillRate;
      await new Promise(resolve => setTimeout(resolve, waitTime));
      this.tokens = 0;
      this.lastRefill = Date.now();
    } else {
      // Enough tokens, consume them
      this.tokens -= cost;
    }
  }
}

// Usage for REST API (40 requests/second)
const restBucket = new LeakyBucket(40, 40/1000); // 40 capacity, refill 40 per second

async function makeShopifyRequest() {
  await restBucket.take();
  // Make the API call
}
```

### Handling API Downtime

**Best Practices:**
1. Implement circuit breaker pattern
2. Queue non-urgent operations for later
3. Provide meaningful user feedback
4. Have fallback options when possible

**Circuit Breaker Implementation:**
```javascript
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 30000;
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.nextAttempt = Date.now();
  }
  
  async fire(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit is OPEN');
      }
      this.state = 'HALF-OPEN';
    }
    
    try {
      const result = await this.fn(...args);
      this.success();
      return result;
    } catch (error) {
      this.failure();
      throw error;
    }
  }
  
  success() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  failure() {
    this.failureCount++;
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
    }
  }
}

// Usage
const apiClient = new CircuitBreaker(
  async (url, options) => {
    const response = await fetch(url, options);
    if (!response.ok) throw new Error(`API Error: ${response.status}`);
    return response.json();
  },
  { failureThreshold: 3, resetTimeout: 10000 }
);

// Using the circuit breaker
try {
  const data = await apiClient.fire(url, options);
  // Process data
} catch (error) {
  if (error.message === 'Circuit is OPEN') {
    // Show user-friendly message
    console.log('Shopify API is currently unavailable. Please try again later.');
  } else {
    // Handle other errors
  }
}
```

---

## App Development Best Practices

### Multi-Store App Architecture

**Key Components:**
1. **Authentication System**: Manages OAuth flow and token storage for each store
2. **Database Schema**: Partitions data by shop domain
3. **Job Processing**: Background workers for async operations
4. **API Client**: Manages rate limits and retries per shop

**Database Schema Example:**
```sql
CREATE TABLE shops (
  id SERIAL PRIMARY KEY,
  shop_domain VARCHAR(255) UNIQUE NOT NULL,
  access_token VARCHAR(255) NOT NULL,
  scopes TEXT NOT NULL,
  installed_at TIMESTAMP NOT NULL DEFAULT NOW(),
  uninstalled_at TIMESTAMP NULL
);

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  shop_id INTEGER REFERENCES shops(id),
  shopify_product_id BIGINT NOT NULL,
  title VARCHAR(255) NOT NULL,
  /* other fields */
  UNIQUE(shop_id, shopify_product_id)
);
```

### Shopify App Bridge

App Bridge provides a framework for embedded app UI integration with Shopify admin.

**Key Features:**
- Navigation within Shopify admin
- Modal windows and toasts
- Session token management
- Resource picker (products, collections, etc.)

**Implementation Example:**
```javascript
import createApp from '@shopify/app-bridge';
import { Toast, NavigationMenu } from '@shopify/app-bridge/actions';

const app = createApp({
  apiKey: API_KEY,
  host: new URLSearchParams(window.location.search).get('host'),
  forceRedirect: true
});

// Show a toast message
const toastNotice = Toast.create(app, {
  message: 'Settings saved successfully',
  duration: 3000
});

// Navigate within the app
const redirect = NavigationMenu.create(app);
redirect.dispatch(NavigationMenu.Action.APP, '/settings');
```

### App Uninstallation Cleanup

**Best Practices:**
1. Set up `app/uninstalled` webhook
2. Soft-delete shop data (mark as inactive but don't remove)
3. Clean up external resources
4. Revoke API tokens
5. Handle potential reinstallation gracefully

**Webhook Handler:**
```javascript
app.post('/webhooks/app/uninstalled', express.raw({ type: 'application/json' }), async (req, res) => {
  // Verify webhook
  
  const shop = JSON.parse(req.body).myshopify_domain;
  
  try {
    // Mark shop as uninstalled
    await db.shops.update({
      where: { shop_domain: shop },
      data: { 
        uninstalled_at: new Date(),
        access_token: null // Clear token for security
      }
    });
    
    // Clean up external resources
    await cleanupExternalResources(shop);
    
    res.status(200).send('Uninstallation processed');
  } catch (error) {
    console.error('Uninstallation handler error', error);
    res.status(500).send('Error processing uninstallation');
  }
});
```

### API Version Migration

**Strategy:**
1. Monitor Shopify announcements for upcoming changes
2. Test against release candidates
3. Use API version constants in your codebase
4. Implement gradual rollout for version upgrades

**Version Management:**
```javascript
// config.js
const API_VERSIONS = {
  current: '2023-07',
  next: '2023-10',
  // Add dates when each version will be unsupported
  deprecationDates: {
    '2023-04': new Date('2024-04-01'),
    '2023-01': new Date('2024-01-01')
  }
};

// API client
function getApiUrl(shop, endpoint, useNext = false) {
  const version = useNext ? API_VERSIONS.next : API_VERSIONS.current;
  return `https://${shop}/admin/api/${version}/${endpoint}`;
}

// Gradual rollout to next version
function shouldUseNextVersion(shop) {
  // Use next version for 10% of shops initially
  const shopHash = createHash('md5').update(shop).digest('hex');
  const hashNum = parseInt(shopHash.substring(0, 8), 16);
  return hashNum % 100 < 10; // 10% of shops
}
```

---

## Checkout & Order Flow

### Order Creation Flow

**Standard Flow:**
1. Customer adds products to cart
2. Customer proceeds to checkout
3. Customer enters shipping/billing information
4. Payment processing occurs
5. Order is created with "paid" or "pending" status
6. Confirmation is shown to customer
7. Fulfillment process begins

**Webhook Sequence:**
1. `carts/create` - When cart is created
2. `carts/update` - When items are added/removed
3. `checkouts/create` - When checkout is initiated
4. `checkouts/update` - When checkout information changes
5. `orders/create` - When order is created
6. `orders/paid` - When payment is processed
7. `orders/fulfilled` - When items are shipped

### Checkout UI Extensions

Checkout UI Extensions allow for customizing the checkout experience within Shopify's system.

**Key Features:**
- Add custom fields and components
- Modify shipping options
- Add post-purchase upsells
- Customize the layout and appearance

**Implementation Example:**
```javascript
import {
  extension,
  TextField,
  BlockStack,
  Button,
  useCartLines
} from '@shopify/checkout-ui-extensions-react';

extension('Checkout::Dynamic::Render', (root, { extensionPoint }) => {
  root.appendChild(
    <BlockStack spacing="loose">
      <TextField
        label="Special Instructions"
        multiline={3}
        onChange={(value) => {
          // Store instructions
          localStorage.setItem('specialInstructions', value);
        }}
      />
      <Button
        onPress={() => {
          // Handle button press
        }}
      >
        Add Gift Wrapping ($5)
      </Button>
    </BlockStack>
  );
});
```

### Accessing Cart Items in Checkout

**Using Checkout UI Extensions:**
```javascript
import { useCartLines } from '@shopify/checkout-ui-extensions-react';

function CartItemsComponent() {
  const cartLines = useCartLines();
  
  return (
    <BlockStack spacing="loose">
      <Heading>Your Items</Heading>
      {cartLines.map((line) => (
        <BlockStack key={line.id}>
          <Text>{line.merchandise.product.title}</Text>
          <Text>Quantity: {line.quantity}</Text>
          <Text>${line.cost.totalAmount.amount}</Text>
        </BlockStack>
      ))}
    </BlockStack>
  );
}
```

### Customer Events and Pixels

**Customer Events:**
Events triggered by user interactions that can be used for analytics and personalization.

**Common Events:**
- `page_viewed`
- `product_viewed` 
- `product_added_to_cart`
- `checkout_started`
- `order_completed`

**Pixel Implementation:**
```javascript
// Add Facebook Pixel to theme
{% if settings.facebook_pixel_id != blank %}
<script>
  !function(f,b,e,v,n,t,s){if(f.fbq)return;n=f.fbq=function(){n.callMethod?
  n.callMethod.apply(n,arguments):n.queue.push(arguments)};if(!f._fbq)f._fbq=n;
  n.push=n;n.loaded=!0;n.version='2.0';n.queue=[];t=b.createElement(e);t.async=!0;
  t.src=v;s=b.getElementsByTagName(e)[0];s.parentNode.insertBefore(t,s)}(window,
  document,'script','https://connect.facebook.net/en_US/fbevents.js');
  
  fbq('init', '{{ settings.facebook_pixel_id }}');
  fbq('track', 'PageView');
  
  document.addEventListener('product:added', function(event) {
    fbq('track', 'AddToCart', {
      content_name: event.detail.product.title,
      content_ids: [event.detail.product.id],
      content_type: 'product',
      value: event.detail.product.price,
      currency: {{ shop.currency | json }}
    });
  });
</script>
{% endif %}
```

---

## Theme & Storefront Customization

### Connecting Video Files to Products

**Options:**
1. **Metafields**: Store video URLs in product metafields
2. **Media API**: Upload videos as product media
3. **File Metafields**: Store video files directly with products

**Implementation with Metafields:**
```html
{% comment %} In product-template.liquid {% endcomment %}
{% if product.metafields.custom.product_video %}
  <div class="product-video-container">
    <video 
      controls 
      poster="{{ product.featured_image | img_url: 'large' }}"
      src="{{ product.metafields.custom.product_video }}"
    >
      Your browser does not support the video tag.
    </video>
  </div>
{% endif %}
```

**Setting up Metafields:**
```javascript
// Admin API call to define metafield
fetch(`https://${shop}/admin/api/2023-07/metafields.json`, {
  method: 'POST',
  headers: {
    'X-Shopify-Access-Token': accessToken,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metafield: {
      namespace: 'custom',
      key: 'product_video',
      type: 'url',
      value: 'https://cdn.example.com/videos/product-demo.mp4',
      owner_id: productId,
      owner_resource: 'product'
    }
  })
});
```

### Multi-Color Products in PLP

**Approaches:**
1. **Separate Products**: Create individual products for each color
2. **Color Swatches**: Display color options on product cards
3. **Dynamic Variants**: Change product card image when color is selected

**Implementation with Color Swatches:**
```html
{% comment %} In product-card.liquid {% endcomment %}
<div class="product-card">
  <a href="{{ product.url }}">
    <img 
      src="{{ product.featured_image | img_url: 'medium' }}" 
      alt="{{ product.title }}"
      id="productImage-{{ product.id }}"
    >
    <h3>{{ product.title }}</h3>
    <p>{{ product.price | money }}</p>
  </a>
  
  {% if product.has_only_default_variant == false %}
    <div class="color-swatch-container">
      {% assign color_option_index = 0 %}
      {% assign color_option_name = '' %}
      
      {% for option in product.options %}
        {% if option == 'Color' or option == 'Colour' %}
          {% assign color_option_index = forloop.index0 %}
          {% assign color_option_name = option %}
        {% endif %}
      {% endfor %}
      
      {% if color_option_name != '' %}
        {% for variant in product.variants %}
          {% assign color_value = variant.options[color_option_index] %}
          <button 
            class="color-swatch" 
            style="background-color: {{ color_value | downcase }}"
            data-image="{{ variant.featured_image | img_url: 'medium' }}"
            data-variant-id="{{ variant.id }}"
            onclick="updateProductImage('{{ product.id }}', this.getAttribute('data-image'))"
          ></button>
        {% endfor %}
      {% endif %}
    </div>
  {% endif %}
</div>

<script>
  function updateProductImage(productId, imageUrl) {
    document.getElementById('productImage-' + productId).src = imageUrl;
  }
</script>
```

### Theme Customization Without Extensions

**Options:**
1. **Theme Settings**: Customize using theme settings in the theme editor
2. **Liquid Files**: Edit or add liquid files to modify functionality
3. **Custom JavaScript**: Add custom JS for additional functionality
4. **Asset Uploads**: Add custom CSS, images, or other assets
5. **Metafields**: Use metafields for custom data

**Example of Theme Settings:**
```html
{% comment %} In theme.liquid {% endcomment %}
{% if settings.enable_custom_header %}
  {% include 'custom-header' %}
{% else %}
  {% include 'header' %}
{% endif %}
```

**Custom Theme Setting in settings_schema.json:**
```json
{
  "name": "Header Settings",
  "settings": [
    {
      "type": "checkbox",
      "id": "enable_custom_header",
      "label": "Use Custom Header",
      "default": false
    },
    {
      "type": "text",
      "id": "header_message",
      "label": "Header Announcement",
      "default": "Free shipping on all orders over $50!"
    }
  ]
}
```

### Hiding Out-of-Stock Products

**Options:**
1. **Collection Filtering**: Filter collections to hide out-of-stock products
2. **Theme Customization**: Hide products with zero inventory
3. **Search & Discovery API**: Use API to filter products by inventory

**Collection Filtering with Liquid:**
```html
{% comment %} In collection-template.liquid {% endcomment %}
<div class="product-grid">
  {% for product in collection.products %}
    {% assign show_product = true %}
    
    {% if settings.hide_out_of_stock %}
      {% for variant in product.variants %}
        {% if variant.available == false %}
          {% assign all_variants_count = product.variants | size %}
          {% assign unavailable_variants_count = 0 %}
          
          {% for v in product.variants %}
            {% unless v.available %}
              {% assign unavailable_variants_count = unavailable_variants_count | plus: 1 %}
            {% endunless %}
          {% endfor %}
          
          {% if all_variants_count == unavailable_variants_count %}
            {% assign show_product = false %}
          {% endif %}
        {% endif %}
      {% endfor %}
    {% endif %}
    
    {% if show_product %}
      <div class="product-card">
        {% include 'product-card', product: product %}
      </div>
    {% endif %}
  {% endfor %}
</div>
```

**Setting in settings_schema.json:**
```json
{
  "type": "checkbox",
  "id": "hide_out_of_stock",
  "label": "Hide out of stock products",
  "default": false
}
```

### Multi-Market PDP Layouts

**Implementation Options:**
1. **Market-Specific Sections**: Create market-specific sections in theme
2. **Dynamic Templates**: Use different templates based on market
3. **CSS Classes**: Apply market-specific CSS classes
4. **Metafields**: Store market-specific content in metafields

**Implementation with Sections:**
```html
{% comment %} In product-template.liquid {% endcomment %}
{% if shop.metafields.markets.current_market == 'US' %}
  {% section 'product-us-market' %}
{% elsif shop.metafields.markets.current_market == 'EU' %}
  {% section 'product-eu-market' %}
{% else %}
  {% section 'product-default' %}
{% endif %}
```

**Market Detection with JavaScript:**
```javascript
// Get customer's country from geolocation or browser locale
function detectMarket() {
  // Use browser language as a simple detection method
  const language = navigator.language || navigator.userLanguage;
  
  if (language.startsWith('en-US')) {
    return 'US';
  } else if (language.startsWith('de') || language.startsWith('fr')) {
    return 'EU';
  } else {
    return 'GLOBAL';
  }
}

// Store market in localStorage and as a cookie
const market = detectMarket();
localStorage.setItem('detected_market', market);
document.cookie = `detected_market=${market}; path=/; max-age=86400`;
```

### Storefront API for Headless Commerce

**Key Features:**
- Public access to product, collection, and checkout data
- Supports custom storefronts
- Requires Storefront API access token

**Product Query Example:**
```graphql
query GetProductDetails($handle: String!) {
  product(handle: $handle) {
    id
    title
    description
    handle
    tags
    priceRange {
      minVariantPrice {
        amount
        currencyCode
      }
    }
    images(first: 10) {
      edges {
        node {
          url(transform: {
            maxWidth: 800
            maxHeight: 800
            crop: CENTER
          })
          altText
        }
      }
    }
    variants(first: 25) {
      edges {
        node {
          id
          title
          availableForSale
          quantityAvailable
          priceV2 {
            amount
            currencyCode
          }
          selectedOptions {
            name
            value
          }
        }
      }
    }
  }
}
```

**React Implementation:**
```jsx
import { useQuery, gql } from '@apollo/client';

const GET_PRODUCT = gql`
  query GetProduct($handle: String!) {
    product(handle: $handle) {
      id
      title
      description
      images(first: 5) {
        edges {
          node {
            url
            altText
          }
        }
      }
      variants(first: 25) {
        edges {
          node {
            id
            title
            price
            availableForSale
          }
        }
      }
    }
  }
`;

function ProductDetail({ handle }) {
  const { loading, error, data } = useQuery(GET_PRODUCT, {
    variables: { handle }
  });
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  
  const product = data.product;
  const images = product.images.edges.map(edge => edge.node);
  const variants = product.variants.edges.map(edge => edge.node);
  
  return (
    <div className="product-detail">
      <div className="product-images">
        {images.map(image => (
          <img key={image.url} src={image.url} alt={image.altText} />
        ))}
      </div>
      
      <div className="product-info">
        <h1>{product.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: product.description }} />
        
        <div className="variants">
          {variants.map(variant => (
            <div key={variant.id} className="variant">
              <h3>{variant.title}</h3>
              <p>${variant.price}</p>
              <button disabled={!variant.availableForSale}>
                {variant.availableForSale ? 'Add to Cart' : 'Sold Out'}
              </button>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

---

## Advanced Implementations

### B2B Implementation

**Key Features:**
- Customer-specific pricing
- Minimum order quantities
- Company accounts with multiple users
- Custom checkout flow
- Purchase orders & invoicing

**Implementation with Metafields:**
```javascript
// Set company-specific pricing
fetch(`https://${shop}/admin/api/2023-07/customers/${customerId}/metafields.json`, {
  method: 'POST',
  headers: {
    'X-Shopify-Access-Token': accessToken,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metafield: {
      namespace: 'b2b',
      key: 'price_list',
      value: JSON.stringify({
        discount_percentage: 10,
        minimum_order_value: 500,
        payment_terms: 'Net 30'
      }),
      type: 'json'
    }
  })
});
```

**Liquid Implementation for B2B Pricing:**
```html
{% comment %} In product-price.liquid {% endcomment %}
{% if customer and customer.metafields.b2b.price_list %}
  {% assign b2b_pricing = customer.metafields.b2b.price_list %}
  {% assign discount = b2b_pricing.discount_percentage | divided_by: 100.0 %}
  {% assign original_price = product.price %}
  {% assign discounted_price = original_price | minus: original_price | times: discount %}
  
  <p class="price">
    <s>{{ original_price | money }}</s>
    <span class="b2b-price">{{ discounted_price | money }}</span>
  </p>
  
  {% if b2b_pricing.minimum_order_value > 0 %}
    <p class="min-order">
      Minimum order: {{ b2b_pricing.minimum_order_value | money }}
    </p>
  {% endif %}
{% else %}
  <p class="price">{{ product.price | money }}</p>
{% endif %}
```

### Multi-Currency Implementation

**Options:**
1. **Shopify Markets**: Built-in multi-currency support
2. **Currency Selector**: Custom implementation with JavaScript
3. **Country/Currency Selector**: Combined selector for country and currency

**Using Shopify Markets:**
```html
{% comment %} In theme.liquid {% endcomment %}
<div class="currency-selector">
  <form method="post" action="/localization" id="localization_form" accept-charset="UTF-8">
    <input type="hidden" name="form_type" value="localization" />
    <input type="hidden" name="return_to" value="{{ request.path }}" />
    
    <select name="country_code" id="country-selector">
      {% for country in localization.available_countries %}
        <option 
          value="{{ country.iso_code }}" 
          {% if country.iso_code == localization.country.iso_code %}selected="selected"{% endif %}
        >
          {{ country.name }} ({{ country.currency.iso_code }} {{ country.currency.symbol }})
        </option>
      {% endfor %}
    </select>
  </form>
</div>

<script>
  document.getElementById('country-selector').addEventListener('change', function() {
    document.getElementById('localization_form').submit();
  });
</script>
```

### Shopify Functions

Shopify Functions allow for customizing core Shopify processes like discount calculation and shipping options.

**Use Cases:**
- Custom discount rules
- Dynamic shipping rates
- Payment method selection
- Order routing

**Example Function (WebAssembly):**
```rust
use shopify_function::prelude::*;
use shopify_function::Result;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct Input {
    cart: Cart,
    configuration: Configuration,
}

#[derive(Serialize, Deserialize)]
struct Configuration {
    minimum_order_amount: String,
    discount_percentage: String,
}

#[derive(Serialize, Deserialize)]
struct Cart {
    lines: Vec<CartLine>,
    cost: CartCost,
}

#[derive(Serialize, Deserialize)]
struct CartLine {
    merchandise: Merchandise,
    quantity: i32,
}

#[derive(Serialize, Deserialize)]
struct Merchandise {
    product_id: String,
}

#[derive(Serialize, Deserialize)]
struct CartCost {
    total_amount: Money,
}

#[derive(Serialize, Deserialize)]
struct Money {
    amount: String,
}

#[derive(Serialize, Deserialize)]
struct Output {
    discounts: Vec<Discount>,
}

#[derive(Serialize, Deserialize)]
struct Discount {
    message: String,
    targets: Vec<Target>,
    value: Value,
}

#[derive(Serialize, Deserialize)]
struct Target {
    product_id: String,
}

#[derive(Serialize, Deserialize)]
struct Value {
    percentage: f64,
}

#[shopify_function]
fn function(input: Input) -> Result<Output> {
    let minimum_amount: f64 = input.configuration.minimum_order_amount.parse()?;
    let discount_percentage: f64 = input.configuration.discount_percentage.parse()?;
    let cart_total: f64 = input.cart.cost.total_amount.amount.parse()?;
    
    let mut discounts = Vec::new();
    
    if cart_total >= minimum_amount {
        for line in &input.cart.lines {
            discounts.push(Discount {
                message: format!("{}% off when you spend ${} or more", discount_percentage, minimum_amount),
                targets: vec![Target {
                    product_id: line.merchandise.product_id.clone(),
                }],
                value: Value {
                    percentage: discount_percentage,
                },
            });
        }
    }
    
    Ok(Output { discounts })
}
```

### Shopify Plus Advantages

**Key Features:**
1. **Advanced API Limits**: Higher rate limits for API calls
2. **Script Editor**: Custom scripts for checkout, shipping, and payments
3. **Multiple Storefronts**: Up to 10 expansion stores
4. **Flow**: Automation platform for business processes
5. **Launchpad**: Scheduled theme and product changes
6. **Wholesale Channel**: B2B sales channel
7. **Dedicated Support**: Priority support and dedicated success manager

**Example Flow Automation:**
```json
{
  "name": "High-Value Order Notification",
  "trigger": {
    "type": "order_created",
    "conditions": [
      {
        "property": "total_price",
        "comparison": "greater_than",
        "value": 500
      }
    ]
  },
  "actions": [
    {
      "type": "email_notification",
      "recipients": ["sales@example.com"],
      "subject": "High-Value Order Received",
      "body": "A new order with value over $500 has been received.\n\nOrder #: {{order.name}}\nCustomer: {{order.customer.first_name}} {{order.customer.last_name}}\nTotal: {{order.total_price | money}}"
    },
    {
      "type": "tag_object",
      "object": "order",
      "tags": ["high-value", "priority"]
    }
  ]
}
```

---

## Testing & Deployment Strategies

### Testing Without Affecting Live Store

**Options:**
1. **Development Store**: Free development store for testing
2. **Test Orders**: Use test payment gateway
3. **Theme Preview**: Test theme changes without publishing
4. **Duplicate Theme**: Work on a copy of the live theme

**Test Payment Gateway:**
```json
{
  "payment_gateway": {
    "name": "Bogus Gateway",
    "provider_id": "bogus",
    "credentials": {
      "bogus": true
    }
  }
}
```

### Testing Webhook Behavior

**Options:**
1. **Webhook Simulators**: Tools like Shopify's CLI webhook simulator
2. **ngrok**: Expose local development server to the internet
3. **Mock Data**: Generate mock webhook payloads

**Using Shopify CLI:**
```bash
shopify webhook trigger orders/create --api-version 2023-07 --delivery-url https://yourapp.ngrok.io/webhooks/orders/create
```

**Using ngrok:**
```bash
# Start your local server
npm run dev

# In another terminal, start ngrok
ngrok http 3000
```

### CI/CD for Shopify Apps

**Tools:**
- GitHub Actions
- CircleCI
- Jenkins
- Shopify CLI

**Example GitHub Actions Workflow:**
```yaml
name: Deploy Shopify App

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: ${{ secrets.HEROKU_APP_NAME }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
```

### Error Monitoring in Production

**Tools:**
- Sentry
- New Relic
- Datadog
- Bugsnag

**Sentry Integration:**
```javascript
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  integrations: [new ProfilingIntegration()],
  tracesSampleRate: 1.0,
  profilesSampleRate: 1.0,
  environment: process.env.NODE_ENV
});

app.use(Sentry.Handlers.requestHandler());

// Your middleware and routes

app.use(Sentry.Handlers.errorHandler());

// Add Shopify context to errors
app.use((req, res, next) => {
  if (req.query.shop) {
    Sentry.configureScope((scope) => {
      scope.setTag('shop', req.query.shop);
    });
  }
  next();
});
```

---

## Case Studies & Project Experience

When discussing past projects, structure your answers using the STAR method:

- **Situation**: Context of the project
- **Task**: What needed to be done
- **Action**: What you specifically did
- **Result**: The outcome and benefits

### Example Case Study: Multi-currency Store

**Situation:**
A fashion retailer wanted to expand internationally, requiring multiple currencies, localized content, and market-specific product offerings.

**Task:**
Implement a multi-market solution that would:
- Support 5 currencies
- Display localized pricing
- Customize product availability by market
- Provide market-specific shipping options

**Action:**
- Implemented Shopify Markets
- Created market-specific metafields for localized content
- Developed custom theme sections that adapt based on market
- Built a currency/country selector in the header
- Integrated with market-specific payment gateways

**Result:**
- 40% increase in international sales
- 25% reduction in cart abandonment
- Successfully launched in 5 new markets within 3 months
- Improved customer experience with localized content

### Example Case Study: B2B Implementation

**Situation:**
A wholesale manufacturer wanted to transform their B2B ordering process from manual orders to an online portal.

**Task:**
Create a B2B portal that would:
- Support customer-specific pricing
- Allow bulk ordering
- Implement purchase order payment methods
- Provide order history and reordering

**Action:**
- Developed on Shopify Plus
- Created customer tags and metafields for company-specific pricing
- Built a custom checkout experience using Checkout Extensions
- Implemented CSV order upload functionality
- Created customer portals for order history

**Result:**
- Reduced order processing time by 75%
- Increased average order value by 30%
- Eliminated manual entry errors
- Improved customer satisfaction with self-service capabilities

---

I hope this comprehensive guide helps you prepare for your Shopify interview! Remember to:

1. Review all topics thoroughly, especially those most relevant to your experience
2. Prepare specific examples for each major area
3. Practice explaining technical concepts clearly
4. Stay up-to-date with latest Shopify features

Good luck with your interview!
