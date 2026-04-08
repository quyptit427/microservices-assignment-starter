# Analysis and Design — Business Process Automation Solution

> **Goal**: Analyze a specific business process and design a service-oriented automation solution (SOA/Microservices).
> Scope: 4–6 week assignment — focus on **one business process**, not an entire system.

**References:**
1. *Service-Oriented Architecture: Analysis and Design for Services and Microservices* — Thomas Erl (2nd Edition)
2. *Microservices Patterns: With Examples in Java* — Chris Richardson
3. *Bài tập — Phát triển phần mềm hướng dịch vụ* — Hung Dang (available in Vietnamese)

---

## Part 1 — Analysis Preparation

### 1.1 Business Process Definition

Describe or diagram the high-level Business Process to be automated.

- **Domain**: E-commerce

- **Business Process**: Order Processing

- **Actors**: 

  - Customer
  - System
  - Warehouse Staff
  - Payment Gateway

- **Scope**: 

    ***In Scope:***
    - Customer submits an order
    - System validates and verifies order data
    - Calculate total order price
    - Process payment through payment gateway
    - Update inventory after successful payment
    - Persist order information in the system
    - Send order confirmation notification to customer
    - Provide API for order status retrieval

    ***Out of Scope:***
    - Physical shipping and delivery performed by warehouse staff or logistics providers
    - Return and refund processing after order completion
    - Advanced marketing and promotional campaign management
    - Business intelligence and analytical reporting
    - Physical warehouse operations beyond inventory quantity updates
    - Manual customer support and post-sale service activities

---

**Process Diagram:**

![Diagram](asset/processdiagram.png)

### 1.2 Existing Automation Systems

List existing systems, databases, or legacy logic related to this process.
**"None — the process is currently performed manually."**

| System Name | Type | Current Role | Interaction Method |
|-------------|------|--------------|-------------------|
|             |      |              |                   |


### 1.3 Non-Functional Requirements

Non-functional requirements serve as input for identifying Utility Service and Microservice Candidates in step 2.7.

| Requirement    | Description                                         |
|----------------|-----------------------------------------------------|
| Performance | API response time under 500ms for most requests; reminder notifications delivered within 1 minute of scheduled time |
| Scalability | Support up to 100 concurrent users; each microservice can be scaled independently if needed |
| Availability | 95% uptime during development and demo phases; basic error handling and service restart on failure |
| Security | JWT-based authentication, HTTPS for all API calls, role-based access control (patient/doctor/staff), basic input validation |
| Maintainability | Codebase separated by microservice with clear README per service; use of Git for version control and collaboration among 3 team members |
| Usability | Simple and intuitive UI requiring minimal technical knowledge for patients and doctors to navigate |
---


## Part 2 — REST/Microservices Modeling

# Part 2 — REST/Microservices Modeling

---

### 2.1 Decompose Business Process & 2.2 Filter Unsuitable Actions

Decompose the process from 1.1 into granular actions. Mark actions unsuitable for service encapsulation.

| # | Action | Actor | Description | Suitable? |
|---|--------|-------|-------------|-----------|
| 1 | Submit Order | Customer | Customer submits a purchase order with selected products and quantities | ✅ |
| 2 | Validate Order | System | Validate order details and verify submitted information | ✅ |
| 3 | Create Order Record | System | Persist initial order information into the order system | ✅ |
| 4 | Process Payment | Payment Gateway | Process payment transaction for the submitted order | ✅ |
| 5 | Confirm Payment | Payment Gateway | Return payment result and confirmation status | ✅ |
| 6 | Check Inventory | System | Verify product stock availability before fulfillment | ✅ |
| 7 | Update Inventory | System | Deduct purchased quantities from inventory stock | ✅ |
| 8 | Send Confirmation Notification | System | Notify customer that the order has been successfully processed | ✅ |
| 9 | Ship Order | Warehouse Staff | Physically package and deliver the order | ❌ |

Actions marked ❌ require manual physical execution and cannot be encapsulated as autonomous services.

---

### 2.3 Entity Service Candidates

Identify business entities and group reusable (agnostic) actions into Entity Service Candidates.

| Entity | Service Candidate | Agnostic Actions |
|--------|------------------|------------------|
| Order | Order Service | Validate Order, Create Order Record, Retrieve Order Details, Update Order Status |
| Payment | Payment Service | Process Payment Transaction, Confirm Payment Status, Retrieve Payment Information |
| Inventory | Inventory Service | Check Inventory Availability, Update Inventory Stock, Retrieve Inventory Status |
| Notification | Notification Service | Send Confirmation Notification, Send Payment Notification |

---

### 2.4 Task Service Candidate

Group process-specific (non-agnostic) actions into a Task Service Candidate.

| Non-agnostic Action | Task Service Candidate |
|--------------------|-----------------------|
| Coordinate end-to-end order processing workflow | Order Processing Task Service |

---

### 2.5 Identify Resources

Map entities/processes to REST URI Resources.

| Entity / Process | Resource URI |
|------------------|-------------|
| Order | /orders |
| Payment | /payments |
| Inventory | /inventory |
| Notification | /notifications |
| Order Processing Workflow | /order-processing |

