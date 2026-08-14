# 03. High-Level Architecture

## 1. Purpose

This document translates the business domain defined in [`02-business-domain.md`](./02-business-domain.md) into a technical architecture.

While the business-domain document describes **what MOSEL represents**, this document describes **how those capabilities are organized and executed within the software system**.

The architecture is designed around three primary objectives:

1. Maintain clear boundaries between business capabilities.
2. Keep application complexity manageable as the platform grows.
3. Provide an evolutionary path toward distributed architecture when scale or operational requirements justify it.

MOSEL currently follows a **modular monolith architecture** implemented using a layered backend structure.

The system is therefore a single deployable application internally divided into domain-oriented modules, with shared infrastructure and clearly defined application boundaries.

---

# 2. Architectural Context

The business domain established in Part 02 can be broadly viewed as three interacting areas:

```text
                    MOSEL
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
   Discovery      Transactions    Engagement
       │              │              │
       │              │              │
   Business        Booking          Feed
   Services        Commerce         Reviews
   Products        Orders           Favorites
   Promotions      Payments         Notifications
   Recommendations Wallet
                  Escrow
                  Payouts
```

These domains are not implemented as isolated applications.

They operate within a shared backend while maintaining distinct responsibilities.

The resulting architecture can be represented as:

```text
┌──────────────────────────────────────────────────────────────┐
│                        CLIENTS                               │
│                                                              │
│                Web / Mobile / External Clients               │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              │ HTTPS / REST
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                      API BOUNDARY                            │
│                                                              │
│   Middleware → Routing → Controllers → Validation           │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     DOMAIN MODULES                           │
│                                                              │
│ Identity │ Business │ Booking │ Commerce │ Feed             │
│ Products │ Orders   │ Promo   │ Recommend│ Notifications    │
│ Analytics│ Admin    │ ...                                      │
└─────────────────────────────┬────────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
┌──────────────────────────────┐   ┌──────────────────────────┐
│       DATA ACCESS            │   │     INFRASTRUCTURE       │
│                              │   │                          │
│       Prisma ORM             │   │ Redis                    │
│                              │   │ Background Jobs           │
└──────────────┬───────────────┘   │ External Integrations    │
               │                   │ Events / Messaging       │
               ▼                   └──────────────────────────┘
┌──────────────────────────────┐
│         PostgreSQL           │
│                              │
│      Primary Persistence     │
└──────────────────────────────┘
```

---

# 3. Architectural Style: Modular Monolith

MOSEL is intentionally designed as a modular monolith.

This means:

> **One application deployment, multiple logical business modules.**

The distinction is important.

A traditional monolith can gradually become a large collection of tightly coupled functionality.

MOSEL instead establishes boundaries between business capabilities from the beginning.

```text
                     MOSEL BACKEND
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
     Business           Booking           Commerce
        │                  │                  │
        ├───────┐          │          ┌───────┤
        │       │          │          │       │
        ▼       ▼          ▼          ▼       ▼
     Service Product    Availability Order   Wallet
        │                              │       │
        └──────────────┬───────────────┘       │
                       │                       │
                       ▼                       ▼
                  Shared Platform        Financial State
```

Each module has a defined responsibility even though the modules currently execute within the same application process.

---

# 4. Why a Modular Monolith?

The architecture deliberately avoids introducing microservices as the starting point.

Distributed systems introduce additional concerns including:

* network communication
* service discovery
* distributed tracing
* deployment coordination
* inter-service authentication
* distributed transactions
* eventual consistency
* failure propagation
* operational monitoring

These concerns can be justified at scale, but introducing them prematurely creates complexity before the business requires it.

MOSEL therefore follows a simpler principle:

> **Start with strong internal boundaries; distribute only when there is a measurable reason to do so.**

The modular monolith provides:

* simpler deployment
* simpler local development
* straightforward debugging
* strong transactional consistency
* lower infrastructure requirements
* easier cross-domain operations
* clear migration paths

---

# 5. Domain Modules as Architectural Boundaries

The business domains identified in Part 02 become logical modules within the application.

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
├── Product Commerce
│
├── Booking
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

These are not simply organizational folders.

Each module represents a **business responsibility**.

For example:

The Booking module owns scheduling rules.

