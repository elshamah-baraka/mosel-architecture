# 03. High-Level Architecture

## 1. Purpose

This document describes the high-level software architecture of MOSEL.

It defines the major architectural layers, domain modules, infrastructure components, communication patterns, data flow, and boundaries that govern how the platform operates.

The objective is to provide a clear architectural model without exposing implementation-specific source code or proprietary business logic.

MOSEL is currently designed as a **modular monolith**: a single deployable backend application composed of independently structured business modules with explicit responsibilities and boundaries.

This approach provides the simplicity and consistency of a monolithic deployment while preserving the modularity required for future evolution.

---

# 2. Architectural Overview

At a high level, MOSEL consists of five major architectural areas:

```text
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT APPLICATIONS                    │
│                                                             │
│             Web Application / Mobile Clients               │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               │ HTTPS / REST
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                         API LAYER                           │
│                                                             │
│        Routing → Middleware → Controllers → DTOs           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      DOMAIN MODULES                         │
│                                                             │
│  Business │ Booking │ Commerce │ Feed │ Promotion │ ...    │
│                                                             │
│        Domain-specific services and business rules          │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
               ▼                               ▼
┌────────────────────────────┐     ┌─────────────────────────┐
│      DATA ACCESS           │     │    INFRASTRUCTURE       │
│                            │     │                         │
│      Prisma ORM            │     │        Redis            │
└──────────────┬─────────────┘     │        Jobs             │
               │                   │        Events            │
               ▼                   │        External APIs    │
┌────────────────────────────┐     └─────────────────────────┘
│        PostgreSQL          │
│                            │
│       Primary Database     │
└────────────────────────────┘
```

The architecture separates external communication, application orchestration, business logic, persistence, and infrastructure concerns.

---

# 3. Architectural Style

## Modular Monolith

MOSEL intentionally avoids starting with a distributed microservice architecture.

The backend is deployed as a single application, but its internal structure is divided into well-defined business modules.

Conceptually:

```text
                    MOSEL BACKEND
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
    Business          Booking         Commerce
        │                │                │
        ├────────────┐   │   ┌────────────┤
        │            │   │   │            │
        ▼            ▼   ▼   ▼            ▼
      Feed       Promotion   Notification
```

The important distinction is that **deployment boundaries and architectural boundaries are not the same thing**.

MOSEL currently has one deployment boundary while maintaining multiple logical domain boundaries.

This allows the system to remain operationally simple without sacrificing internal structure.

---

# 4. Why a Modular Monolith?

A microservice architecture introduces significant operational complexity:

* service discovery
* distributed tracing
* network communication
* deployment orchestration
* independent service versioning
* distributed transactions
* failure propagation
* increased infrastructure requirements

For an early-stage platform, introducing these concerns before they are necessary can create more problems than it solves.

MOSEL therefore follows a progression:

```text
Modular Monolith
       │
       │ Growth
       ▼
Identify Scaling Boundaries
       │
       ▼
Extract Independent Services
       │
       ▼
Distributed Architecture
```

The architecture is therefore **microservice-ready without being microservice-dependent**.

A module should only become an independent service when there is a concrete engineering reason to do so.

Examples could include:

* significantly different scaling requirements
* independent deployment requirements
* computationally expensive workloads
* organizational ownership boundaries
* infrastructure isolation requirements

---

# 5. Layered Application Architecture

Within the backend, requests follow a consistent layered structure.

```text
┌───────────────────────────────┐
│            Routes             │
│      HTTP endpoint mapping    │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│         Controllers           │
│   HTTP request/response layer │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│           Services            │
│      Business logic layer     │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│        Data Access            │
│        Prisma / ORM           │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│          PostgreSQL           │
│       Persistent storage      │
└───────────────────────────────┘
```

Each layer has a specific responsibility.

---

## 5.1 Routes

Routes define the public HTTP surface of the application.

Responsibilities include:

* mapping HTTP methods to handlers
* organizing API endpoints
* attaching middleware
* defining resource paths

Routes should not contain business logic.

---

## 5.2 Controllers

Controllers represent the HTTP boundary of the application.

They are responsible for:

