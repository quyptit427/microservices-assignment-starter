# Analysis and Design — Movie Ticket Booking System

> **Goal**: Analyze a specific business process and design a service-oriented automation solution (SOA/Microservices).
> Scope: 4–6 week assignment — focus on **one business process**, not an entire system.
>
> **Target decomposition**: 5 microservices — **Movie, Room, Schedule, Customer, Booking**.

**References:**
1. *Service-Oriented Architecture: Analysis and Design for Services and Microservices* — Thomas Erl (2nd Edition)
2. *Microservices Patterns: With Examples in Java* — Chris Richardson
3. *Bài tập — Phát triển phần mềm hướng dịch vụ* — Hung Dang (available in Vietnamese)

---

## Part 1 — Analysis Preparation

### 1.1 Business Process Definition

Describe or diagram the high-level Business Process to be automated.

- **Domain**: Entertainment / Cinema

- **Business Process**: Movie Ticket Booking

- **Actors**:

  - Customers

- **Scope**:

    ***In Scope:***
    - Browsing movies and available screening schedules
    - Viewing seat availability for a specific schedule
    - Booking one or more seats for a schedule (with loyalty point discount)
    - Automatic creation of ticket bill and tickets after successful booking
    - Canceling a booking and restoring used loyalty points

    ***Out of Scope:***
    - Online payment gateway integration (e.g., VNPay, Momo)
    - Email or SMS notification to customers
    - Customer account registration with password
    - Reporting and analytics dashboard
    - Multi-cinema / multi-branch support
    - Staff management of movies, rooms, seats, and schedules
    - Staff check-in of tickets at the cinema entrance
    - Staff authentication

---

**Process Diagram:**

![Class Diagram](classdiagram.png)

### 1.2 Existing Automation Systems

**"None — the process is currently performed manually."**

| System Name | Type | Current Role | Interaction Method |
|-------------|------|--------------|-------------------|
| None | — | — | — |

### 1.3 Non-Functional Requirements

Non-functional requirements serve as input for identifying Utility Service and Microservice Candidates in step 2.7.

| Requirement | Description |
|-------------|-------------|
| Performance | API response time under 500ms for most requests; booking transaction completes within 1 second |
| Scalability | Each microservice can be scaled independently; supports concurrent seat booking without double-booking |
| Availability | Services restart automatically on failure; health check endpoint on every service |
| Security | Input validation on all endpoints; shared secret between services for internal calls |
| Maintainability | Codebase separated by microservice with clear README per service; Git version control |
| Usability | Simple SPA interface requiring no technical knowledge for customers to browse and book tickets |

---

## Part 2 — REST/Microservices Modeling

### 2.1 Decompose Business Process & 2.2 Filter Unsuitable Actions

Decompose the process from 1.1 into granular actions. Mark actions unsuitable for service encapsulation.

| # | Action | Actor | Description | Suitable? |
|---|--------|-------|-------------|-----------|
| 1 | Browse movies | Customer | Customer views list of currently showing movies | ✅ |
| 2 | View schedules | Customer | Customer views available showtimes for a selected movie | ✅ |
| 3 | View seat map | Customer | Customer views available/booked seats for a schedule | ✅ |
| 4 | Select seats | Customer | Customer selects one or more available seats | ✅ |
| 5 | Enter booking info | Customer | Customer enters phone number, name, and loyalty points to use | ✅ |
| 6 | Validate movie, schedule & seats | System | System verifies schedule exists, seats belong to the correct room, and movie-room mapping is valid | ✅ |
| 7 | Check seat availability | System | System checks no active ticket exists for the same schedule + seat | ✅ |
| 8 | Calculate payment | System | System computes total, discount from points, paid amount, change | ✅ |
| 9 | Create ticket bill | System | System creates a TicketBill record (pending → paid) | ✅ |
| 10 | Create tickets | System | System creates one Ticket per selected seat linked to the bill | ✅ |
| 11 | Award loyalty points | System | System deducts used points and awards earned points to customer | ✅ |
| 12 | Cancel booking | Customer | Customer requests cancellation of a booking | ✅ |
| 13 | Restore points on cancel | System | System restores used points and removes earned points (compensation) | ✅ |

