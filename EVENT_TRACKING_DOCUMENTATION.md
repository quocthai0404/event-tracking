# 📊 Event Tracking System - Complete Technical Documentation

> **Tài liệu đầy đủ cho Frontend & Backend Team**  
> Document này được viết dựa trên **SOURCE CODE THỰC TẾ** của hệ thống

---

## 📋 Table of Contents

1. [System Architecture](#-system-architecture)
2. [API Endpoint](#-api-endpoint)
3. [Request Flow](#-request-flow)
4. [Authentication & Identity](#-authentication--identity)
5. [Request Schema](#-request-schema)
6. [Response Format](#-response-format)
7. [17 Event Types](#-17-event-types)
8. [Context Enrichment](#-context-enrichment)
9. [Queue & Worker System](#-queue--worker-system)
10. [Implementation Guide](#-implementation-guide)
11. [Best Practices](#-best-practices)
12. [Database Schema](#-database-schema)
13. [Troubleshooting](#-troubleshooting)

---

## 🏗 System Architecture

### Complete Request Flow

```
┌─────────────┐
│  Frontend   │
│  (Vue 3)    │
└──────┬──────┘
       │ POST /api/events
       │ { event_name, properties, context }
       ↓
┌─────────────────────────────────────────────────────────┐
│                    Hono Server                          │
│                  (Port: 4200)                           │
├─────────────────────────────────────────────────────────┤
│  Middleware Stack:                                      │
│  1. Request ID + Timing                                 │
│  2. CORS (origin: *, credentials: true)                 │
│  3. Compress                                            │
│  4. CSRF Protection (skip /api/internal/*)              │
│  5. eventIdentityMiddleware ← QUAN TRỌNG               │
│     - Đọc/tạo cookie `aid` (30 ngày)                   │
│     - Đọc/tạo cookie `sid` (30 phút / 24h max)         │
│     - Extract JWT token từ Authorization header         │
│     - Parse UTM params (utm_source, utm_campaign)       │
│     - Set c.get('EVENT_IDENTITY')                       │
└─────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────┐
│          EventController.createEvent                    │
├─────────────────────────────────────────────────────────┤
│  1. Validate request với Zod schema                     │
│  2. Lấy EVENT_IDENTITY từ middleware                    │
│  3. Enrich context:                                     │
│     - timestamp, received_at                            │
│     - event_date (YYYY-MM-DD)                           │
│     - user_agent, ip                                    │
│     - referrer, utm params                              │
│  4. EventQueue.add(eventRecord) ← MongoDB Queue         │
│     - Tạo document trong `events` collection            │
│     - queue_status: Pending                             │
│     - queue_created_at, queue_host, queue_platform      │
│     - Gọi notifyNewEvent() → wake worker                │
│  5. Return response ngay lập tức                        │
└─────────────────────────────────────────────────────────┘
       ↓ (async)
┌─────────────────────────────────────────────────────────┐
│               Event Worker (Background)                 │
├─────────────────────────────────────────────────────────┤
│  Loop:                                                  │
│  1. EventQueue.getNextEvent()                           │
│     - findOneAndUpdate({ queue_status: Pending })       │
│     - Set queue_status: Processing                      │
│  2. Nếu không có event:                                 │
│     - listenNewEvent() → wait cho notification          │
│  3. Process event:                                      │
│     - Normalize payload                                 │
│     - Validate data                                     │
│  4. EventQueue.updateStatus(Completed/Failed)           │
│     - Mark queue_status: Completed                      │
│     - Hoặc Failed + queue_error nếu lỗi                 │
│  5. Repeat                                              │
└─────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────┐
│             MongoDB `events` collection                 │
├─────────────────────────────────────────────────────────┤
│  Document structure:                                    │
│  - Event data: event_name, user_id, properties, ...     │
│  - Queue metadata:                                      │
│    * queue_status: Pending → Processing → Completed    │
│    * queue_created_at, queue_updated_at                 │
│    * queue_host, queue_platform                         │
│    * queue_error (nếu Failed)                           │
└─────────────────────────────────────────────────────────┘
```

---

## 🌐 API Endpoint

### Base URL

```
Development:  http://localhost:4200
Production:   https://api.petpet.vn
```

### Endpoint

```http
POST /api/events
```

**⚠️ LƯU Ý:** Không có `/v1` trong URL!

**Headers:**
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>  # Optional - nếu user đã login
```

**CURL Example:**
```bash
curl -X POST http://localhost:4200/api/events \
  -H "Content-Type: application/json" \
  -d '{
    "event_name": "product_view",
    "properties": {
      "product_id": 1234
    }
  }'
```

---

## 🔄 Request Flow

### 1. Request đến server
```javascript
fetch('http://localhost:4200/api/events', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    event_name: 'product_view',
    properties: { product_id: 123 }
  })
})
```

### 2. Middleware `eventIdentityMiddleware` xử lý

**Source code:** `src/middlewares/event_session.middleware.ts`

```typescript
export async function eventIdentityMiddleware(c: Context, next: Next) {
  const now = Date.now();
  const queryParams = c.req.query();

  // 1. Parse UTM từ query params
  const currentUtm = {
    source: queryParams?.utm_source ?? null,
    campaign: queryParams?.utm_campaign ?? null
  };

  // 2. Ensure anonymous_id (cookie 'aid' - 30 ngày)
  const anonymousId = ensureAnonymousId(c);
  
  // 3. Ensure session_id (cookie 'sid' - 30 phút hoặc 24h max)
  const sessionResult = ensureSession(c, currentUtm, now);

  // 4. Build identity object
  const identity: EventIdentity = {
    anonymousId,
    sessionId: sessionResult.sessionId,
    userId: null,  // Sẽ được set nếu có JWT token
    utm: sessionResult.meta.utm ?? currentUtm
  };

  // 5. Extract user_id từ JWT token (nếu có)
  const authorization = c.req.header('authorization');
  if (authorization?.startsWith('Bearer ')) {
    const token = authorization.split(' ')[1]?.trim();
    await resolveUserFromToken(c, token, identity, now);
  }

  // 6. Set vào context để controller dùng
  c.set('EVENT_IDENTITY', identity);

  await next();
}
```

**Cookies được set:**

| Cookie | Name | TTL | Purpose |
|--------|------|-----|---------|
| Anonymous ID | `aid` | 30 ngày | Track người dùng ẩn danh |
| Session ID | `sid` | 30 phút idle | Track session |
| Session Meta | `sid_meta` | 30 phút idle | Store UTM, timestamps |

**Session Rotation Rules:**
- ✅ Tạo session mới nếu:
  - Chưa có session
  - Session timeout (30 phút không có request)
  - Session quá 24 giờ kể từ lúc tạo
  - UTM params thay đổi (utm_source hoặc utm_campaign)

### 3. Controller `EventController.createEvent` xử lý

**Source code:** `src/controllers/event.controller.ts`

```typescript
static async createEvent(c: Context) {
  // 1. Parse & validate request body
  const body = await c.req.json();
  const validationResult = eventSchema.safeParse(body);
  if (!validationResult.success) {
    return c.json({ success: false, error: validationResult.error.issues }, 400);
  }

  const { event_name, timestamp, properties, context: payloadContext } = validationResult.data;

  // 2. Lấy identity từ middleware
  const identity = c.get("EVENT_IDENTITY") as EventIdentity;

  // 3. Build request context từ headers
  const requestContext = {
    user_agent: c.req.header("user-agent"),
    referer: c.req.header("referer") ?? c.req.header("origin") ?? null,
    ip: c.req.header("x-forwarded-for")?.split(",")[0]?.trim() ?? null
  };

  // 4. Merge contexts
  const enrichedContext = {
    ...requestContext,
    ...payloadContext,
    utm: identity?.utm
  };

  // 5. Build event record
  const now = new Date();
  const eventTimestamp = timestamp ?? now;
  const eventDate = new Date(eventTimestamp);
  eventDate.setUTCHours(0, 0, 0, 0);  // Midnight UTC

  const eventRecord = {
    event_name,
    user_id: identity?.userId ?? null,
    session_id: identity?.sessionId ?? crypto.randomUUID(),
    anonymous_id: identity?.anonymousId ?? crypto.randomUUID(),
    event_date: eventDate,
    timestamp: eventTimestamp,
    properties: properties ?? null,
    context: enrichedContext,
    received_at: now
  };

  // 6. Push vào MongoDB Queue
  await EventQueue.add(eventRecord);

  // 7. Return response NGAY LẬP TỨC (không chờ ghi DB)
  return c.json({
    success: true,
    data: {
      anonymous_id: eventRecord.anonymous_id,
      session_id: eventRecord.session_id,
      user_id: eventRecord.user_id
    }
  });
}
```

### 4. Queue đẩy vào MongoDB

**Source code:** `src/common/queue/EventQueue.ts`

```typescript
import { eventRepo } from "../../repositories/event.repository.js";
import { notifyNewEvent } from "./EventQueueEmitter.js";
import os from "os";

export enum EventStatus {
  Pending = "Pending",
  Processing = "Processing",
  Completed = "Completed",
  Failed = "Failed",
}

export class EventQueue {
  static async add(eventData: any) {
    const hostname = os.hostname();
    const platform = os.platform();
    
    const eventWithQueue = {
      ...eventData,
      queue_status: EventStatus.Pending,
      queue_created_at: new Date(),
      queue_updated_at: new Date(),
      queue_host: hostname,
      queue_platform: platform,
    };
    
    await eventRepo.create(eventWithQueue);
    notifyNewEvent(); // Wake worker
  }
}
```

### 5. Worker xử lý background

**Source code:** `src/workers/eventWorker.ts`

```typescript
import { listenNewEvent } from "../common/queue/EventQueueEmitter.js";
import { EventQueue, EventStatus } from "../common/queue/EventQueue.js";

export async function processEventQueue(): Promise<void> {
  console.log('Event Worker started...');
  let retry: number = 0;
  
  while (retry <= 3) {
    let event: any = null;
    
    try {
      console.log('Finding event..., retry::', retry);
      const nextEvent = await EventQueue.getNextEvent();

      if (!nextEvent?._id) {
        // Nếu không có event, wait cho notification
        await new Promise<void>((resolve) => {
          listenNewEvent(resolve);
        });
        continue;
      }
      
      retry = 0; // Reset retry khi tìm thấy event
      event = nextEvent;
      
      console.log(`Processing event ${event._id}: ${event.event_name}`);

      // Event đã được normalize và save vào MongoDB rồi
      // Chỉ cần mark là Completed
      await EventQueue.updateStatus(event._id, EventStatus.Completed);
      console.log(`Event ${event._id} completed.`);

    } catch (err: unknown) {
      console.error('Error processing event:', err);
      
      if (event?._id) {
        // Mark event là Failed với error message
        const errorMessage = (err instanceof Error) ? err.message : String(err);
        await EventQueue.updateStatus(event._id, EventStatus.Failed, errorMessage);
      } else {
        // Nếu lỗi trước khi tìm được event, increment retry
        retry++;
        if (retry > 3) {
          console.error('Event worker stopped after 3 failed retries');
          break;
        }
      }
    }
  }
}
```

**EventQueue Methods:**

```typescript
// Get next pending event (atomic operation)
static async getNextEvent() {
  const hostname = os.hostname();
  return await eventRepo.collection.findOneAndUpdate(
    { queue_status: EventStatus.Pending },
    { 
      $set: { 
        queue_status: EventStatus.Processing,
        queue_updated_at: new Date(),
        queue_host: hostname
      } 
    },
    { returnDocument: "after", sort: { queue_created_at: 1 } }
  );
}

// Update event status
static async updateStatus(id: ObjectId, status: EventStatus, error?: string) {
  const update: any = {
    queue_status: status,
    queue_updated_at: new Date(),
  };
  
  if (error) {
    update.queue_error = error;
  }
  
  await eventRepo.collection.updateOne(
    { _id: id },
    { $set: update }
  );
}
```

**Worker khởi động:** `src/workers/register.ts` được import trong `src/app.ts`

---

## 🔐 Authentication & Identity

### 1. Anonymous ID (`aid` cookie)

**Purpose:** Track người dùng ẩn danh  
**TTL:** 30 ngày (2,592,000 seconds)  
**Format:** UUID v4  
**Cookie Settings:**
```javascript
{
  path: '/',
  httpOnly: true,
  sameSite: 'Lax',
  secure: true,  // chỉ trong production
  maxAge: 2592000
}
```

**Auto-generation:**
```typescript
function ensureAnonymousId(c: Context): string {
  const existing = getCookie(c, 'aid');
  const anonymousId = isValidId(existing) ? existing : crypto.randomUUID();
  
  setCookie(c, 'aid', anonymousId, {...});
  return anonymousId;
}
```

### 2. Session ID (`sid` cookie)

**Purpose:** Track session người dùng  
**TTL:** 30 phút idle (1,800 seconds)  
**Max Lifetime:** 24 giờ  
**Format:** UUID v4

**Rotation Rules:**
```typescript
const shouldRotateForLifetime = now - meta.createdAt > 86400000; // 24h
const shouldRotateForUtm = hasUtmChanged(meta.utm, currentUtm);
const needsNewSession = !sessionId || shouldRotateForLifetime || shouldRotateForUtm;
```

**Session Meta (`sid_meta` cookie):**
```json
{
  "createdAt": 1729584000000,
  "lastSeen": 1729585800000,
  "utm": {
    "source": "facebook",
    "campaign": "summer-sale"
  }
}
```

Encoded as Base64 trong cookie.

### 3. User ID (từ JWT Token)

**Headers:**
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Token phải chứa:**
- `userId` hoặc `user_id` hoặc `uid`

**Validation flow:**
```typescript
async function resolveUserFromToken(c, token, identity, now) {
  // 1. Decode token
  const decoded = decodeToken(token);
  
  // 2. Check expiry
  if (decoded.exp && decoded.exp * 1000 < now) return;
  
  // 3. Verify với public key từ DB
  const keyRecord = await keyTokenService.findByUserId(decoded.userId, userAgent);
  const verified = verifyToken(token, keyRecord.publicKey);
  
  // 4. Set user_id
  if (verified?.userId === decoded.userId) {
    identity.userId = decoded.userId;
    c.set('AUTHORIZED_USER', verified);
  }
}
```

**⚠️ Frontend KHÔNG CẦN gửi `user_id` trong request body - Server tự extract từ JWT token!**

---

## 📝 Request Schema

### Zod Validation Schema

**Source:** `src/models/validation/event.ts`

```typescript
export const eventSchema = z.object({
  event_name: z.string().min(1),
  timestamp: z.preprocess(
    (val) => (val ? new Date(val as string) : undefined),
    z.date().optional()
  ),
  properties: z.record(z.any()).optional(),
  context: z.record(z.any()).optional(),
  anonymous_id: z.string().optional(),
  session_id: z.string().optional(),
  user_id: z.string().optional()
});
```

### Schema Rules

| Field | Type | Required | Auto-handled | Description |
|-------|------|----------|--------------|-------------|
| `event_name` | `string` | ✅ **Yes** | ❌ | Tên event (bắt buộc) |
| `timestamp` | `string` (ISO 8601) | ❌ No | ✅ | Server dùng `new Date()` nếu không có |
| `properties` | `object` | ❌ No | ❌ | Data riêng của event |
| `context` | `object` | ❌ No | ⚠️ Partially | Frontend có thể gửi, server sẽ merge thêm |
| `anonymous_id` | `string` | ❌ No | ✅ | Server dùng cookie `aid` |
| `session_id` | `string` | ❌ No | ✅ | Server dùng cookie `sid` |
| `user_id` | `string` | ❌ No | ✅ | Server extract từ JWT token |

### Minimal Request (Recommended)

```json
{
  "event_name": "product_view",
  "properties": {
    "product_id": 123,
    "product_name": "Royal Canin"
  }
}
```

### Full Request (với optional fields)

```json
{
  "event_name": "product_view",
  "timestamp": "2025-10-22T10:30:00.000Z",
  "properties": {
    "product_id": 123,
    "product_name": "Royal Canin",
    "price": 1250000,
    "currency": "VND"
  },
  "context": {
    "list_name": "recommendation_similar",
    "context_product_id": "prod_456"
  }
}
```

---

## ✅ Response Format

### Success Response

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": {
    "anonymous_id": "11111111-2222-3333-4444-555555555555",
    "session_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "user_id": "user_abc123"
  }
}
```

**Response Fields:**
- `anonymous_id`: UUID từ cookie `aid`
- `session_id`: UUID từ cookie `sid`
- `user_id`: User ID từ JWT token (hoặc `null` nếu anonymous)

### Error Response

**HTTP Status:** `400 Bad Request`

```json
{
  "success": false,
  "error": [
    {
      "code": "invalid_type",
      "expected": "string",
      "received": "undefined",
      "path": ["event_name"],
      "message": "Required"
    }
  ]
}
```

**HTTP Status:** `500 Internal Server Error`

```json
{
  "success": false,
  "error": "Queue error: Connection refused"
}
```

---

## 🎯 17 Event Types

### Event Naming Convention

**Pattern:** `{noun}_{verb}`

Examples:
- ✅ `product_view`, `product_click`
- ✅ `category_view`, `brand_view`
- ✅ `search_query`, `search_submit`
- ✅ `add_to_cart`, `remove_from_cart`

---

### 1️⃣ Navigation Events

#### 1.1. `page_view`

**Khi nào dùng:** Khi user vào một trang bất kỳ

**Properties:**
```typescript
{
  page_url: string;      // "/products/123"
  page_title: string;    // "Royal Canin Medium Adult"
  referrer?: string;     // "/categories/thuc-an"
}
```

**Request Example:**
```json
{
  "event_name": "page_view",
  "properties": {
    "page_url": "/products/12345",
    "page_title": "Royal Canin Medium Adult 15kg",
    "referrer": "/categories/thuc-an-cho-cho"
  }
}
```

**Frontend Code (Vue 3):**
```javascript
// pages/products/[id].vue
import { onMounted } from 'vue'

onMounted(() => {
  fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'page_view',
      properties: {
        page_url: window.location.pathname,
        page_title: document.title,
        referrer: document.referrer
      }
    })
  }).catch(console.error)
})
```

---

#### 1.2. `category_view`

**Properties:**
```typescript
{
  category_id: string;
  category_name: string;
  category_slug?: string;
}
```

**Request Example:**
```json
{
  "event_name": "category_view",
  "properties": {
    "category_id": "cat_123",
    "category_name": "Thức Ăn Cho Chó",
    "category_slug": "thuc-an-cho-cho"
  }
}
```

---

#### 1.3. `brand_view`

**Properties:**
```typescript
{
  brand_id: string;
  brand_name: string;
  brand_slug?: string;
}
```

**Request Example:**
```json
{
  "event_name": "brand_view",
  "properties": {
    "brand_id": "brand_456",
    "brand_name": "Royal Canin",
    "brand_slug": "royal-canin"
  }
}
```

---

### 2️⃣ Product Interaction Events

#### 2.1. `product_view`

**Khi nào dùng:** Khi user xem chi tiết sản phẩm

**Properties:**
```typescript
{
  product_id: string;
  product_name: string;
  price: number;
  currency: string;
  category?: string;
  brand?: string;
  product_type?: 'dog' | 'cat' | 'other';
}
```

**Context (Optional):**
```typescript
{
  context_product_id?: string;  // Sản phẩm trước đó
  list_name?: string;           // Nguồn: "search" | "category" | "recommendation"
}
```

**Request Example:**
```json
{
  "event_name": "product_view",
  "properties": {
    "product_id": "prod_789",
    "product_name": "Royal Canin Medium Adult 15kg",
    "price": 1250000,
    "currency": "VND",
    "category": "Thức Ăn > Thức Ăn Hạt",
    "brand": "Royal Canin",
    "product_type": "dog"
  },
  "context": {
    "context_product_id": "prod_123",
    "list_name": "recommendation_similar"
  }
}
```

**Frontend Code:**
```javascript
// pages/products/[id].vue
const route = useRoute()

onMounted(async () => {
  const product = await fetchProduct(route.params.id)
  
  await fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'product_view',
      properties: {
        product_id: product.id,
        product_name: product.name,
        price: product.price,
        currency: 'VND',
        category: product.category,
        brand: product.brand,
        product_type: product.type
      },
      context: {
        context_product_id: route.query.from_product,
        list_name: route.query.list || 'direct'
      }
    })
  })
})
```

---

#### 2.2. `product_click`

**Khi nào dùng:** Khi user click vào product card (trước khi navigate)

**Properties:**
```typescript
{
  product_id: string;
  position?: number;
  list_name?: string;
}
```

**Request Example:**
```json
{
  "event_name": "product_click",
  "properties": {
    "product_id": "prod_789",
    "position": 3,
    "list_name": "category_listing"
  }
}
```

**Frontend Code:**
```javascript
function handleProductClick(product, index, listName) {
  // Track click (fire-and-forget)
  fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'product_click',
      properties: {
        product_id: product.id,
        position: index,
        list_name: listName
      }
    })
  }).catch(console.error)
  
  // Navigate
  router.push(`/products/${product.id}?list=${listName}`)
}
```

---

### 3️⃣ Search Events

#### 3.1. `search_query`

**Khi nào dùng:** Mỗi lần user gõ từ khóa (debounce 500ms)

**Properties:**
```typescript
{
  query: string;
  results_count?: number;
}
```

**Request Example:**
```json
{
  "event_name": "search_query",
  "properties": {
    "query": "thức ăn cho chó",
    "results_count": 24
  }
}
```

**Frontend Code (với debounce):**
```javascript
import { ref, watch } from 'vue'
import { debounce } from 'lodash-es'

const searchQuery = ref('')
const resultsCount = ref(0)

const trackSearch = debounce((query, count) => {
  fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'search_query',
      properties: { query, results_count: count }
    })
  }).catch(console.error)
}, 500)

watch([searchQuery, resultsCount], ([query, count]) => {
  if (query.length > 2) {
    trackSearch(query, count)
  }
})
```

---

#### 3.2. `search_submit`

**Khi nào dùng:** Khi user nhấn Enter hoặc apply filters

**Properties:**
```typescript
{
  query: string;
  results_count: number;
  filters?: {
    category?: string;
    brand?: string;
    price_min?: number;
    price_max?: number;
    [key: string]: any;
  };
}
```

**Request Example:**
```json
{
  "event_name": "search_submit",
  "properties": {
    "query": "thức ăn cho chó",
    "results_count": 24,
    "filters": {
      "category": "thuc-an",
      "price_min": 100000,
      "price_max": 500000,
      "brand": "royal-canin"
    }
  }
}
```

---

### 4️⃣ Cart & Wishlist Events

#### 4.1. `add_to_cart`

**Properties:**
```typescript
{
  product_id: string;
  product_name: string;
  price: number;
  currency: string;
  quantity: number;
}
```

**Request Example:**
```json
{
  "event_name": "add_to_cart",
  "properties": {
    "product_id": "prod_789",
    "product_name": "Royal Canin Medium Adult 15kg",
    "price": 1250000,
    "currency": "VND",
    "quantity": 2
  }
}
```

**Frontend Code:**
```javascript
async function addToCart(product, quantity = 1) {
  // Add to cart API
  await cartAPI.add(product.id, quantity)
  
  // Track event
  await fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'add_to_cart',
      properties: {
        product_id: product.id,
        product_name: product.name,
        price: product.price,
        currency: 'VND',
        quantity
      }
    })
  })
}
```

---

#### 4.2. `remove_from_cart`

**Properties:**
```typescript
{
  product_id: string;
  quantity: number;
}
```

**Request Example:**
```json
{
  "event_name": "remove_from_cart",
  "properties": {
    "product_id": "prod_789",
    "quantity": 1
  }
}
```

---

#### 4.3. `add_to_wishlist`

**Properties:**
```typescript
{
  product_id: string;
  product_name: string;
  price: number;
  currency: string;
}
```

**Request Example:**
```json
{
  "event_name": "add_to_wishlist",
  "properties": {
    "product_id": "prod_789",
    "product_name": "Royal Canin Medium Adult 15kg",
    "price": 1250000,
    "currency": "VND"
  }
}
```

---

#### 4.4. `remove_from_wishlist`

**Properties:**
```typescript
{
  product_id: string;
}
```

**Request Example:**
```json
{
  "event_name": "remove_from_wishlist",
  "properties": {
    "product_id": "prod_789"
  }
}
```

---

### 5️⃣ Purchase Event

#### 5.1. `purchase`

**Khi nào dùng:** Khi user hoàn tất đơn hàng

**Properties:**
```typescript
{
  order_id: string;
  total_amount: number;
  currency: string;
  shipping_fee?: number;
  discount?: number;
  payment_method?: string;
  products: Array<{
    product_id: string;
    product_name: string;
    price: number;
    quantity: number;
    category?: string;
    brand?: string;
  }>;
}
```

**Request Example:**
```json
{
  "event_name": "purchase",
  "properties": {
    "order_id": "order_12345",
    "total_amount": 2750000,
    "currency": "VND",
    "shipping_fee": 30000,
    "discount": 250000,
    "payment_method": "vnpay",
    "products": [
      {
        "product_id": "prod_789",
        "product_name": "Royal Canin Medium Adult 15kg",
        "price": 1250000,
        "quantity": 2,
        "category": "Thức Ăn > Thức Ăn Hạt",
        "brand": "Royal Canin"
      }
    ]
  }
}
```

**Frontend Code:**
```javascript
// pages/checkout/success.vue
onMounted(async () => {
  const orderId = route.query.orderId
  const order = await fetchOrder(orderId)
  
  await fetch('http://localhost:4200/api/events', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      event_name: 'purchase',
      properties: {
        order_id: order.id,
        total_amount: order.total,
        currency: 'VND',
        shipping_fee: order.shipping,
        discount: order.discount,
        payment_method: order.payment_method,
        products: order.items.map(item => ({
          product_id: item.product_id,
          product_name: item.name,
          price: item.price,
          quantity: item.quantity,
          category: item.category,
          brand: item.brand
        }))
      }
    })
  })
})
```

---

### 6️⃣ Recommendation Events

#### 6.1. `recommendation_impression`

**Khi nào dùng:** Khi recommendation widget xuất hiện (50% visible)

**Properties:**
```typescript
{
  recommendation_id: string;
  algorithm: string;
  product_ids: string[];
  displayed_count: number;
}
```

**Context:**
```typescript
{
  context_product_id?: string;
  list_name: string;
}
```

**Request Example:**
```json
{
  "event_name": "recommendation_impression",
  "properties": {
    "recommendation_id": "rec_20241022_123456",
    "algorithm": "collaborative_filtering",
    "product_ids": ["prod_1", "prod_2", "prod_3", "prod_4"],
    "displayed_count": 4
  },
  "context": {
    "context_product_id": "prod_main_789",
    "list_name": "recommendation_similar"
  }
}
```

**Frontend Code (với Intersection Observer):**
```javascript
import { onMounted, ref } from 'vue'

const props = defineProps({
  recommendationId: String,
  algorithm: String,
  products: Array,
  contextProductId: String
})

const widgetRef = ref(null)

onMounted(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      fetch('http://localhost:4200/api/events', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          event_name: 'recommendation_impression',
          properties: {
            recommendation_id: props.recommendationId,
            algorithm: props.algorithm,
            product_ids: props.products.map(p => p.id),
            displayed_count: props.products.length
          },
          context: {
            context_product_id: props.contextProductId,
            list_name: `recommendation_${props.algorithm}`
          }
        })
      }).catch(console.error)
      
      observer.disconnect()
    }
  }, { threshold: 0.5 })
  
  if (widgetRef.value) {
    observer.observe(widgetRef.value)
  }
})
```

---

#### 6.2. `recommendation_click`

**Properties:**
```typescript
{
  recommendation_id: string;
  product_id: string;
  position: number;
  algorithm: string;
}
```

**Context:**
```typescript
{
  context_product_id?: string;
  list_name: string;
}
```

**Request Example:**
```json
{
  "event_name": "recommendation_click",
  "properties": {
    "recommendation_id": "rec_20241022_123456",
    "product_id": "prod_2",
    "position": 2,
    "algorithm": "collaborative_filtering"
  },
  "context": {
    "context_product_id": "prod_main_789",
    "list_name": "recommendation_similar"
  }
}
```

---

#### 6.3. `recommendation_add_to_cart`

**Properties:**
```typescript
{
  recommendation_id: string;
  product_id: string;
  quantity: number;
  algorithm: string;
}
```

**Request Example:**
```json
{
  "event_name": "recommendation_add_to_cart",
  "properties": {
    "recommendation_id": "rec_20241022_123456",
    "product_id": "prod_2",
    "quantity": 1,
    "algorithm": "collaborative_filtering"
  },
  "context": {
    "context_product_id": "prod_main_789"
  }
}
```

---

#### 6.4. `recommendation_add_to_wishlist`

**Properties:**
```typescript
{
  recommendation_id: string;
  product_id: string;
  algorithm: string;
}
```

**Request Example:**
```json
{
  "event_name": "recommendation_add_to_wishlist",
  "properties": {
    "recommendation_id": "rec_20241022_123456",
    "product_id": "prod_2",
    "algorithm": "collaborative_filtering"
  },
  "context": {
    "context_product_id": "prod_main_789"
  }
}
```

---

## 🌍 Context Enrichment

### Server tự động thêm các fields sau:

**Source code:** `src/controllers/event.controller.ts` - `buildRequestContext()`

```typescript
function buildRequestContext(c: Context): Record<string, unknown> | null {
  const userAgent = c.req.header("user-agent");
  const referer = c.req.header("referer") ?? c.req.header("origin") ?? null;
  const ipHeader = c.req.header("x-forwarded-for") ?? c.req.header("x-real-ip") ?? null;
  const ip = ipHeader ? ipHeader.split(",")[0]?.trim() || null : null;

  return {
    user_agent: userAgent,
    referer: referer,
    ip: ip
  };
}
```

### Context Fields

```typescript
{
  // Frontend có thể gửi:
  context_product_id?: string;
  list_name?: string;
  
  // Server tự động thêm:
  user_agent: string;        // "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
  referer: string | null;    // "https://petpet.vn/categories/thuc-an"
  ip: string | null;         // "123.45.67.89"
  utm: {
    source: string | null;   // từ query param utm_source
    campaign: string | null; // từ query param utm_campaign
  }
}
```

**⚠️ Frontend chỉ cần gửi context bổ sung:**

```json
{
  "context": {
    "context_product_id": "prod_123",
    "list_name": "recommendation_similar"
  }
}
```

Server sẽ merge với context tự động enrich.

---

## ⚙️ Queue & Worker System

### EventQueue (MongoDB)

**Location:** `src/common/queue/EventQueue.ts`

**Queue Architecture:**
- **Storage:** MongoDB `events` collection (không cần Redis riêng)
- **Status Tracking:** EventStatus enum (Pending → Processing → Completed/Failed)
- **Notification:** EventEmitter pattern để wake worker
- **Atomic Operations:** findOneAndUpdate để prevent race conditions

**EventQueue Class:**

```typescript
import { eventRepo } from "../../repositories/event.repository.js";
import { notifyNewEvent } from "./EventQueueEmitter.js";
import os from "os";

export enum EventStatus {
  Pending = "Pending",
  Processing = "Processing",
  Completed = "Completed",
  Failed = "Failed",
}

export class EventQueue {
  // Add new event to queue
  static async add(eventData: any) {
    const hostname = os.hostname();
    const platform = os.platform();
    
    const eventWithQueue = {
      ...eventData,
      queue_status: EventStatus.Pending,
      queue_created_at: new Date(),
      queue_updated_at: new Date(),
      queue_host: hostname,
      queue_platform: platform,
    };
    
    await eventRepo.create(eventWithQueue);
    notifyNewEvent(); // Wake worker ngay lập tức
  }
  
  // Get next pending event (atomic)
  static async getNextEvent() {
    const hostname = os.hostname();
    const result = await eventRepo.collection.findOneAndUpdate(
      { queue_status: EventStatus.Pending },
      { 
        $set: { 
          queue_status: EventStatus.Processing,
          queue_updated_at: new Date(),
          queue_host: hostname
        } 
      },
      { returnDocument: "after", sort: { queue_created_at: 1 } }
    );
    
    return result?.value || null;
  }
  
  // Update event status
  static async updateStatus(id: ObjectId, status: EventStatus, error?: string) {
    const update: any = {
      queue_status: status,
      queue_updated_at: new Date(),
    };
    
    if (error) {
      update.queue_error = error;
    }
    
    await eventRepo.collection.updateOne(
      { _id: id },
      { $set: update }
    );
  }
}
```

**EventQueueEmitter:**

```typescript
import { EventEmitter } from "events";

const eventEmitter = new EventEmitter();

export function notifyNewEvent(): void {
  eventEmitter.emit("newEvent");
}

export function listenNewEvent(callback: () => void): void {
  eventEmitter.once("newEvent", callback);
}
```

### Worker Process

**File:** `src/workers/eventWorker.ts`

```typescript
import { listenNewEvent } from "../common/queue/EventQueueEmitter.js";
import { EventQueue, EventStatus } from "../common/queue/EventQueue.js";

export async function processEventQueue(): Promise<void> {
  console.log('Event Worker started...');
  let retry: number = 0;
  
  while (retry <= 3) {
    let event: any = null;
    
    try {
      const nextEvent = await EventQueue.getNextEvent();

      if (!nextEvent?._id) {
        // Không có event → wait cho notification
        await new Promise<void>((resolve) => {
          listenNewEvent(resolve);
        });
        continue;
      }
      
      retry = 0;
      event = nextEvent;
      
      console.log(`Processing event ${event._id}: ${event.event_name}`);
      
      // Event đã được save vào MongoDB, chỉ cần mark Completed
      await EventQueue.updateStatus(event._id, EventStatus.Completed);

    } catch (err: unknown) {
      console.error('Error processing event:', err);
      
      if (event?._id) {
        const errorMessage = (err instanceof Error) ? err.message : String(err);
        await EventQueue.updateStatus(event._id, EventStatus.Failed, errorMessage);
      } else {
        retry++;
        if (retry > 3) break;
      }
    }
  }
}
```

**Worker Registration:** `src/workers/register.ts`

```typescript
import { processEventQueue } from "./eventWorker.js";

export function registerWorkers() {
  try {
    processEventQueue();
    console.log('Event Worker started...');
  } catch (error) {
    console.error('Worker error:', error);
  }
}
```

**Auto-start:** Imported in `src/app.ts`

```typescript
import './workers/register.js';
```

### Event Processing Flow

```
Controller.createEvent()
    ↓
await EventQueue.add(eventRecord)  ← Tạo document với queue_status: Pending
    ↓
notifyNewEvent()  ← Wake worker qua EventEmitter
    ↓
Return response ngay (non-blocking)
    ↓
    ↓ (Background processing)
    ↓
EventQueue.getNextEvent()  ← findOneAndUpdate (atomic)
    ↓
Set queue_status: Processing
    ↓
Normalize & validate event
    ↓
EventQueue.updateStatus(Completed)  ← Mark done
```

**Key Points:**
- ✅ **Non-blocking:** API trả về ngay sau khi tạo event
- ✅ **No external dependency:** Không cần Redis Cloud riêng
- ✅ **Atomic:** findOneAndUpdate prevents race conditions
- ✅ **Event-driven:** listenNewEvent() cho efficient waiting
- ✅ **Retry logic:** Worker tự động retry 3 lần nếu lỗi
- ✅ **Multi-host support:** Track hostname/platform cho distributed setup

---

## 💻 Implementation Guide

### Vue 3 Composable

**File:** `composables/useEventTracking.js`

```javascript
const API_ENDPOINT = 'http://localhost:4200/api/events'

async function trackEvent(eventName, properties = {}, context = {}) {
  try {
    const token = localStorage.getItem('access_token')
    const headers = {
      'Content-Type': 'application/json'
    }
    
    if (token) {
      headers['Authorization'] = `Bearer ${token}`
    }
    
    const response = await fetch(API_ENDPOINT, {
      method: 'POST',
      headers,
      body: JSON.stringify({
        event_name: eventName,
        properties,
        context
      })
    })
    
    if (!response.ok) {
      console.error('Event tracking failed:', await response.text())
    }
  } catch (error) {
    console.error('Event tracking error:', error)
  }
}

export function useEventTracking() {
  function trackPageView() {
    trackEvent('page_view', {
      page_url: window.location.pathname,
      page_title: document.title,
      referrer: document.referrer
    })
  }
  
  function trackProductView(product, contextProduct = null, listName = null) {
    trackEvent('product_view', {
      product_id: product.id,
      product_name: product.name,
      price: product.price,
      currency: 'VND',
      category: product.category,
      brand: product.brand,
      product_type: product.type
    }, {
      context_product_id: contextProduct?.id,
      list_name: listName
    })
  }
  
  function trackAddToCart(product, quantity = 1) {
    trackEvent('add_to_cart', {
      product_id: product.id,
      product_name: product.name,
      price: product.price,
      currency: 'VND',
      quantity
    })
  }
  
  function trackSearch(query, resultsCount, filters = {}) {
    trackEvent('search_submit', {
      query,
      results_count: resultsCount,
      filters
    })
  }
  
  function trackRecommendationImpression(recommendationId, products, algorithm, contextProductId) {
    trackEvent('recommendation_impression', {
      recommendation_id: recommendationId,
      algorithm,
      product_ids: products.map(p => p.id),
      displayed_count: products.length
    }, {
      context_product_id: contextProductId,
      list_name: `recommendation_${algorithm}`
    })
  }
  
  return {
    trackPageView,
    trackProductView,
    trackAddToCart,
    trackSearch,
    trackRecommendationImpression
  }
}
```

### Usage Examples

#### Product Detail Page

```vue
<script setup>
import { onMounted, ref } from 'vue'
import { useRoute } from 'vue-router'
import { useEventTracking } from '@/composables/useEventTracking'

const route = useRoute()
const { trackPageView, trackProductView, trackAddToCart } = useEventTracking()

const product = ref(null)

onMounted(async () => {
  product.value = await fetchProduct(route.params.id)
  
  // Track page view
  trackPageView()
  
  // Track product view
  const contextProductId = route.query.from_product
  const listName = route.query.list || 'direct'
  trackProductView(product.value, { id: contextProductId }, listName)
})

function handleAddToCart() {
  trackAddToCart(product.value, 1)
  // Add to cart logic...
}
</script>
```

#### Search Page

```vue
<script setup>
import { ref, watch } from 'vue'
import { debounce } from 'lodash-es'
import { useEventTracking } from '@/composables/useEventTracking'

const { trackSearch } = useEventTracking()
const searchQuery = ref('')
const results = ref([])
const filters = ref({})

const debouncedSearch = debounce(async () => {
  results.value = await searchAPI.search(searchQuery.value, filters.value)
  trackSearch(searchQuery.value, results.value.length, filters.value)
}, 500)

watch([searchQuery, filters], debouncedSearch, { deep: true })
</script>
```

#### Recommendation Widget

```vue
<script setup>
import { onMounted, ref } from 'vue'
import { useEventTracking } from '@/composables/useEventTracking'

const props = defineProps({
  recommendationId: String,
  algorithm: String,
  products: Array,
  contextProductId: String
})

const { trackRecommendationImpression } = useEventTracking()
const widgetRef = ref(null)

onMounted(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      trackRecommendationImpression(
        props.recommendationId,
        props.products,
        props.algorithm,
        props.contextProductId
      )
      observer.disconnect()
    }
  }, { threshold: 0.5 })
  
  observer.observe(widgetRef.value)
})
</script>

<template>
  <div ref="widgetRef">
    <ProductCard 
      v-for="product in products" 
      :key="product.id" 
      :product="product" 
    />
  </div>
</template>
```

---

## ✅ Best Practices

### 1. Error Handling

```javascript
// ✅ ĐÚNG - Log error nhưng không throw
async function trackEvent(eventName, properties) {
  try {
    await fetch('http://localhost:4200/api/events', {...})
  } catch (error) {
    console.error('Event tracking failed:', error)
    // KHÔNG throw để không block UI
  }
}

// ❌ SAI - Throw error sẽ block UI
async function trackEvent(eventName, properties) {
  const response = await fetch('http://localhost:4200/api/events', {...})
  if (!response.ok) throw new Error('Failed')  // ❌
}
```

### 2. Fire-and-Forget cho Navigation

```javascript
// ✅ ĐÚNG - Không await
function handleProductClick(product) {
  fetch('http://localhost:4200/api/events', {
    method: 'POST',
    body: JSON.stringify({
      event_name: 'product_click',
      properties: { product_id: product.id }
    })
  }).catch(console.error)
  
  router.push(`/products/${product.id}`)  // Navigate ngay
}

// ❌ SAI - Await sẽ delay navigation
async function handleProductClick(product) {
  await fetch('http://localhost:4200/api/events', {...})  // ❌ Chờ
  router.push(`/products/${product.id}`)
}
```

### 3. Debounce cho Search

```javascript
// ✅ ĐÚNG - Debounce 500ms
const trackSearch = debounce((query, count) => {
  trackEvent('search_query', { query, results_count: count })
}, 500)

watch(searchQuery, (query) => {
  if (query.length > 2) {
    trackSearch(query, resultsCount.value)
  }
})
```

### 4. Intersection Observer cho Impressions

```javascript
// ✅ ĐÚNG - Track khi 50% visible
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    trackRecommendationImpression(...)
    observer.disconnect()  // Disconnect sau khi track
  }
}, { threshold: 0.5 })

observer.observe(widgetRef.value)
```

### 5. Validate Properties

```javascript
// ✅ ĐÚNG - Validate trước khi track
function trackProductView(product) {
  if (!product?.id) {
    console.warn('Invalid product for tracking')
    return
  }
  
  trackEvent('product_view', {
    product_id: product.id,
    product_name: product.name || 'Unknown',
    price: product.price || 0
  })
}
```

### 6. Không Gửi Sensitive Data

```javascript
// ❌ SAI - Không gửi password, email, phone
trackEvent('user_login', {
  email: 'user@example.com',  // ❌ SAI
  password: '123456'           // ❌ SAI
})

// ✅ ĐÚNG - Chỉ gửi metadata
trackEvent('user_login', {
  login_method: 'email',
  user_type: 'customer'
})
```

---

## 💾 Database Schema

### MongoDB Collection: `events`

**Schema:**
```typescript
{
  _id: ObjectId,
  
  // Event data
  event_name: string,           // "product_view"
  user_id: string | null,       // "user_abc123"
  session_id: string,           // UUID
  anonymous_id: string,         // UUID
  event_date: Date,             // ISODate("2025-10-22T00:00:00.000Z")
  timestamp: Date,              // ISODate("2025-10-22T10:30:45.123Z")
  received_at: Date,            // ISODate("2025-10-22T10:30:45.200Z")
  properties: object | null,    // { product_id: 123, ... }
  context: object | null,       // { user_agent: "...", utm: {...} }
  
  // Queue metadata
  queue_status: string,         // "Pending" | "Processing" | "Completed" | "Failed"
  queue_created_at: Date,       // Khi event được tạo
  queue_updated_at: Date,       // Lần cuối update status
  queue_host: string,           // Hostname của server xử lý
  queue_platform: string,       // OS platform (linux, darwin, win32)
  queue_error?: string          // Error message nếu Failed
}
```

**Indexes (Recommended):**
```javascript
// Event querying
db.events.createIndex({ event_name: 1, event_date: -1 })
db.events.createIndex({ user_id: 1, event_date: -1 })
db.events.createIndex({ session_id: 1, event_date: -1 })
db.events.createIndex({ anonymous_id: 1, event_date: -1 })
db.events.createIndex({ event_date: -1 })

// Queue processing (IMPORTANT!)
db.events.createIndex({ queue_status: 1, queue_created_at: 1 })
```
```

**Query Examples:**

```javascript
// Get product views in last 7 days
db.events.find({
  event_name: 'product_view',
  event_date: { $gte: new Date('2025-10-15') }
})

// Get user journey by session
db.events.find({ 
  session_id: 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee' 
}).sort({ timestamp: 1 })

// Count events by type
db.events.aggregate([
  {
    $match: {
      event_date: { $gte: new Date('2025-10-15') }
    }
  },
  {
    $group: {
      _id: '$event_name',
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])

// Co-viewed products (for recommendation)
db.events.aggregate([
  {
    $match: {
      event_name: 'product_view',
      event_date: { $gte: new Date('2025-10-15') }
    }
  },
  {
    $group: {
      _id: '$session_id',
      products: { $push: '$properties.product_id' }
    }
  }
])
```

---

## 🐛 Troubleshooting

### 1. Events không được track?

**Check API endpoint:**
```bash
curl -X POST http://localhost:4200/api/events \
  -H "Content-Type: application/json" \
  -d '{"event_name": "test"}'
```

**Expected response:**
```json
{
  "success": true,
  "data": {
    "anonymous_id": "...",
    "session_id": "...",
    "user_id": null
  }
}
```

**Check console logs:**
```javascript
console.log('Tracking event:', { event_name, properties })
fetch('http://localhost:4200/api/events', {...})
  .then(res => res.json())
  .then(data => console.log('Track response:', data))
  .catch(err => console.error('Track error:', err))
```

### 2. Cookies không được set?

**Check DevTools > Application > Cookies:**
- `aid` - 30 ngày TTL
- `sid` - 30 phút TTL
- `sid_meta` - 30 phút TTL

**Nếu không có cookies:**
- Check CORS settings (origin, credentials)
- Check domain/path của cookies
- Check secure flag (production vs development)

### 3. Session bị reset liên tục?

**Nguyên nhân:**
- Session timeout sau 30 phút idle
- Session rotate sau 24 giờ
- UTM parameters thay đổi

**Debug:**
```bash
# Check sid_meta cookie
# Decode Base64:
echo "eyJjcmVhdGVkQXQiOjE3Mjk1ODQwMDAwMDB9" | base64 -d
```

### 4. User ID không được track?

**Check JWT token:**
```javascript
const token = localStorage.getItem('access_token')
if (token) {
  const payload = JSON.parse(atob(token.split('.')[1]))
  console.log('Token payload:', payload)
  // Phải có: userId, user_id, hoặc uid
}
```

**Gửi token trong header:**
```javascript
headers: {
  'Authorization': `Bearer ${token}`
}
```

### 5. CORS Error?

**Error message:**
```
Access to fetch at 'http://localhost:4200/api/events' from origin 'http://localhost:3000' 
has been blocked by CORS policy
```

**Solution:**

Server đã config CORS:
```typescript
app.use(`*`, cors({
  origin: '*',
  credentials: true,
  allowMethods: ['GET', 'HEAD', 'PUT', 'POST', 'DELETE', 'PATCH'],
}))
```

Nếu vẫn lỗi, check:
- Frontend có dùng `credentials: 'include'` không?
- Server có chạy không? (Port 4200)

### 6. Worker không xử lý events?

**Check worker logs:**
```bash
# Terminal output
Event Worker started...
Finding event..., retry:: 0
Processing event 67...abc: product_view
Event 67...abc completed.
```

**Check MongoDB events collection:**
```javascript
// Xem pending events
db.events.find({ queue_status: "Pending" })

// Xem processing events (có thể bị stuck)
db.events.find({ queue_status: "Processing" })

// Xem failed events
db.events.find({ queue_status: "Failed" })

// Reset stuck events
db.events.updateMany(
  { queue_status: "Processing" },
  { $set: { queue_status: "Pending" } }
)
```

**Check indexes:**
```javascript
db.events.getIndexes()
// Phải có: { queue_status: 1, queue_created_at: 1 }
```

### 7. Events bị duplicate?

**Nguyên nhân:**
- Frontend gọi API nhiều lần
- Multiple workers process cùng 1 event

**Solution:**
- Frontend debounce tracking calls
- EventQueue.getNextEvent() dùng findOneAndUpdate (atomic)

### 8. Performance Issues?

**Check queue status:**
```javascript
// Số lượng pending events
db.events.countDocuments({ queue_status: "Pending" })

// Events lâu nhất chưa xử lý
db.events.find({ queue_status: "Pending" })
  .sort({ queue_created_at: 1 })
  .limit(10)
```

**Solution:**
- Scale worker instances (multiple processes)
- Add index cho queue_status + queue_created_at
- Monitor failed events và retry

---

## 📞 Support & Resources

### API Documentation

- **Swagger UI:** http://localhost:4200/doc
- **OpenAPI Spec:** http://localhost:4200/api-doc

### Source Code Files

- **Route:** `src/routes/index.ts` (event routes)
- **Controller:** `src/controllers/event.controller.ts`
- **Middleware:** `src/middlewares/event_session.middleware.ts`
- **Queue:** `src/common/queue/EventQueue.ts`
- **Queue Emitter:** `src/common/queue/EventQueueEmitter.ts`
- **Worker:** `src/workers/eventWorker.ts`
- **Worker Register:** `src/workers/register.ts`
- **Validation:** `src/models/validation/event.ts`
- **Interfaces:** `src/models/interfaces/tracking_events.ts`
- **Repository:** `src/repositories/event.repository.ts`

### Contact

- **Backend Team:** [backend-team@example.com]
- **Slack:** #event-tracking
- **Project:** pepe_web/api

---

**Document Version:** 2.0.0  
**Last Updated:** January 2025  
**Based on:** Source Code Analysis (MongoDB Queue Migration)  
**Maintainer:** Backend Team  
**Major Changes:** Migrated from Bull (Redis) to MongoDB Queue

