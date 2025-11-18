# Food Delivery System (System Design)

Cloud-native, event-driven food delivery system that connects consumers,
restaurants, and couriers. This repository contains the high-level architecture
and order lifecycle for the system.

## System Requirements

- Consumers order food via a mobile application.
- Users can see restaurants within a 10 km radius.
- All payments are made in-app; no cash payments.
- Couriers are centrally managed by system.
- Restaurants notify YS when an order is ready for pickup.
- If no courier is at the restaurant, system assigns the nearest available courier.
- Delivery must not exceed 30 minutes.
- Late deliveries are refunded; cost distribution depends on the cause.
- Restaurants can cancel if they cannot meet the SLA.
- All order status changes must trigger real-time notifications.

These constraints shape the architectural decisions.

## High-level architecture (high-level-architecture.mmd)

### What the diagram shows

- **Clients → API Gateway**

  - `Mobile App` talks only to an `API Gateway`, which handles routing, auth, and request shaping.

- **Microservices**

  - **User Service → PostgreSQL - Users**: user profiles, addresses, preferences.
  - **Restaurant Service → PostgreSQL - Restaurants**: restaurant metadata, opening hours, configs.
  - **Menu Service → MongoDB - Menus**: flexible, nested menu and options stored as documents.
  - **Order Service → PostgreSQL - Orders**: transactional order data and status history.
  - **Payment Service → PostgreSQL - Payments**: strongly consistent financial records.
  - **Courier Service → Redis/DynamoDB - Courier Locations**: fast read/write of live courier locations.
  - **Notification Service**: sends push / realtime notifications based on domain events.

- **Realtime & geo**

  - `Courier Service` also writes to `Redis GEO / Elastic Geo` for “nearest courier” queries and fast map updates.

- **Event streaming**
  - `Kafka Event Bus` receives events such as `order_created`, `payment_success`, `restaurant_accepts`, `courier_assigned`.
  - Multiple services (e.g. `Notification Service`, analytics, monitoring) can consume these events independently.

### Why it is designed this way

- **Clear bounded contexts**: each domain (users, restaurants, menus, orders, payments, couriers, notifications) has its own service + database, matching team and ownership boundaries.
- **Polyglot persistence**: PostgreSQL for relational/transactional data, MongoDB for flexible menus, Redis/geo store for low‑latency location search.
- **API Gateway**: a single entry point for the mobile app so we can evolve internal services without breaking the client.
- **Event‑driven core with Kafka**: producers don’t know about consumers; new consumers (e.g. analytics, refunds, fraud detection) can be added by subscribing to existing topics.
- **Scalability & resilience**: each service can scale independently; Kafka buffers spikes so slow consumers don’t break the order flow.

## Order lifecycle (order-lifecycle.mmd / order-lifecycle.svg)

### What the diagram shows

1. **Mobile App → API Gateway: Create Order Request**
   - User submits an order from the app.
2. **API Gateway → Order Service: Validate & Initialize Order**
   - `Order Service` validates the request and creates an initial order record.
3. **Order Service → Payment Service: Charge Payment**
   - Synchronous call; the user must know immediately if the payment succeeds.
4. **Payment Service → Order Service: Payment Success**
   - Order moves to a paid state.
5. **Order Service → Kafka: order_created**
   - Order creation is published as an event.
6. **Kafka → Restaurant Service: new_order**
   - Restaurant sees the new order and decides to accept or reject.
7. **Restaurant Service → Kafka: restaurant_accepts**
   - Acceptance is emitted as an event.
8. **Kafka → Courier Service: assign_courier**
   - Courier assignment is triggered asynchronously.
9. **Courier Service → Kafka: courier_assigned**
   - Assigned courier is published as an event.
10. **Kafka → Notification Service: Push status update events**
    - Notification service consumes all relevant events.
11. **Notification Service → Mobile App: Notifications**
    - User receives realtime updates (accepted, assigned, delivered, etc.).

### Why the lifecycle works this way

- **Synchronous only where needed**: payment is blocking, everything else (restaurant accept, courier assignment, notifications) is event‑driven and async to keep latency low.
- **Event‑driven orchestration**: services react to events instead of one giant orchestrator service, which makes it easier to add new behaviours without changing existing code.
- **Good UX by design**: users see fast “order placed” feedback plus live status updates driven by the event stream.
- **Operational robustness**: if a downstream service is slow or briefly unavailable, Kafka keeps events until it catches up, instead of failing the whole order flow.

## Diagrams

- [High Level Architecture](diagrams/exports/high-level-architecture.svg)
- [Order Lifecycle](diagrams/exports/order-lifecycle.svg)