> Actions marked ❌: manual-only, require human interaction, or are pure UI state — cannot be encapsulated as a service.

---

### 2.3 Entity Service Candidates

Identify business entities and group reusable (agnostic) actions into Entity Service Candidates.

| Entity | Service Candidate | Agnostic Actions |
|--------|-------------------|------------------|
| Movie | Movie Service | Get all movies, get movie by ID |
| Room | Room Service | Get all rooms, get room by ID |
| Seat | Room Service | Get seats by room, get seat by ID |
| Schedule | Schedule Service | Get all schedules (filter by movie), get schedule by ID, get seats for schedule |
| Customer | Customer Service | Register customer, get customer by ID, get customer by phone number |
| LoyaltyPoints | Customer Service | Deduct points, award points, get point balance |
| TicketBill | Booking Service | Get bill by ID, get all bills for customer |
| Ticket | Booking Service | Get ticket by ID |

---

### 2.4 Task Service Candidate

Group process-specific (non-agnostic) actions into Task Service Candidates.

| Non-agnostic Action | Task Service Candidate |
|---------------------|------------------------|
| Validate movie + schedule + seats → check availability → create bill + tickets → process points → commit | **Booking Saga** — `POST /booking` in Booking Service; orchestrates cross-service validation (Schedule Service + Room Service + Customer Service) and local DB writes atomically |
| Cancel booking: mark tickets canceled + restore loyalty points (compensating transaction) | **Cancel Saga** — `POST /booking/{id}/cancel` in Booking Service; compensates the booking saga by calling Customer Service to restore points |

---

### 2.5 Identify Resources

Map entities/processes to REST URI Resources.

| Entity / Process | Resource URI | Service |
|------------------|--------------|---------|
| Health check | `/health` | All services |
| Movie list | `/movies` | Movie Service |
| Movie detail | `/movies/{id}` | Movie Service |
| Room list | `/rooms` | Room Service |
| Room detail | `/rooms/{id}` | Room Service |
| Seats in room | `/rooms/{id}/seats` | Room Service |
| Schedule list | `/schedules` | Schedule Service |
| Schedule detail | `/schedules/{id}` | Schedule Service |
| Seats for schedule | `/schedules/{id}/seats` | Schedule Service |
| Customer register | `/customers` | Customer Service |
| Customer by ID | `/customers/{id}` | Customer Service |
| Customer by phone | `/customers/phone/{phone}` | Customer Service |
| Update customer points | `/customers/{id}/points` | Customer Service |
| Book tickets (Saga) | `/booking` | Booking Service |
| Cancel booking | `/booking/{id}/cancel` | Booking Service |
| Available seats | `/booking/schedule/{id}/available-seats` | Booking Service |
| Bill detail | `/bills/{id}` | Booking Service |
| Bills by customer | `/bills/customer/{id}` | Booking Service |
| Ticket detail | `/tickets/{id}` | Booking Service |

---

### 2.6 Associate Capabilities with Resources and Methods

