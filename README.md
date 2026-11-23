# Full-Stack System Design: Food Delivery App

**Template:** GreatFrontEnd RADIO Framework (Process)  
**Client Side:** GreatFrontEnd Principles | **Server/Data Side:** GeeksforGeeks Roadmap Principles

---

## R — Requirements Exploration

Before drawing architecture boxes, define what you are building, who it's for, and at what scale.

### 1. Functional Requirements (Core Scope)
* **Customer App:** Search/browse nearby restaurants & menus, filter by cuisine/rating, manage cart, checkout & pay, track live delivery status & driver location on a map.
* **Restaurant Portal:** Accept/reject incoming orders, update preparation status, manage menu items & availability.
* **Delivery Partner App:** Receive nearby delivery requests, accept/decline, stream real-time GPS location, mark order picked up/delivered.

### 2. Non-Functional Requirements (GFG Focus)
* **Scale:** 10M Daily Active Users (DAU), 1M concurrent users during lunch/dinner peak hours (high write load for GPS pings, high read load for browsing menus).
* **Latency:** Menu browsing $< 200\text{ms}$; live location updates $< 1\text{--}2\text{ seconds}$.
* **CAP Theorem Trade-offs:**
  * **Order Placement & Payments:** Consistency (CP) is critical (no double charges or lost orders).
  * **Browsing Restaurants & Live Driver Map:** Availability & Partition Tolerance (AP) is preferred (showing slightly stale menu prices or a 2-second-old driver pin is fine; crashing is not).

### 3. Client & Product Constraints (GreatFrontEnd Focus)
* **Devices & Networks:** Mobile-first (responsive web/PWA or React Native). Must handle flaky 3G/4G networks gracefully (especially for riders in elevators/basements and users on the go).
* **UX & Accessibility:** Clear order-status visibility, screen-reader support for checkout & live updates, localization (currency, regional languages).

---

## A — Architecture & High-Level Design (HLD)

```text
[ Customer App ]       [ Restaurant Portal ]       [ Driver App ]
       |                         |                       |
       +-------------------------+-----------------------+
                                 |
                        [ CDN + API Gateway ]
                     (Auth, SSL, Rate Limiting)
                                 |
            +--------------------+--------------------+
            |                    |                    |
   [ Core Services ]     [ Order & Payment ]   [ Location Service ]
   (Search/Menu API)       (ACID + Saga)       (WebSockets / SSE)
            |                    |                    |
     [ Redis Cache ]      [ PostgreSQL ]       [ Redis GeoHash ]
   (Hot Menus/Ratings)   (Orders/Users/Txn)    (Live Driver Pins)
            |                    |
   [ Elasticsearch ]       [ Kafka Queue ] ---> [ Notification Service ]
   (Fuzzy Dish Search)                          (Push / SMS / Email)
```

### 1. Client-Side Architecture (GreatFrontEnd Focus)
Design the frontend with clear separation of concerns rather than stuffing everything into UI components:
* **View / UI Layer:** Reusable, dumb presentation components (`RestaurantCard`, `MenuItemRow`, `CartDrawer`, `TrackingMap`).
* **Client State & Cache Layer:**
  * **Server-Cache Store (e.g., TanStack Query / RTK Query):** Caches fetched restaurant lists, menus, and order status with stale-while-revalidate policies.
  * **Client UI Store (e.g., Zustand / Redux):** Manages local cart draft, active delivery address, modal visibility, and search filters.
* **Network & Real-Time Controller:** Centralized API client with auth-token refresh interceptors, exponential backoff retries on flaky networks, and a WebSocket manager that auto-reconnects when the tracking screen is open.
* **Persistence Layer:** `localStorage` / `IndexedDB` to preserve the user's cart if the browser refreshes or loses connectivity.

### 2. Server-Side Architecture (GFG Focus)
* **API Gateway & Load Balancers:** Routes traffic, terminates SSL, validates JWT tokens, and applies rate limiting (e.g., Token Bucket algorithm to stop scraping or DDoS).
* **Microservices:**
  * **Restaurant & Search Service:** Reads from Elasticsearch (for typo-tolerant search like `"biriyani"`) and Redis (cached menus).
  * **Order Service:** Handles order lifecycle via PostgreSQL for strict ACID transactions.
  * **Payment Service:** Integrates with external gateways (Razorpay/Stripe) using webhooks.
  * **Location/Dispatch Service:** Ingests driver GPS pings every $3\text{--}5\text{ seconds}$ into Redis Geospatial indexes and matches orders to the nearest available driver.
* **Message Broker (Kafka):** Decouples slow operations from the checkout path. When an order succeeds, Order Service emits an `OrderPlaced` event $\rightarrow$ Notification Service sends push/SMS $\rightarrow$ Analytics updates metrics.

---

## D — Data Model & Low-Level Design (LLD)

