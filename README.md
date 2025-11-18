# Food Delivery System (System Design)

Cloud-native, event-driven food delivery system that connects consumers,
restaurants, and couriers. This repository contains the high-level architecture,
order lifecycle, database design, and microservice decomposition for the system.

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

## Architecture Overview

Key architectural principles:

- API Gateway as the single entry point.
- Microservices for business domain isolation.
- Kafka for event streaming and decoupled communication.
- Redis Geo / Elastic Geo for low-latency distance calculations.
- PostgreSQL, MongoDB, and Redis selected based on workload patterns.
- Stateless services with horizontal scaling.

## Diagrams

- [High Level Architecture](diagrams/exports/high-level-architecture.svg)
- [Order Lifecycle](diagrams/exports/order-lifecycle.svg)