* receiving HTTP requests
* extracting request data
* invoking appropriate services
* translating application results into HTTP responses
* returning appropriate status codes

Controllers should remain thin.

They should coordinate the request rather than implement business rules.

---

## 5.3 Services

Services contain the application's business logic.

This is where decisions such as:

* whether a booking can be created
* whether inventory is available
* how a promotion is ranked
* how a commerce transaction is processed
* whether a user is authorized for an operation

are made.

The service layer is therefore the primary boundary between HTTP concerns and domain behavior.

---

## 5.4 Data Access

The data access layer abstracts persistence operations from business logic.

MOSEL uses Prisma ORM to interact with PostgreSQL.

This provides a structured interface for:

* querying data
* creating records
* updating records
* relational operations
* transactions
* schema migrations

Business services should interact with persistence through defined data-access operations rather than embedding raw database concerns throughout the application.

---

# 6. Domain Modules

The backend is divided into business-oriented modules.

A simplified representation is:

```text
MOSEL
│
├── Identity & Access
│
├── Business Management
│
├── Service Marketplace
│
├── Booking
│
├── Product Commerce
│
├── Orders & Delivery
│
├── Wallet & Payments
│
├── Promotions
│
├── Feed
│
├── Recommendations
│
├── Notifications
│
├── Analytics
│
└── Administration
```

Each module owns a specific business capability.

The purpose of modularization is not simply to create folders.

A module represents a **boundary of responsibility**.

---

# 7. Module Responsibilities

## Identity & Access

Responsible for:

* users
* authentication
* authorization
* roles
* permissions
* session-related concerns

---

## Business Management

Responsible for:

* business profiles
* business types
* categories
* operating configuration
* business ownership
* business availability

---

## Service Marketplace

Responsible for:

* services
* pricing
* service availability
* service discovery
* service configuration

---

## Booking

Responsible for:

* appointment creation
* availability
* time slots
* booking state
* conflict prevention
* cancellation
* rescheduling

The booking module must preserve scheduling consistency even when multiple requests occur concurrently.

---

## Commerce

Responsible for:

* orders
* payments
* wallets
* ledger operations
* escrow
* commissions
* refunds
* payouts

Commerce is treated as a financial domain rather than simply an order-processing feature.

---

## Promotions

Responsible for:

* promotional campaigns
* budgets
* bids
* campaign state
* impressions
* clicks
* promotion ranking

---

## Feed

Responsible for:

* posts
* engagement
* feed generation
* content ranking
* content visibility

---

## Recommendation

Responsible for:

* recommendation signals
* business ranking
* behavioral signals
* relevance scoring
* personalization

---

## Notifications

Responsible for:

* notification creation
* notification delivery
* event-driven communication
* user notification state

---

## Analytics

Responsible for collecting and aggregating operational and behavioral signals.

Examples include:

* impressions
* clicks
* engagement
* conversion
* booking activity
* commerce activity
* session behavior

---

# 8. Module Interaction

Modules are not completely isolated.

Real-world marketplace operations require domains to collaborate.

For example, a booking may involve:

```text
Customer
   │
   ▼
Booking
   │
   ├──────► Business
   │
   ├──────► Service
   │
   ├──────► Availability
   │
   └──────► Commerce
```

A completed booking can subsequently produce events or state changes consumed by other domains:

```text
Booking Completed
        │
        ├────► Commerce
        │
        ├────► Notifications
        │
        ├────► Analytics
        │
        └────► Recommendation Signals
```

This allows the system to maintain domain separation while still supporting cross-domain workflows.

---

# 9. Synchronous vs Asynchronous Work

Not every operation should execute within the critical HTTP request path.

MOSEL distinguishes between operations that require an immediate response and operations that can be processed asynchronously.

### Synchronous operations

Examples:

* authentication
* retrieving a business
* creating a booking
* checking availability
* retrieving an order

These require an immediate result.

### Asynchronous operations

Examples:

* notification delivery
* analytics processing
* feed recalculation
* recommendation updates
* background maintenance
* non-critical external integrations

Conceptually:

```text
HTTP Request
     │
     ▼
Core Business Operation
     │
     ├──────────────► Immediate Response
     │
     └──────────────► Event / Job
                           │
                           ▼
                    Background Worker
```