The Commerce module owns financial state.

The Feed module owns content delivery and engagement behavior.

The Recommendation module consumes signals to produce ranked results.

The distinction between these responsibilities is what gives the modular monolith its structure.

---

# 6. Module Ownership

A fundamental architectural rule is:

> **A domain should own the business rules associated with its responsibility.**

For example, Booking owns:

* booking lifecycle
* availability checks
* conflict prevention
* booking state transitions

Commerce owns:

* payment state
* wallet operations
* ledger entries
* escrow
* refunds
* payouts

Promotion owns:

* campaign state
* budget rules
* bidding
* promotion eligibility

This prevents business rules from being duplicated across unrelated parts of the system.

---

# 7. Internal Module Structure

Each major module follows a consistent application structure.

Conceptually:

```text
Booking Module
│
├── Routes
├── Controllers
├── DTOs / Validation
├── Services
├── Domain Rules
└── Data Access
```

The same pattern can be applied to Commerce:

```text
Commerce Module
│
├── Routes
├── Controllers
├── DTOs / Validation
├── Services
├── Financial Rules
└── Data Access
```

This provides consistency while allowing each module to contain domain-specific logic.

---

# 8. Layered Architecture

Inside the modules, MOSEL uses a layered request-processing model.

```text
┌───────────────────────────────┐
│            Routes             │
│       HTTP endpoint map       │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│         Controllers           │
│      HTTP coordination        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│      DTOs / Validation        │
│       Input boundaries        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│           Services            │
│       Business behavior       │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│        Data Access            │
│       Prisma / ORM            │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│          PostgreSQL           │
│        Persistent data        │
└───────────────────────────────┘
```

Each layer exists for a reason.

---

# 9. Routes

Routes define the HTTP surface exposed by the backend.

Responsibilities include:

* mapping HTTP methods to controllers
* organizing resource endpoints
* attaching relevant middleware
* establishing API structure

Routes should not contain business rules.

Their purpose is to connect the external HTTP interface to the application.

---

# 10. Controllers

Controllers operate at the HTTP boundary.

They are responsible for:

* receiving requests
* extracting request data
* invoking application services
* handling service results
* constructing HTTP responses
* selecting appropriate status codes

A controller should answer:

> "How does this HTTP request enter and leave the application?"

It should not answer:

> "What are all the business rules governing this operation?"

Those belong deeper in the application.

---

# 11. DTOs and Validation

Data Transfer Objects define the structure of data entering or leaving application boundaries.

The request pipeline therefore becomes:

```text
External Input
      │
      ▼
DTO
      │
      ▼
Validation
      │
      ▼
Service
```

This prevents raw external input from being treated as trusted application data.

Validation may enforce:

* required fields
* data types
* ranges
* formats
* allowed values
* structural constraints

Domain-level validation then determines whether the operation is actually permissible.

For example:

```text
"Is the request structurally valid?"
             │
             ▼
          DTO /
        Validation
             │
             ▼
"Is this operation allowed?"
             │
             ▼
        Domain Rules
```

These are related but distinct responsibilities.

---

# 12. Services

Services contain the primary business behavior of the application.

A service coordinates:

* domain rules
* persistence operations
* cross-domain interactions
* transactions
* external integrations where necessary

For example, a booking operation may conceptually perform:

```text
Create Booking
      │
      ├── Validate Customer
      ├── Validate Business
      ├── Validate Service
      ├── Check Availability
      ├── Prevent Conflict
      ├── Create Booking
      └── Trigger Follow-Up Operations
```

The controller should not own this workflow.

The Booking service does.

---

# 13. Data Access

MOSEL uses Prisma as the primary data-access abstraction over PostgreSQL.

The purpose of the data-access layer is to isolate persistence concerns from business behavior.

```text
Service
   │
   ▼
Data Access
   │
   ▼
Prisma
   │
   ▼
PostgreSQL
```

This separation makes it easier to reason about:

* queries
* transactions
* relationships
* persistence constraints
* migrations
* database-specific concerns

---

# 14. Shared Infrastructure

Not every capability belongs to a business domain.

Some capabilities support the entire application.

These include:

* configuration
* logging
* error handling
* authentication middleware
* authorization infrastructure
* caching
* background jobs
* external service clients
* observability
* auditing

