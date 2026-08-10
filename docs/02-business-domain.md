# 02. Business Domain

## 1. Purpose

This document defines the business domain of MOSEL and establishes the core entities, relationships, and business capabilities that make up the platform.

The purpose is to describe **what the system represents and the business responsibilities it supports**, independently of implementation details.

The business domain forms the foundation for the rest of the architecture.

The application architecture, database model, API design, booking engine, commerce infrastructure, feed system, and recommendation engine are all built around these domain concepts.

---

# 2. Domain Overview

MOSEL is a multi-domain marketplace platform connecting customers with businesses that provide services and products.

At the center of the platform are three fundamental concepts:

```text
                    MOSEL
                      │
          ┌───────────┴───────────┐
          │                       │
       Customers              Businesses
          │                       │
          │              ┌────────┴────────┐
          │              │                 │
          │           Services          Products
          │              │                 │
          └──────────────┴─────────────────┘
                         │
                    Transactions
```

Customers discover businesses, interact with their services and products, create bookings or orders, make payments, and engage with platform content.

Businesses manage their profiles, services, products, availability, promotions, orders, and customer relationships.

The platform provides the infrastructure connecting these interactions.

---

# 3. Core Actors

MOSEL has several categories of participants.

## 3.1 Customers

Customers are users who interact with businesses and platform content.

Typical customer activities include:

* discovering businesses
* viewing services
* viewing products
* booking services
* purchasing products
* making payments
* submitting reviews
* saving businesses
* interacting with posts
* receiving notifications

A user may therefore participate in multiple platform workflows simultaneously.

---

## 3.2 Business Owners

Business owners operate businesses on the platform.

They can:

* create and manage business profiles
* configure services
* manage products
* define availability
* manage bookings
* process orders
* create promotions
* publish content
* view analytics

Business ownership is separated from the business entity itself so that ownership and operational responsibilities can evolve independently.

---

## 3.3 Business Staff

Businesses may require multiple users to operate the same organization.

Staff members can perform operational tasks according to their assigned permissions.

Examples include:

* managing bookings
* handling orders
* updating services
* managing inventory
* responding to customers

This creates a distinction between **business ownership** and **business operations**.

---

## 3.4 Platform Administrators

Platform administrators operate MOSEL itself rather than an individual business.

Administrative responsibilities may include:

* platform moderation
* user management
* business verification
* dispute handling
* promotion management
* system monitoring
* financial oversight

Administrative capabilities are therefore separate from normal customer and business workflows.

---

# 4. Business Hierarchy

Businesses are organized within a flexible classification hierarchy.

```text
Business Type
     │
     ▼
Category
     │
     ▼
Business
     │
     ├── Services
     ├── Products
     ├── Availability
     ├── Promotions
     ├── Posts
     └── Reviews
```

This hierarchy allows MOSEL to support different types of businesses without creating separate application architectures for each vertical.

For example, the same underlying platform can support:

* salons
* barbershops
* spas
* wellness businesses
* retail businesses
* service providers

The business type determines the context, while the core business model remains consistent.

---

# 5. Business Entity

The **Business** is one of the central entities in the platform.

A business represents an organization or service provider operating within MOSEL.

Conceptually, a business contains:

* identity
* ownership
* classification
* location
* contact information
* operating configuration
* services
* products
* availability
* promotions
* content
* customer interactions
* analytics

A business can participate in multiple domains simultaneously.

For example:

```text
Business
   │
   ├── Service
   │      └── Booking
   │
   ├── Product
   │      └── Order
   │
   ├── Promotion
   │      └── Impression / Click
   │
   ├── Post
   │      └── Engagement
   │
   └── Analytics
```

This makes the Business entity a major integration point within the platform.

---

# 6. Service Domain

Services represent activities that a business provides to customers.

Examples include:

* haircuts
* styling
* massage
* consultations
* grooming
* other appointment-based services

A service typically contains:

* name
* description
* pricing
* duration
* availability requirements
* business ownership
* service status

Services become actionable through the Booking domain.

```text
Business
    │
    ▼
Service
    │
    ▼
Availability
    │
    ▼
Booking
    │
    ▼
Customer
```

---

# 7. Product Domain

Products represent physical or digital items that businesses can offer for purchase.

The product domain introduces commerce concepts that are distinct from service bookings.

A product may involve:

* pricing
* inventory
* availability
* product status
* business ownership
* order relationships

The relationship between products and orders is:

```text
Business
    │
    ▼
Product
    │
    ▼
Order
    │
    ▼
Customer
```

The separation between Services and Products allows MOSEL to support both service-based and product-based businesses without forcing them into the same transactional model.

---

# 8. Booking Domain

The Booking domain manages time-based service transactions.

A booking represents a customer's reservation for a service at a particular business and time.

Conceptually:

```text
Customer
   │
   ▼
Booking
   │
   ├── Business
   ├── Service
   ├── Time
   ├── Availability
   └── Commerce
```

The booking domain must maintain important business invariants.

Examples include:

* a booking must reference a valid service
* the service must belong to the selected business
* the requested time must be available
* conflicting bookings must be prevented
* booking state transitions must be valid

The Booking domain therefore represents more than CRUD operations.

It contains scheduling rules and consistency requirements.

---

# 9. Availability Domain

Availability represents when a business or service can accept bookings.

Availability can depend on factors such as:

* business operating hours
* service duration
* staff availability
* existing bookings
* blocked periods
* scheduling rules

The availability system exists independently from bookings because **availability is a constraint**, while a booking is a transaction against that constraint.

Conceptually:

```text
Operating Rules
      │
      ▼
Availability
      │
      ├──── Existing Bookings
      │
      ├──── Service Duration
      │
      └──── Scheduling Constraints
                  │
                  ▼
             Available Slots
```

---

# 10. Commerce Domain

Commerce is treated as a platform-level domain rather than being embedded directly inside Booking or Order management.

This allows multiple business workflows to participate in financial transactions.

Potential commerce flows include:

```text
Customer Payment
       │
       ├── Booking
       ├── Order
       ├── Delivery
       ├── Subscription
       └── Promotion
```

The commerce domain is responsible for representing financial movement and transaction state.

---

# 11. Wallet and Ledger Domain

MOSEL's financial architecture separates **wallet balances** from the underlying **transaction ledger**.

A wallet represents the current financial state associated with an account.

The ledger records the history of financial movements.

Conceptually:

```text
                  Wallet
                    │
                    ▼
              Current Balance
                    ▲
                    │
          ┌─────────┴─────────┐
          │                   │
       Credits              Debits
          │                   │
          └─────────┬─────────┘
                    │
                    ▼
              Ledger Entries
```

This distinction is important because a balance represents current state, while a ledger provides an auditable history of how that state was produced.

---

# 12. Escrow Domain

Certain transactions may require funds to be held temporarily before being released.

The escrow lifecycle can be represented as:

```text
Customer Payment
       │
       ▼
   Escrow Hold
       │
       ▼
Service / Order Completion
       │
       ▼
 Escrow Release
       │
       ▼
Business Settlement
```

Escrow introduces additional financial states and requires careful transaction management.

The architecture therefore treats escrow as part of the commerce domain rather than as a property of individual bookings or orders.

---

# 13. Orders and Delivery

The Order domain manages product-based commerce.

A simplified workflow is:

```text
Customer
   │
   ▼
Cart / Purchase
   │
   ▼
Order
   │
   ├── Products
   ├── Payment
   └── Delivery
          │
          ▼
       Fulfillment
```

Orders maintain their own lifecycle and status transitions.

Delivery is modeled separately because fulfillment may involve logistics that are independent from the underlying product order.

---

# 14. Payments, Refunds, and Payouts

Commerce operations can produce several financial flows.

```text
Customer
   │
   │ Payment
   ▼
Platform
   │
   ├── Platform Fee
   │
   ├── Business Amount
   │
   └── Held Amount
          │
          ├── Release
          └── Refund
```

A business may eventually receive a payout based on completed transactions and applicable platform fees or commissions.

Refunds represent the reversal of eligible financial transactions.

These concepts remain separate because they represent different stages in the financial lifecycle.

---

# 15. Reviews and Ratings

Customer reviews provide reputation signals for businesses and services.

A review may be associated with:

* customer
* business
* booking or transaction context
* rating
* written feedback

Conceptually:

```text
Customer
    │
    ▼
Completed Interaction
    │
    ▼
Review
    │
    ▼
Business Reputation
```

Reviews contribute to both customer decision-making and platform intelligence.

They can also provide signals to recommendation and ranking systems.

---

# 16. Favorites and Customer Relationships

Customers can establish persistent relationships with businesses and content.

Examples include:

* favoriting a business
* following content
* engaging with posts

These relationships create behavioral signals that can later influence discovery and recommendation systems.