This prevents non-critical work from unnecessarily increasing API response latency.

---

# 10. Data Architecture

PostgreSQL serves as the primary persistent data store.

The relational model provides strong consistency for business-critical operations such as:

* bookings
* orders
* payments
* wallets
* ledger transactions
* users
* businesses

Redis provides a complementary role for workloads where low-latency access is important.

Potential use cases include:

* caching
* session-related data
* frequently accessed resources
* rate limiting
* temporary state
* ranking and analytics support

The two systems therefore serve different architectural purposes.

```text
                  Application
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
       Redis                  PostgreSQL
    Fast / Temporary        Persistent / Durable
```

---

# 11. Request Lifecycle

A typical REST request follows the following path:

```text
Client
  │
  │ HTTPS
  ▼
HTTP Server
  │
  ▼
Middleware
  │
  ├── Authentication
  ├── Authorization
  ├── Validation
  └── Request Context
  │
  ▼
Router
  │
  ▼
Controller
  │
  ▼
Service
  │
  ├──────────────► Redis
  │
  └──────────────► Prisma
                       │
                       ▼
                   PostgreSQL
                       │
                       ▼
                  Service Result
                       │
                       ▼
                  Controller
                       │
                       ▼
                  HTTP Response
                       │
                       ▼
                     Client
```

This flow creates a predictable separation between transport concerns and business behavior.

---

# 12. Cross-Cutting Concerns

Certain capabilities operate across multiple modules.

These are treated as cross-cutting concerns rather than being duplicated inside individual business domains.

Examples include:

* authentication
* authorization
* validation
* error handling
* logging
* rate limiting
* auditing
* configuration
* observability

Conceptually:

```text
                 Cross-Cutting Infrastructure
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
     Booking           Commerce            Feed
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                     Shared Platform
```

The goal is to maintain consistent behavior without creating tightly coupled domain dependencies.

---

# 13. Error Handling

Errors are treated as part of the API contract.

The architecture distinguishes between categories such as:

* validation errors
* authentication failures
* authorization failures
* resource-not-found errors
* business rule violations
* conflict errors
* infrastructure failures
* unexpected application errors

A centralized error-handling strategy ensures that clients receive predictable responses while internal implementation details remain protected.

---

# 14. Transaction Boundaries

Some operations require multiple database changes to succeed or fail as a single unit.

Examples include:

```text
Create Booking
    │
    ├── Reserve Availability
    ├── Create Booking
    └── Create Commerce Transaction
```

or:

```text
Payment
    │
    ├── Record Transaction
    ├── Update Wallet
    └── Create Ledger Entry
```

These workflows require carefully defined transaction boundaries to prevent partial state.

Database transactions are therefore used where atomicity is required.

The objective is to ensure that the system never reports a successful business operation while leaving related state inconsistent.

---

# 15. Concurrency Considerations

Marketplace systems frequently operate under concurrent access.

Two customers may attempt to book the same time slot.

Multiple requests may update the same resource.

A payment and refund may occur against related financial records.

The architecture therefore treats concurrency as a first-class design concern.

Important mechanisms include:

* database transactions
* unique constraints
* optimistic or pessimistic concurrency strategies where appropriate
* atomic updates
* idempotent operations
* state-transition validation

The objective is to preserve domain invariants even when multiple requests execute simultaneously.

---

# 16. Security Boundary

The API represents the primary boundary between external clients and internal business logic.

The architecture therefore assumes that all external input is untrusted.

The request boundary includes controls such as:

```text
External Request
      │
      ▼
Authentication
      │
      ▼
Authorization
      │
      ▼
Validation
      │
      ▼
Business Rules
      │
      ▼
Persistence
```

This prevents unvalidated external data from directly reaching business-critical operations.

Security concerns are documented in greater detail in the dedicated security architecture document.

---

# 17. Scalability Strategy

MOSEL's scalability strategy is evolutionary rather than premature.

The first stage focuses on maintaining a clean modular monolith.

```text
Stage 1
Modular Monolith
      │
      ▼
Stage 2
Horizontal Scaling
      │
      ▼
Stage 3
Caching + Background Processing
      │
      ▼
Stage 4
Identify Bottleneck Domains
      │
      ▼
Stage 5
Selective Service Extraction
```