Conceptually:

```text
                    Shared Infrastructure
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
    Booking             Commerce              Feed
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                            ▼
                       Platform Layer
```

Shared infrastructure should provide capabilities without taking ownership of unrelated business rules.

---

# 15. Cross-Domain Communication

The domains established in Part 02 naturally interact.

For example:

```text
Customer
   │
   ▼
Business
   │
   ▼
Service
   │
   ▼
Booking
   │
   ▼
Commerce
```

A completed booking can also affect:

```text
Booking Completed
       │
       ├────► Commerce
       ├────► Notifications
       ├────► Analytics
       └────► Recommendations
```

This creates an important architectural requirement:

**Domains must be able to collaborate without becoming tightly coupled.**

---

# 16. Synchronous Communication

Some interactions require an immediate result.

Examples include:

* retrieving a business
* checking availability
* creating a booking
* retrieving an order
* validating payment state

These operations remain synchronous within the request lifecycle.

```text
Client
  │
  ▼
API
  │
  ▼
Domain Service
  │
  ▼
Database
  │
  ▼
Response
  │
  ▼
Client
```

The client receives the result directly.

---

# 17. Asynchronous Communication

Other operations do not need to block the primary request.

Examples include:

* sending notifications
* processing analytics
* updating recommendation signals
* background feed processing
* non-critical integrations

Conceptually:

```text
                    Core Operation
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
        Immediate Result       Event / Job
                                    │
                                    ▼
                             Background Worker
                                    │
                       ┌────────────┼────────────┐
                       ▼            ▼            ▼
                  Analytics   Notification   Recommendation
```

This separation allows the critical request path to remain focused on the operation the user actually requested.

---

# 18. Event-Oriented Evolution

MOSEL does not require a fully event-driven architecture today.

However, the domain structure provides a natural path toward event-based communication.

For example:

```text
Booking Completed
       │
       ▼
Domain Event
       │
       ├────────────► Notification
       │
       ├────────────► Analytics
       │
       ├────────────► Recommendation
       │
       └────────────► Commerce
```

This approach can reduce direct dependencies between modules as the platform grows.

The event itself represents a meaningful domain occurrence rather than an implementation detail.

---

# 19. Data Architecture

PostgreSQL is the primary source of durable application state.

It is particularly important for domains where consistency matters:

* users
* businesses
* bookings
* orders
* payments
* wallets
* ledger transactions

Redis provides complementary low-latency capabilities.

Potential responsibilities include:

* caching
* temporary state
* rate limiting
* frequently accessed data
* session-related workloads
* ranking support

The conceptual relationship is:

```text
                         Application
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
               Redis                 PostgreSQL
                  │                       │
          Fast / Temporary          Durable / Primary
```

Redis is therefore not intended to replace PostgreSQL as the system of record.

---

# 20. Request Lifecycle

A typical request through MOSEL follows this lifecycle:

```text
                    HTTP Request
                         │
                         ▼
                   Middleware
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
        Authentication Authorization Validation
              │          │          │
              └──────────┼──────────┘
                         │
                         ▼
                       Router
                         │
                         ▼
                    Controller
                         │
                         ▼
                   Application
                     Service
                         │
                ┌────────┴────────┐
                │                 │
                ▼                 ▼
          Domain Logic       Data Access
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
```

This lifecycle keeps external communication separate from internal business behavior.

---

# 21. Transaction Boundaries

Some business operations require multiple state changes to succeed atomically.

This is particularly important in the Booking and Commerce domains.

For example:

```text
Booking Workflow
      │
      ├── Validate Availability
      ├── Create Booking
      ├── Reserve Relevant State
      └── Create Financial Record
```

Similarly:

```text
Financial Workflow
      │
      ├── Record Transaction
      ├── Update Wallet State
      └── Create Ledger Entry
```

Where operations must succeed or fail together, database transaction boundaries are used.

The objective is to prevent partial state.

A system should not end up with:

```text
Booking = SUCCESS
Payment = FAILED
Financial Record = MISSING
```

when the business operation requires these states to remain consistent.

---

# 22. Concurrency

Marketplace systems are inherently concurrent.

Multiple users can interact with the same resources simultaneously.