---

### 2.6 Associate Capabilities with Resources and Methods

| Service Candidate | Capability | Resource | HTTP Method |
|------------------|-----------|----------|-------------|
| Order Service | Validate Order | /orders/validate | POST |
| Order Service | Create Order Record | /orders | POST |
| Order Service | Retrieve Order Details | /orders/{id} | GET |
| Order Service | Update Order Status | /orders/{id}/status | PUT |
| Payment Service | Process Payment Transaction | /payments | POST |
| Payment Service | Confirm Payment Status | /payments/{id}/confirm | GET |
| Inventory Service | Check Inventory Availability | /inventory/check | GET |
| Inventory Service | Update Inventory Stock | /inventory/{id} | PUT |
| Notification Service | Send Confirmation Notification | /notifications/order-confirmation | POST |
| Order Processing Task Service | Execute Order Processing Workflow | /order-processing | POST |

---

### 2.7 Utility Service & Microservice Candidates

Based on Non-Functional Requirements (1.3) and Processing Requirements, identify cross-cutting utility logic or logic requiring high autonomy/performance.

| Candidate | Type (Utility / Microservice) | Justification |
|----------|------------------------------|--------------|
| Notification Service | Utility Service | Cross-cutting reusable communication concern across business processes |
| Payment Service | Microservice | Requires enhanced security, independent scalability, and fault isolation |

---

### 2.8 Service Composition Candidates

Interaction diagram showing how Service Candidates collaborate to fulfill the business process.

```mermaid
sequenceDiagram
    participant Client
    participant OrderProcessingTaskService
    participant OrderService
    participant PaymentService
    participant InventoryService
    participant NotificationService

    Client->>OrderProcessingTaskService: Submit Order

    OrderProcessingTaskService->>OrderService: Validate Order
    OrderService-->>OrderProcessingTaskService: Order Validated

    OrderProcessingTaskService->>OrderService: Create Order Record
    OrderService-->>OrderProcessingTaskService: Order Created

    OrderProcessingTaskService->>PaymentService: Process Payment Transaction
    PaymentService-->>OrderProcessingTaskService: Payment Confirmed

    OrderProcessingTaskService->>InventoryService: Check Inventory Availability
    InventoryService-->>OrderProcessingTaskService: Inventory Available

    OrderProcessingTaskService->>InventoryService: Update Inventory Stock
    InventoryService-->>OrderProcessingTaskService: Inventory Updated

    OrderProcessingTaskService->>NotificationService: Send Confirmation Notification
    NotificationService-->>OrderProcessingTaskService: Notification Sent

    OrderProcessingTaskService-->>Client: Return Order Confirmation
```

## Part 3 — Service-Oriented Design

## 3.1 Uniform Contract Design

Service Contract specification for each service. Full OpenAPI specs:

docs/api-specs/order-service.yaml  
docs/api-specs/payment-service.yaml  
docs/api-specs/inventory-service.yaml  
docs/api-specs/notification-service.yaml  
docs/api-specs/order-processing-task-service.yaml  

---

### Order Service

| Endpoint | Method | Media Type | Response Codes |
|---------|--------|------------|---------------|
| /orders/validate | POST | application/json | 200, 400 |
| /orders | POST | application/json | 201, 400 |
| /orders/{id} | GET | application/json | 200, 404 |
| /orders/{id}/status | PUT | application/json | 200, 400, 404 |

---

### Payment Service

| Endpoint | Method | Media Type | Response Codes |
|---------|--------|------------|---------------|
| /payments | POST | application/json | 200, 400, 402 |
| /payments/{id} | GET | application/json | 200, 404 |
| /payments/{id}/confirm | GET | application/json | 200, 404 |

---

### Inventory Service

| Endpoint | Method | Media Type | Response Codes |
|---------|--------|------------|---------------|
| /inventory/check | GET | application/json | 200, 404 |
| /inventory/{id} | GET | application/json | 200, 404 |
| /inventory/{id} | PUT | application/json | 200, 400, 404 |

---

### Notification Service

| Endpoint | Method | Media Type | Response Codes |
|---------|--------|------------|---------------|
| /notifications/order-confirmation | POST | application/json | 200, 400 |
| /notifications/payment | POST | application/json | 200, 400 |

---

### Order Processing Task Service

| Endpoint | Method | Media Type | Response Codes |
|---------|--------|------------|---------------|
| /order-processing | POST | application/json | 201, 400, 402, 409 |
| /order-processing/{id} | GET | application/json | 200, 404 |

### 3.2 Service Logic Design

Internal processing flow for each service.

**Authentication Service:**

![Auth Flow](asset/AuthService.png)

**Patient Service:**

![Patient Flow](asset/PatientService.png)

**Staff Service:**

![Staff Flow](asset/StaffService.png)

**Appointment Service:**

![Appointment Flow](asset/AppointmentService.png)

**Medication Service:**

![Medication Flow](asset/MedicationService.png)

**Notification Service:**

![Notification Flow](asset/NotiService.png)

---

