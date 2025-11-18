# food-delivery-aggregator-report
System Design Document for Food-Delivery Aggregator Architecture
Food-Delivery Aggregator System – Full Technical Report
1. Executive Summary

The Food-Delivery Aggregator System is a large-scale platform that connects customers, restaurants, and couriers to facilitate seamless ordering, dispatching, and real-time delivery tracking. The platform focuses on 24/7 availability, low-latency location updates, optimized driver assignment, and geographically scalable architecture using microservices.

The objective is to provide:

A unified interface for customers to browse menus, place orders, and track deliveries.

Tools for restaurants to manage menus, prices, inventory, and incoming orders.

A dispatching system that assigns drivers using intelligent matching algorithms.

Real-time location streaming with <2s latency.

Scalable, fault-tolerant infrastructure capable of supporting millions of requests.

2. Problem Definition

The system solves three major challenges:

1. Inefficient Order Coordination

Customers need accurate delivery timelines and clear restaurant menus. Restaurants need to handle large order volumes efficiently.

2. Driver Dispatch Complexity

Assigning the best driver requires evaluating distance, route traffic, load balancing, and driver availability.

3. Real-Time Visibility

Customers expect second-by-second live tracking of the courier across the delivery route.

Thus, the system must deliver:

Real-time streaming of courier GPS data.

Scalable menu and order management.

Reliable dispatch at peak hours.

Fault tolerance and consistency.

3. Stakeholder Analysis
Stakeholder	Role	Needs
Customers	App/Web users	Browse menus, order food, track deliveries, secure payments
Restaurants	Sellers	Manage menus/orders, fulfillment dashboard
Couriers	Delivery workforce	Receive assignments, route navigation, earnings insights
Operations Team	Admin & Monitoring	Fraud detection, system health visibility, emergency handling
Business Owners	Product stakeholders	Revenue optimization, analytics
Developers / DevOps	Platform maintainers	Resilient architecture, maintainability
4. Functional Requirements
Customer App

Browse restaurants filtered by cuisine, rating, location.

View menus & prices.

Add items to cart.

Place and pay for orders.

Track order status: Placed → Accepted → Preparing → Ready → Picked → Delivered.

Live courier map tracking.

Restaurant Dashboard

Add/update menu items, prices, availability.

Accept/Reject orders.

Update preparation times.

Mark order ready for pickup.

Courier App

Receive delivery request.

Accept/Reject assignments.

Show best route (Google Maps/OSRM integration).

Upload status updates.

Stream location.

Backend Services

Menu Service

Order Service

Dispatch/Assignment Service

Delivery Tracking Service (WebSockets)

Payment Service

User Service (auth)

Notification Service (SMS, Push)

5. Non-Functional Requirements
Performance

Menu reads <100 ms (cached)

Order creation <200 ms

Live tracking <2 s latency

ETA calculation refresh <30 s

Availability

99.9% uptime (SLO)

Geo-redundant databases

Scalability

Auto-scaling microservices

CDN + Redis caching

Geo-sharded databases

Security

End-to-end encryption (HTTPS/TLS)

Role-based access control (RBAC)

OAuth2 + JWT

PCI-DSS compliance for payments

6. System Architecture Overview
Architecture Style: Microservices + Event-Driven + CQRS (optional)
Core Components

API Gateway

Identity Service

Menu Service

Order Service

Cart Service

Payment Service

Dispatch Service

Delivery Tracking Service (WebSocket)

Notification Service

Restaurants Admin Portal

Customer Mobile/Web App

Tech Stack (Recommended)

Backend: Node.js / Go / Java Spring Boot

Database: PostgreSQL + Redis + MongoDB (for events)

Streaming: Kafka / Pulsar

Cloud: Azure/AWS/GCP

Mapping: Google Maps / Mapbox

CDN: Cloudflare / CloudFront

7. Context Diagram

High-level interactions:

Customer App → API Gateway → (Order, Menu, Payment)
                ↑                ↑
Restaurant App →|                |→ Dispatch Service → Courier App
                ↓                ↓
            Notification     Tracking Service (WebSockets)


External systems:

Payment Gateway

Maps API

SMS Provider

8. Use Case Diagram