For example:

```text
Customer A ────┐
               ├──► Same Time Slot
Customer B ────┘
```

Without concurrency controls, both requests could potentially observe the same availability and attempt to create conflicting bookings.

MOSEL therefore considers mechanisms such as:

* database transactions
* unique constraints
* atomic operations
* state-transition validation
* idempotency
* appropriate locking strategies

The objective is to preserve domain invariants under concurrent execution.

---

# 23. Security Boundary

The API represents a trust boundary.

External clients cannot be assumed to behave correctly.

The architecture therefore follows:

```text
Untrusted Input
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
Domain Rules
      │
      ▼
Persistence
```

Each stage provides a different form of protection.

Authentication answers:

> Who is making the request?

Authorization answers:

> Is this actor allowed to perform the operation?

Validation answers:

> Is the input structurally acceptable?

Domain rules answer:

> Is this operation valid within the business?

This separation prevents security responsibilities from being reduced to a single authentication check.

---

# 24. Error Propagation

Errors should move through the architecture without leaking internal implementation details.

A simplified model is:

```text
Domain / Infrastructure Error
             │
             ▼
       Error Handling
             │
             ▼
      Application Error
             │
             ▼
       HTTP Response
```

The API should expose predictable error semantics while internal errors such as database details remain protected.

This provides a consistent contract for clients and makes operational debugging easier.

---

# 25. Cross-Cutting Concerns

Several capabilities operate across the entire application.

These include:

* authentication
* authorization
* validation
* logging
* error handling
* rate limiting
* auditing
* observability
* configuration

They should be implemented as reusable platform capabilities rather than independently recreated inside every domain.

For example:

```text
                 Authentication
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
     Booking        Commerce         Feed
```

This reduces duplication and improves consistency.

---

# 26. External Integrations

MOSEL may interact with external infrastructure and third-party services.

Examples include:

* payment providers
* notification providers
* mapping services
* cloud services
* authentication providers
* external analytics systems

External dependencies should be isolated behind integration boundaries.

```text
MOSEL Domain
     │
     ▼
Integration Boundary
     │
     ▼
External Provider
```

The domain should depend on the capability it needs rather than becoming tightly coupled to a particular external provider.

This makes provider replacement and testing easier.

---

# 27. Scalability Strategy

The architecture follows an evolutionary scaling model.

### Stage 1 — Modular Monolith

```text
One Application
     │
     ├── Business
     ├── Booking
     ├── Commerce
     ├── Feed
     └── Other Domains
```

### Stage 2 — Horizontal Scaling

```text
Load Balancer
      │
 ┌────┼────┐
 ▼    ▼    ▼
API  API  API
```

Because the API layer is designed to remain stateless where possible, multiple application instances can serve requests.

### Stage 3 — Infrastructure Scaling

Introduce:

* distributed caching
* background workers
* asynchronous processing
* queue-based workloads

### Stage 4 — Domain Extraction

Only domains demonstrating independent scaling or operational requirements become candidates for service extraction.

```text
Modular Monolith
       │
       ▼
Identify Bottleneck
       │
       ▼
Extract Specific Domain
       │
       ▼
Independent Service
```

This is fundamentally different from designing every component as a microservice from day one.

---

# 28. Potential Service Extraction

If MOSEL eventually becomes large enough to justify distributed architecture, extraction should follow domain boundaries.

Potential candidates could include workloads such as:

```text
                MOSEL Platform
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    Core API      Commerce      Recommendation
        │             │             │
        │             │             │
        ▼             ▼             ▼
   PostgreSQL     Financial     Ranking
                   Systems       Infrastructure
```

The actual extraction strategy would depend on measured system behavior.

The architecture does not assume that every module will eventually become a service.

---

# 29. Observability

A scalable system requires visibility into both technical and business behavior.

Technical observability may include:

* structured logs
* request metrics
* latency
* error rates
* database performance
* cache performance
* background job status

Business observability may include:

* booking conversion
* order completion
* payment failures
* promotion CTR
* customer engagement
* recommendation performance

This distinction is important because a technically healthy system can still have a failing business workflow.

---

# 30. Architectural Trade-Offs

The current architecture deliberately makes several trade-offs.

### Benefits