```text
Customer
   │
   ├── Favorite ───► Business
   │
   └── Engagement ─► Content
                         │
                         ▼
                    Recommendation
```

---

# 17. Social Feed Domain

The Feed domain allows businesses and potentially other platform actors to publish content.

A post may generate:

* impressions
* likes
* comments
* shares
* clicks
* saves

The feed therefore becomes both a social feature and a source of behavioral data.

```text
Business
   │
   ▼
Post
   │
   ├── Impression
   ├── Click
   ├── Like
   ├── Comment
   └── Share
```

These interactions can feed into ranking and recommendation systems.

---

# 18. Promotion and Advertising Domain

MOSEL includes a promotion system designed to increase business visibility.

Businesses can create promotional campaigns with configurable parameters such as:

* budget
* bid
* campaign status
* targeting context
* quality signals

The platform can then evaluate promotional candidates using both commercial and relevance signals.

Conceptually:

```text
Promotion
    │
    ├── Budget
    ├── Bid
    ├── Quality
    ├── Impressions
    └── Clicks
           │
           ▼
      Promotion Score
```

This creates the foundation for an auction-oriented advertising system while keeping promotion logic separate from the core feed.

---

# 19. Recommendation Domain

The recommendation domain converts platform signals into ranked results.

Signals can originate from multiple domains:

```text
Bookings ───────┐
Orders ─────────┤
Favorites ──────┤
Reviews ────────┤
Clicks ─────────┤
Engagement ─────┤
Recency ────────┤
Business Quality┘
        │
        ▼
 Recommendation Engine
        │
        ▼
 Ranked Results
```

The recommendation system can operate across multiple surfaces, including:

* businesses
* services
* products
* posts
* promotions

The important architectural principle is that recommendation logic consumes signals from other domains without owning those domains.

---

# 20. Notification Domain

Notifications communicate important system events to users.

Potential notification triggers include:

* booking confirmation
* booking cancellation
* order updates
* payment events
* promotion events
* business activity
* platform announcements

Conceptually:

```text
Domain Event
     │
     ▼
Notification Service
     │
     ├── In-App
     ├── Push
     ├── Email
     └── Other Channels
```

The notification domain should remain independent from the domain that generated the event.

For example, Booking should not need to know how a notification is delivered.

---

# 21. Analytics Domain

Analytics captures operational and behavioral information generated throughout the platform.

Examples include:

* impressions
* clicks
* conversion
* booking activity
* order activity
* revenue
* session behavior
* promotion performance

Analytics therefore acts as a cross-domain intelligence layer.

```text
Business ───────┐
Booking ────────┤
Commerce ───────┤
Feed ───────────┤
Promotion ──────┤
User Activity ──┘
        │
        ▼
     Analytics
        │
        ▼
 Metrics / Insights
```

Analytics should consume domain activity rather than becoming responsible for the underlying business operations.

---

# 22. Administrative Domain

The administrative domain provides platform-level controls.

It may include:

* user administration
* business verification
* moderation
* dispute management
* financial oversight
* promotion oversight
* platform configuration
* operational monitoring

Administrative operations are separated from normal customer and business workflows because they operate at a different trust and responsibility level.

---

# 23. Domain Relationships

The major domains form an interconnected ecosystem.

A simplified relationship map is:

```text
                           ┌──────────────┐
                           │    User      │
                           └──────┬───────┘
                                  │
                  ┌───────────────┼────────────────┐
                  │               │                │
                  ▼               ▼                ▼
             Business        Favorites        Engagement
                  │                                │
        ┌─────────┼─────────┐                      │
        │         │         │                      ▼
        ▼         ▼         ▼                    Feed
     Service   Product   Promotion                 │
        │         │         │                      │
        ▼         ▼         ▼                      │
     Booking    Order   Advertising                │
        │         │                                │
        └────┬────┘                                │
             │                                     │
             ▼                                     ▼
         Commerce                           Recommendation
             │
       ┌─────┼──────┐
       │     │      │
       ▼     ▼      ▼
    Wallet Escrow  Payout
       │
       ▼
    Ledger

All major domains
        │
        ▼
   Notifications
        │
        ▼
     Analytics
```

This relationship model demonstrates an important characteristic of MOSEL:

**The platform is not a collection of independent features.**

It is a network of related business domains sharing common actors, transactions, and signals.

---

# 24. Core Domain vs Supporting Domains

Not every domain has equal business importance.

A useful conceptual distinction is between core transactional domains and supporting platform domains.

### Core Domains

These represent the primary marketplace transactions:

* Business
* Service
* Product
* Booking
* Order
* Commerce

### Supporting Domains

These enhance discovery, engagement, and platform intelligence:

* Feed
* Reviews
* Favorites
* Recommendations
* Promotions
* Notifications
* Analytics

### Platform Domains

These provide infrastructure and governance:

* Identity & Access
* Administration
* Configuration
* Auditing

This separation helps establish architectural priorities and boundaries.

---

# 25. Domain Invariants

A major purpose of domain modeling is to identify rules that must always remain true.

Examples include:

### Booking

* A booking must reference a valid service.
* A service must belong to the selected business.
* A confirmed booking must occupy a valid time period.
* Conflicting bookings must not be accepted.

### Commerce

* Financial transactions must have traceable records.
* Ledger entries must preserve transaction history.
* Wallet state must remain consistent with financial operations.
* Refunds must reference eligible transactions.

### Business

* A service belongs to a business.
* A product belongs to a business.
* Business ownership must be explicitly represented.

### Promotion

* Promotional spending cannot exceed applicable budget constraints.
* Campaign state must control whether a promotion can participate in delivery.

These invariants are enforced by the application and persistence layers rather than relying solely on client behavior.

---

# 26. Domain Events

As domains interact, certain state changes can produce events.

Examples include:

```text
Booking Created
Booking Completed
Order Created
Payment Completed
Payment Refunded
Promotion Clicked
Post Published
Review Submitted
```

These events can become inputs for other platform capabilities.

For example:

```text
Booking Completed
       │
       ├────► Commerce
       ├────► Notifications
       ├────► Analytics
       └────► Recommendations
```

This allows the originating domain to remain focused on its own responsibility while other domains react independently.

The architecture can therefore evolve toward more event-driven processing without requiring every domain to communicate directly with every other domain.

---

# 27. Domain Boundary Principles

MOSEL follows several principles when defining domain boundaries.

### 1. A module should own a business capability.

A module should exist because there is a meaningful business responsibility behind it.

### 2. Business rules should have a clear owner.

A rule should not be duplicated across multiple modules.

### 3. Domains should minimize unnecessary coupling.

A domain should depend on another domain only when the business relationship requires it.

### 4. Shared data does not imply shared responsibility.

Two domains may reference the same entity without both owning its business rules.

### 5. Integration should occur through explicit boundaries.

Cross-domain workflows should be deliberate rather than relying on uncontrolled internal access.

---

# 28. Example: End-to-End Marketplace Interaction

Consider a customer discovering a salon and booking a service.

The interaction may involve multiple domains:

```text
Customer
   │
   ▼
Recommendation / Discovery
   │
   ▼
Business
   │
   ▼
Service
   │
   ▼
Availability
   │
   ▼
Booking
   │
   ▼
Commerce
   │
   ▼
Payment / Escrow
   │
   ▼
Notification
   │
   ▼
Analytics
```

After the appointment:

```text
Booking Completed
       │
       ├──► Commerce Settlement
       │
       ├──► Review Eligibility
       │
       ├──► Analytics
       │
       └──► Recommendation Signals
```

A single customer interaction can therefore generate activity across multiple domains.

This is one of the reasons domain boundaries are critical: the system must support collaboration without allowing every domain to become responsible for everything.

---

# 29. Business Domain Summary

MOSEL's business domain can be summarized around four major platform concerns:

```text
                     MOSEL
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
   Discovery       Transactions      Engagement
       │               │                │
       │               │                │
 Business           Booking           Feed
 Services           Commerce          Reviews
 Products           Orders            Favorites
 Promotions         Payments          Notifications
 Recommendations   Wallets
                    Escrow
                    Payouts
                       │
                       ▼
                   Analytics
```

These domains form a connected marketplace ecosystem rather than isolated features.

The architecture is designed so that each domain can evolve independently while participating in larger business workflows.

---

# 30. Conclusion

The MOSEL business domain is centered around a simple relationship:

**Customers discover businesses, interact with their services and products, conduct transactions, and generate signals that improve the marketplace experience.**

Around this core interaction, the platform introduces supporting capabilities for:

* discovery
* scheduling
* commerce
* engagement
* advertising
* recommendations
* notifications
* analytics
* administration

The resulting domain model provides the foundation upon which the technical architecture is built.

The next architectural concern is therefore not *what* the platform represents, but **how these domains are organized inside the software system**.

That separation leads directly into the High-Level Architecture documented in:

`03-high-level-architecture.md`