Not every module needs to become a separate service.

A future extraction would be driven by measurable requirements such as:

* traffic
* latency
* resource consumption
* deployment independence
* failure isolation
* organizational ownership

This prevents distributed architecture from becoming an end in itself.

---

# 18. Observability

As the platform grows, operational visibility becomes essential.

The architecture is designed to support:

* structured logging
* application metrics
* request tracing
* database performance monitoring
* cache metrics
* background job monitoring
* business-level metrics

Technical metrics alone are insufficient for a marketplace.

Business metrics such as:

* booking conversion
* order completion
* promotion CTR
* payment failures
* recommendation engagement

can provide additional insight into system behavior.

---

# 19. External Integrations

MOSEL is designed to communicate with external infrastructure and third-party services through controlled integration boundaries.

Potential integrations include:

* payment providers
* notification providers
* cloud infrastructure
* mapping and geolocation services
* authentication providers
* analytics systems

External dependencies should not leak directly into core domain logic.

Instead, integration boundaries should isolate provider-specific behavior from the rest of the application.

Conceptually:

```text
MOSEL Domain
     │
     ▼
Integration Interface
     │
     ▼
External Provider
```

This allows external providers to be replaced without requiring widespread changes throughout the platform.

---

# 20. Architectural Decision Summary

The major architectural decisions can be summarized as follows:

| Decision                           | Rationale                                   |
| ---------------------------------- | ------------------------------------------- |
| Modular Monolith                   | Balance simplicity and modularity           |
| REST API                           | Predictable client-server communication     |
| Layered Architecture               | Separation of concerns                      |
| Domain Modules                     | Business capability isolation               |
| PostgreSQL                         | Strong relational consistency               |
| Prisma                             | Structured data-access abstraction          |
| Redis                              | Low-latency caching and temporary state     |
| Transactions                       | Preserve business consistency               |
| DTOs & Validation                  | Protect application boundaries              |
| Asynchronous Processing            | Keep non-critical work out of request paths |
| Explicit Module Boundaries         | Enable future service extraction            |
| Centralized Cross-Cutting Concerns | Consistent platform behavior                |

---

# 21. Architectural Trade-Offs

No architecture is universally optimal.

MOSEL's current architecture deliberately accepts certain trade-offs.

### Advantages

* simpler deployment
* strong transactional consistency
* lower infrastructure overhead
* easier local development
* easier debugging
* shared data access
* clear domain organization
* straightforward scaling during early growth

### Limitations

* modules share the same deployment lifecycle
* a failure in the application can affect multiple domains
* individual modules cannot scale independently
* strict module boundaries must be maintained through engineering discipline
* very high-growth domains may eventually require extraction

These limitations are accepted because the current architecture prioritizes simplicity without sacrificing future evolution.

---

# 22. Evolution Path

The architecture is intentionally designed to evolve.

A possible future topology could become:

```text
                       API Gateway
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   Core Platform       Commerce          Discovery
        │                  │                  │
        ▼                  ▼                  ▼
   PostgreSQL          Financial DB       Search/Cache
        │
        └──────────────┬─────────────────────┐
                       │                     │
                       ▼                     ▼
                   Event Bus            Background Jobs
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
        Analytics   Feed      Notifications
```

This is not the current deployment architecture.

It represents a potential evolution path if system scale, organizational requirements, or workload characteristics justify increased distribution.

The key principle is:

> **Architecture should evolve in response to constraints, not assumptions.**

---

# 23. Summary

MOSEL uses a modular monolith architecture to establish strong internal boundaries while maintaining operational simplicity.

The architecture separates:

* HTTP communication
* application orchestration
* business logic
* data access
* persistence
* infrastructure
* cross-cutting concerns

Business domains such as Booking, Commerce, Feed, Promotions, Recommendations, and Notifications remain independently structured while collaborating through controlled application flows.

The system is designed to scale through progressive evolution rather than premature distribution.

The fundamental architectural objective is therefore not simply to make MOSEL scalable.

It is to make MOSEL **understandable, maintainable, consistent, and capable of evolving** as the platform and its requirements grow.