| Service Candidate | Capability | Resource | Protocol | HTTP Method | Response Codes |
|-------------------|------------|----------|----------|-------------|----------------|
| **API Gateway** | Route & forward all client requests | All `/api/*` routes | REST (proxy) | — | — |
| Movie Service | Health check | `/health` | REST | GET | 200 |
| Movie Service | List movies | `/movies` | REST | GET | 200 |
| Movie Service | Get movie | `/movies/{id}` | REST | GET | 200, 404 |
| Room Service | Health check | `/health` | REST | GET | 200 |
| Room Service | List rooms | `/rooms` | REST | GET | 200 |
| Room Service | Get room | `/rooms/{id}` | REST | GET | 200, 404 |
| Room Service | List seats in room | `/rooms/{id}/seats` | REST | GET | 200, 404 |
| Schedule Service | Health check | `/health` | REST | GET | 200 |
| Schedule Service | List schedules | `/schedules` | REST | GET | 200 |
| Schedule Service | Get schedule | `/schedules/{id}` | REST | GET | 200, 404 |
| Schedule Service | Get seats for schedule | `/schedules/{id}/seats` | REST | GET | 200, 404 |
| Customer Service | Health check | `/health` | REST | GET | 200 |
| Customer Service | Register customer | `/customers` | REST | POST | 201, 400 |
| Customer Service | Get customer by ID | `/customers/{id}` | REST | GET | 200, 404 |
| Customer Service | Get customer by phone | `/customers/phone/{phone}` | REST | GET | 200, 404 |
| Customer Service | Update loyalty points | `/customers/{id}/points` | REST | PATCH | 200, 400, 404 |
| Booking Service | Health check | `/health` | REST | GET | 200 |
| Booking Service | **Book tickets (Saga)** | `/booking` | REST | POST | 201, 400, 409, 503 |
| Booking Service | **Cancel booking (Compensation)** | `/booking/{id}/cancel` | REST | POST | 200, 400, 404 |
| Booking Service | Available seats | `/booking/schedule/{id}/available-seats` | REST | GET | 200, 503 |
| Booking Service | Get bill | `/bills/{id}` | REST | GET | 200, 404 |
| Booking Service | Get bills by customer | `/bills/customer/{id}` | REST | GET | 200 |
| Booking Service | Get ticket | `/tickets/{id}` | REST | GET | 200, 404 |

---

### 2.7 Utility Service & Microservice Candidates

Based on Non-Functional Requirements (1.3) and Processing Requirements, identify cross-cutting utility logic or logic requiring high autonomy/performance.

| Candidate | Type | Justification |
|-----------|------|---------------|
| API Gateway (Nginx) | Utility Service | Cross-cutting concern: single entry point for all client requests — handles routing and CORS; decouples frontend from internal service topology |
| Movie Service | Entity Service | Manages the movie catalog domain; independent lifecycle and simple read operations exposed to customers |
| Room Service | Entity Service | Owns screening rooms and seats; can scale independently from schedules and booking traffic |
| Schedule Service | Entity Service | Owns showtime lifecycle; isolated bounded context with high read volume from customers |
| Customer Service | Entity Service | Owns customer profile and loyalty point balance; called by Booking Service during saga |
| Booking Service | Task Service | High autonomy — orchestrates the booking saga spanning multiple services; owns transactional data (bills, tickets); handles compensating transactions on cancel |

---

### 2.8 Service Composition Candidates

Interaction diagram showing how Service Candidates collaborate to fulfill the business process.

```mermaid
sequenceDiagram
    participant C as Customer
    participant FE as Frontend :4000
    participant GW as Gateway :8082
    participant MO as Movie Service :5001
    participant RO as Room Service :5004
    participant SD as Schedule Service :5005
    participant CU as Customer Service :5003
    participant BO as Booking Service :5002

    %% Browse movies
    C->>FE: Open app
    FE->>GW: GET /api/movies
    GW->>MO: GET /movies
    MO-->>GW: Movie[]
    GW-->>FE: Movie[]

    %% View schedules & seat map
    C->>FE: Select movie → select schedule
    FE->>GW: GET /api/schedules?movie_id={id}
    GW->>SD: GET /schedules?movie_id={id}
    SD-->>GW: Schedule[]
    GW-->>FE: Schedule[]
    FE->>GW: GET /api/booking/schedule/{id}/available-seats
    GW->>BO: GET /booking/schedule/{id}/available-seats
    BO->>SD: GET /schedules/{id}
    SD-->>BO: Schedule(room_id, movie_id)
    BO->>RO: GET /rooms/{room_id}/seats
    RO-->>BO: Seat[]
    BO-->>GW: available_seat_ids[]
    GW-->>FE: available_seat_ids[]

    %% Book tickets (Saga)
    C->>FE: Select seats → confirm booking
    FE->>GW: POST /api/booking
    GW->>BO: POST /booking
    Note over BO: Step 1-2: Validate schedule + room seats
    BO->>SD: GET /schedules/{id}
    SD-->>BO: Schedule (movie_id, room_id, price)
    BO->>RO: GET /rooms/{room_id}/seats
    RO-->>BO: Seat[]
    Note over BO: Step 3: Check double-booking (local DB)
    BO->>CU: GET /customers/phone/{phone}
    CU-->>BO: Customer (or 404 → create)
    BO->>CU: POST /customers (if new)
    CU-->>BO: Customer
    Note over BO: Step 5: Calculate payment (price, points)
    Note over BO: Step 6-7: INSERT TicketBill + Tickets
    BO->>CU: PATCH /customers/{id}/points
    CU-->>BO: updated Customer
    Note over BO: Step 9: bill status=paid → COMMIT
    BO-->>GW: BillOut (with tickets)
    GW-->>FE: BillOut
    FE-->>C: Booking confirmed ✅

    %% Cancel booking (Compensation)
    C->>FE: Cancel booking
    FE->>GW: POST /api/booking/{id}/cancel
    GW->>BO: POST /booking/{id}/cancel
    Note over BO: Mark tickets canceled
    BO->>CU: PATCH /customers/{customer_id}/points (restore)
    CU-->>BO: updated Customer
    Note over BO: bill status=canceled → COMMIT
    BO-->>FE: canceled bill
    FE-->>C: Cancellation confirmed ✅
```