### 1. Client-Side Data Model (GreatFrontEnd Focus)
Keep client state normalized (avoid deeply nested arrays so updating one item's quantity or stock status is $O(1)$) and strictly separate Server State from UI State:

```typescript
// Server State (managed by query cache)
interface Restaurant {
  id: string;
  name: string;
  rating: number;
  etaMinutes: number;
  isOpen: boolean;
}

// Client State (Cart + UI)
interface ClientStore {
  cart: {
    restaurantId: string | null;
    itemsById: Record<string, { menuItemId: string; name: string; price: number; qty: number }>;
  };
  ui: {
    selectedAddressId: string;
    isCartDrawerOpen: boolean;
    checkoutStatus: 'idle' | 'submitting' | 'error';
  };
}
```

### 2. Server-Side Database Schema & LLD (GFG Focus)
**Relational DB (PostgreSQL) for transactional integrity:**
* `Users` (`id`, `name`, `phone`, `default_address_id`)
* `Restaurants` (`id`, `name`, `lat`, `lng`, `is_active`) — indexed spatially (PostGIS) or paired with Redis Geo.
* `MenuItems` (`id`, `restaurant_id`, `name`, `price_paise`, `in_stock`)
* `Orders` (`id`, `user_id`, `restaurant_id`, `driver_id`, `status`, `total_amount`, `idempotency_key`, `created_at`)
* `OrderItems` (`id`, `order_id`, `menu_item_id`, `quantity`, `price_at_purchase`) — *Note: `price_at_purchase` is snapshot-copied so future menu price edits don't alter past receipts.*

**Low-Level Design (OOP & Design Patterns):**
* **State Pattern for `OrderStatus`:** Enforces valid transitions (`CREATED` $\rightarrow$ `PAID` $\rightarrow$ `ACCEPTED` $\rightarrow$ `PREPARING` $\rightarrow$ `OUT_FOR_DELIVERY` $\rightarrow$ `DELIVERED` or `CANCELLED`) so a driver cannot accidentally mark an unpaid order as delivered.
* **Strategy Pattern for `PricingEngine`:** Swappable fee rules (`StandardDeliveryFee`, `RainSurgePricing`, `SubscriptionFreeDelivery`).

---

## I — Interface Definition (APIs & Component Contracts)

### 1. Client $\leftrightarrow$ Server APIs (REST + WebSockets)

**Browse Nearby Restaurants (Cursor-based pagination for infinite scroll):**
```http
GET /api/v1/restaurants?lat=12.9716&lng=77.5946&cursor=rest_982&limit=20
```
> **Why Cursor over Offset?** If a restaurant opens/closes while the user scrolls, offset pagination (`?page=2`) causes duplicate or skipped restaurants. Cursor pagination avoids that.

**Place Order (with Idempotency Key):**
```http
POST /api/v1/orders
Headers: { "Idempotency-Key": "uuid-v4-generated-on-client" }
Body: {
  "restaurantId": "rest_101",
  "addressId": "addr_55",
  "items": [{ "menuItemId": "item_9", "quantity": 2 }],
  "paymentMethod": "UPI"
}

Response 201: { "orderId": "ord_778", "status": "PAYMENT_PENDING", "paymentIntent": "..." }
```

**Live Order Tracking (WebSocket / SSE):**
* **Why SSE (Server-Sent Events) or WebSockets instead of short polling?** Short polling every 2 seconds wastes battery and floods servers.
* Driver app pushes coordinates via WebSocket/HTTP; Customer app subscribes to `GET /api/v1/orders/ord_778/stream` (SSE) to receive live `STATUS_UPDATE` and `DRIVER_LOCATION` events.

### 2. Internal UI Component API (GreatFrontEnd Focus)
Define clean contracts for core components:

```typescript
interface MenuItemRowProps {
  item: MenuItem;
  quantityInCart: number;
  onAdd: (itemId: string) => void;
  onRemove: (itemId: string) => void;
  disabled?: boolean; // e.g., when restaurant is closed
}
```

---

## O — Optimizations & Deep Dive

### 1. Client-Side Optimizations (GreatFrontEnd Focus)
* **Optimistic UI Updates:** When a user taps `"+"` on a food item, increment the cart counter immediately in local state before the API responds. Roll back with a toast notification only if the server returns an error (e.g., `"Item just sold out"`).
* **Search Debouncing & Request Cancellation:** Debounce search typing by $300\text{ms}$ and use `AbortController` to cancel outdated in-flight search requests so old results don't overwrite newer ones (preventing race conditions).
* **Rendering Performance:**
  * **List Virtualization** (e.g., `react-window`) for massive menus ($500+$ dishes) so the DOM stays lightweight.
  * **Code Splitting & Lazy Loading:** Load heavy Map SDKs (Google Maps / Mapbox) only when the user enters the Address Picker or Tracking page—don't bloat the initial bundle.
* **Image Optimization:** Serve resized WebP/AVIF food photos via CDN with `loading="lazy"` and skeleton placeholders to prevent Cumulative Layout Shift (CLS).
* **Accessibility (a11y):** Use `aria-live="polite"` regions on the tracking page so screen readers announce `"Your driver has picked up the order"` automatically, and trap keyboard focus inside the Cart/Checkout modal.

### 2. Server-Side Optimizations & Edge Cases (GFG Focus)
* **Preventing Double Orders (Idempotency):** On shaky mobile networks, a user might tap `"Pay"` twice. The backend stores the client's `Idempotency-Key` in Redis/Postgres; duplicate requests return the original order instead of creating a second one.
* **Handling Flash Traffic / Hot Keys:** During IPL finals or New Year's Eve, popular restaurants get hammered. Cache restaurant menus in Redis using Cache-Aside with a short TTL + event-driven invalidation whenever the restaurant edits a dish.
* **Database Scaling:**
  * **Read Replicas:** Route all browsing/history reads to read replicas; keep the primary DB strictly for writes (orders/payments).
  * **Sharding:** Shard the Orders and Location services geographically by `city_id` (since a user in Bengaluru never queries drivers in Mumbai).
* **Graceful Degradation:** If the Recommendation Engine fails, fall back to a static list of top-rated nearby restaurants instead of failing the home screen.