* straightforward deployment
* strong transactional consistency
* simpler debugging
* lower infrastructure overhead
* easier development
* shared domain model
* clear business boundaries
* straightforward evolution

### Costs

* domains share the same deployment lifecycle
* application failures can potentially affect multiple modules
* individual modules cannot independently scale
* module boundaries require engineering discipline
* large workloads may eventually require extraction

These trade-offs are intentional.

The architecture optimizes for **simplicity with a credible path to scale**, rather than maximum distribution from the beginning.

---

# 31. Architectural Evolution

The long-term evolution of MOSEL can be viewed as:

```text
                 Current
                   │
                   ▼
          Modular Monolith
                   │
                   ▼
          Horizontal Scaling
                   │
                   ▼
      Caching + Background Jobs
                   │
                   ▼
       Identify Real Bottlenecks
                   │
                   ▼
       Selective Service Extraction
                   │
                   ▼
          Distributed Platform
```

The architecture should evolve in response to:

* traffic
* latency
* workload characteristics
* operational requirements
* failure isolation
* deployment requirements
* team ownership

rather than architectural fashion.

---

# 32. Architectural Principles

The high-level architecture can be summarized through the following principles.

### Principle 1 — Domains own business rules

A domain should be responsible for the rules associated with its capability.

### Principle 2 — Transport should not own business logic

HTTP concerns belong at the API boundary.

### Principle 3 — Business logic should not depend directly on HTTP

Domain behavior should remain independent of the transport mechanism wherever practical.

### Principle 4 — Persistence should remain isolated

Database-specific concerns should not spread throughout the application.

### Principle 5 — Cross-domain interactions should be deliberate

Dependencies between domains should exist because the business relationship requires them.

### Principle 6 — Critical operations require consistency

Financial and scheduling workflows must preserve their invariants under failure and concurrency.

### Principle 7 — Asynchronous processing should protect critical paths

Non-critical work should not unnecessarily increase user-facing latency.

### Principle 8 — Distribution should follow evidence

A module should become a separate service only when there is a concrete reason.

---

# 33. End-to-End Example

Consider a customer booking a service.

The request crosses several architectural boundaries:

```text
Customer
   │
   ▼
REST API
   │
   ▼
Authentication
   │
   ▼
Validation
   │
   ▼
Booking Controller
   │
   ▼
Booking Service
   │
   ├──► Business
   │
   ├──► Service
   │
   ├──► Availability
   │
   └──► Commerce
            │
            ▼
         PostgreSQL
            │
            ▼
       Booking Result
            │
      ┌─────┼───────────────┐
      ▼     ▼               ▼
Notification Analytics Recommendation
```

The important observation is that the customer experiences a single action:

**"Book this service."**

Internally, however, the platform coordinates multiple domains while maintaining clear ownership boundaries.

This is the fundamental architectural challenge MOSEL is designed to solve.

---

# 34. Relationship to the Remaining Architecture

The high-level architecture establishes the foundation for the remaining technical documentation.

```text
01 System Overview
        │
        ▼
02 Business Domain
        │
        ▼
03 High-Level Architecture
        │
        ├──────────────► API Design
        │
        ├──────────────► Authentication
        │
        ├──────────────► Database Design
        │
        ├──────────────► Booking Engine
        │
        ├──────────────► Commerce Engine
        │
        ├──────────────► Feed Engine
        │
        ├──────────────► Recommendation Engine
        │
        ├──────────────► Notification System
        │
        ├──────────────► Scalability
        │
        └──────────────► Security
```

Each subsequent document should refine one part of this architecture rather than redefining it.

---

# 35. Conclusion

MOSEL translates its business-domain model into a modular software architecture designed around clear ownership, layered request processing, controlled cross-domain interaction, and evolutionary scalability.

The modular monolith provides a practical starting point while preserving architectural boundaries that can support future service extraction.

The resulting architecture separates:

* external communication
* transport concerns
* validation
* business logic
* persistence
* infrastructure
* cross-domain workflows
* asynchronous processing

The central architectural principle is straightforward:

> **Keep the system modular before making it distributed.**

MOSEL is therefore not designed around the assumption that scale requires microservices.

It is designed around the assumption that **good boundaries make future architectural decisions possible**.