---

## Part 3 — Service-Oriented Design

### 3.1 Uniform Contract Design

Service Contract specification for each service. Full OpenAPI specs:
- [`docs/api-specs/service-movie.yaml`](api-specs/service-movie.yaml) — Movie Service
- [`docs/api-specs/service-room.yaml`](api-specs/service-room.yaml) — Room Service
- [`docs/api-specs/service-schedule.yaml`](api-specs/service-schedule.yaml) — Schedule Service
- [`docs/api-specs/service-customer.yaml`](api-specs/service-customer.yaml) — Customer Service
- [`docs/api-specs/service-booking.yaml`](api-specs/service-booking.yaml) — Booking Service

**API Gateway (Nginx :8082):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/api/movies` | GET | application/json | 200 |
| `/api/movies/{id}` | GET | application/json | 200, 404 |
| `/api/rooms` | GET | application/json | 200 |
| `/api/rooms/{id}` | GET | application/json | 200, 404 |
| `/api/rooms/{id}/seats` | GET | application/json | 200, 404 |
| `/api/schedules` | GET | application/json | 200 |
| `/api/schedules/{id}` | GET | application/json | 200, 404 |
| `/api/schedules/{id}/seats` | GET | application/json | 200, 404 |
| `/api/customers` | POST | application/json | 201, 400 |
| `/api/customers/{id}` | GET | application/json | 200, 404 |
| `/api/customers/phone/{phone}` | GET | application/json | 200, 404 |
| `/api/customers/{id}/points` | PATCH | application/json | 200, 400, 404 |
| `/api/booking` | POST | application/json | 201, 400, 409, 503 |
| `/api/booking/{id}/cancel` | POST | application/json | 200, 400, 404 |
| `/api/booking/schedule/{id}/available-seats` | GET | application/json | 200, 503 |
| `/api/bills/{id}` | GET | application/json | 200, 404 |
| `/api/bills/customer/{id}` | GET | application/json | 200 |
| `/api/tickets/{id}` | GET | application/json | 200, 404 |

---

**Movie Service (FastAPI :5001):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/health` | GET | `application/json` | 200 |
| `/movies` | GET | `application/json` | 200 |
| `/movies/{id}` | GET | `application/json` | 200, 404 |

---

**Room Service (FastAPI :5004):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/health` | GET | `application/json` | 200 |
| `/rooms` | GET | `application/json` | 200 |
| `/rooms/{id}` | GET | `application/json` | 200, 404 |
| `/rooms/{id}/seats` | GET | `application/json` | 200, 404 |

---

**Schedule Service (FastAPI :5005):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/health` | GET | `application/json` | 200 |
| `/schedules` | GET | `application/json` | 200 |
| `/schedules/{id}` | GET | `application/json` | 200, 404 |
| `/schedules/{id}/seats` | GET | `application/json` | 200, 404 |