Key Actors: Customer, Restaurant, Courier, System Admin

Major Use Cases:

Customer → Place Order, Track Delivery

Restaurant → Manage Menu, Accept Orders

Courier → Accept Assignment, Update Location

System Admin → View Reports, Configure Platform

9. Dispatcher Design: Matching Algorithm

The dispatcher selects the best courier for each order.

Input Parameters

Courier availability

Distance from restaurant

Distance to customer

Traffic conditions

Historical delivery performance

Courier acceptance history

Current load (ongoing deliveries)

Matching Algorithms

Greedy Nearest-Driver

Fastest but less optimal in traffic.

Weighted Score-Based Model

Score = 0.4*(distance_to_restaurant) +
        0.3*(estimated_delivery_time) +
        0.2*(courier_rating) +
        0.1*(load)


ML-Based Assignment (Advanced)

Predicts best driver based on historical patterns.

Dispatch Flow

Order → "Ready for Pickup"

Dispatcher gets list of nearby drivers (<3 km)

Calculate scores

Push notification to top-ranked courier

Courier accepts or rejects

If reject → fallback to next best driver

10. Data Model
Orders Table
Field	Type	Description
order_id	UUID	Primary key
user_id	UUID	Customer
restaurant_id	UUID	Restaurant
items	JSONB	Cart items
total_price	float	Final bill
status	enum	placed/preparing/ready/picked/delivered
created_at	timestamp	order creation
eta	int	Estimated delivery time
Routes
Field	Description
courier_id	Driver
gps_lat, gps_lng	Real-time location
route_path	Polyline
last_updated	Timestamp
11. API Contracts & Examples
POST /orders

Request

{
  "user_id": "123",
  "restaurant_id": "456",
  "items": [
    {"id": 1, "qty": 2}
  ]
}


Response

{
  "order_id": "O789",
  "status": "placed",
  "eta": 30
}

GET /courier/location/{id}
{
  "courier_id": "C22",
  "lat": 17.4412,
  "lng": 78.3923,
  "timestamp": "2025-11-18T08:30:11Z"
}

12. Menu Caching & CDN Strategy
Why needed?

Menus change rarely but are read constantly → caching saves cost.

Layers

CDN Cache – HTML, static assets, menu snapshots (TTL 5–15 min)

Redis Cache – Real-time menu store, invalidated via events

Database – Long-term menu storage

Cache Invalidation

Whenever restaurant updates item

Event emitted → Cache bust in Redis + CDN purge

13. Rate Limiting & Backoff Strategies
Rate Limiting

API Gateway enforces:

100 req/min per customer

500 req/min per restaurant

300 req/min couriers (location updates)

Backoff Strategies

Exponential Backoff: For failed dispatch, retry after N^2 sec

Circuit Breaker: For unhealthy services

Token Bucket: For burst control on notifications

14. SLOs & Error Budgets
Service-Level Objectives (SLO)
Metric	Target	Notes
Availability	99.9%	Monthly
Order API latency	<200 ms p95	
Location update latency	<2 sec	
Menu read	<100 ms	Cached
Error Budget

For 99.9% availability:

Monthly allowable downtime = ~43 minutes

If consumed early → freeze new releases, focus on stability.

15. Scalability: Geo-Sharding & Hot Regions
Geo-Sharding

Split by regions:

North India Cluster

South India Cluster

East/West Clusters

Each shard has:

Regional DB

Regional Dispatch Engine

Local CDN edges

Hot-Region Handling

Areas like:

IT parks during lunch

City centers in evenings

Solutions:

Auto-scaling dispatch service

Prioritize cache-first reads

Faster courier assignment

Local WebSocket clusters

16. Maintainability, Security & Engineering Challenges
Maintainability

Service Ownership Model (SRE + Dev Teams)

CI/CD with automated tests

Centralized Observability Stack (Grafana, Prometheus)

Security

OAuth2 + JWT

PCI-DSS compliant payment flow

Geo-fencing for courier fraud detection

Engineering Challenges

Handling peak loads (festivals, rains)

Near real-time tracking at scale

Balancing accuracy vs speed in ETAs

Handling high write throughput in dispatch

Database replication lag in multi-region setups