---

**Customer Service (FastAPI :5003):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/health` | GET | `application/json` | 200 |
| `/customers` | POST | `application/json` | 201, 400 |
| `/customers/{id}` | GET | `application/json` | 200, 404 |
| `/customers/phone/{phone}` | GET | `application/json` | 200, 404 |
| `/customers/{id}/points` | PATCH | `application/json` | 200, 400, 404 |

---

**Booking Service (FastAPI :5002):**

| Endpoint | Method | Media Type | Response Codes |
|----------|--------|------------|----------------|
| `/health` | GET | `application/json` | 200 |
| `/booking` | POST | `application/json` | 201, 400, 409, 503 |
| `/booking/{id}/cancel` | POST | `application/json` | 200, 400, 404 |
| `/booking/schedule/{id}/available-seats` | GET | `application/json` | 200, 503 |
| `/bills/{id}` | GET | `application/json` | 200, 404 |
| `/bills/customer/{id}` | GET | `application/json` | 200 |
| `/tickets/{id}` | GET | `application/json` | 200, 404 |

---

### 3.2 Service Logic Design

## Movie Service — Logic

```mermaid
flowchart TD
    A[GET /movies] --> B[Return active movies]

    C[GET /movies id] --> D[Lookup by ID]
    D -->|Not found| E[Return 404]
    D --> F[Return 200]
```

## Room Service — Logic

```mermaid
flowchart TD
    A[GET /rooms] --> B[Return room list]

    C[GET /rooms id] --> D[Lookup by ID]
    D -->|Not found| E[Return 404]
    D --> F[Return 200]

    G[GET /rooms id seats] --> H[Return seat list]
```

## Schedule Service — Logic

```mermaid
flowchart TD
    A[GET /schedules by movie] --> B[Return schedules]

    C[GET /schedules id] --> D[Lookup by ID]
    D -->|Not found| E[Return 404]
    D --> F[Return 200]

    G[GET /schedules id seats] --> H[Get room_id]
    H --> I[Call Room Service]
    I --> J[Return seats]
```

## Customer Service — Logic

```mermaid
flowchart TD
    A[POST /customers] --> B[Check phone unique]
    B -->|Duplicate| C[Return 400]
    B --> D[Insert customer]
    D --> E[Return 201]

    F[GET /customers phone] --> G[Lookup customer]
    G -->|Not found| H[Return 404]

    I[PATCH /customers points] --> J[Update points]
    J --> K[Clamp >= 0]
    K --> L[Return updated customer]
```

## Booking Service — Booking Saga Flow

```mermaid
flowchart TD
    A[POST /booking] --> B[Get schedule]
    B -->|Error| X1[Return error]

    B --> C[Get seats from schedule]
    C --> D[Check seat conflicts]
    D -->|Conflict| X2[Return 409]

    D --> E[Get customer by phone]
    E -->|Not found| F[Create customer]

    E --> G[Calculate payment]
    G -->|Invalid| X3[Return 400]

    G --> H[Insert TicketBill]
    H --> I[Insert Tickets]

    I --> J[Update customer points]
    J --> K[Set bill paid]

    K --> L[Commit DB]
    L --> M[Return success]

    X1 --> N[End]
    X2 --> N
    X3 --> N
```

## Booking Service — Cancel Saga (Compensation)

```mermaid
flowchart TD
    A[POST /booking cancel] --> B[Get bill]
    B -->|Not found| X1[Return 404]

    B --> C[Check already canceled]
    C -->|Yes| X2[Return 400]

    C --> D[Restore points via Customer Service]
    D -->|Error| X3[Return 503]

    D --> E[Mark tickets canceled]
    E --> F[Set bill canceled]
    F --> G[Commit DB]

    G --> H[Return success]

    X1 --> Z[End]
    X2 --> Z
    X3 --> Z
```
