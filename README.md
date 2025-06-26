# Pawtner: The All-in-One Pet Care Platform

## 1. Executive Summary

**Pawtner** is a digital platform designed to be the ultimate partner for modern pet owners. It addresses critical daily challenges by consolidating pet-related needs—including product commerce, service booking, and health management—into a single, user-friendly mobile application. The platform connects pet owners (Customers) with local pet businesses (Veterinarians, Shops, Daycares) to create a seamless and integrated care ecosystem with **secure, integrated payment processing powered by Midtrans**.

This document outlines the plan for a **Minimum Viable Product (MVP)** to be delivered in a series of focused sprints. The goal is to solve the most pressing user problems and validate the core transactional value proposition of the application.

## 2. Core Problem Statement

The current pet care landscape is fragmented, leading to significant frustrations for pet owners. Pawtner aims to solve the following key issues:

*   **Limited Access to Emergency Assistance:** Difficulty in quickly locating nearby, open veterinary clinics with available emergency contact information during urgent situations.
*   **Fragmented & Insecure E-commerce:** The online purchasing and service booking experience is often clunky, lacks transparent costs, and requires users to leave the app for payment, creating a disjointed and untrustworthy process.
*   **Holiday Boarding Challenges:** During peak seasons, pet boarding facilities are frequently overbooked. Owners face inconsistent service standards, unclear policies, and no real-time availability checks.
*   **Vulnerable Paper-Based Prescriptions:** Reliance on paper prescriptions creates a high risk of loss or damage, jeopardizing a pet's treatment regimen and continuity of care.
*   **Absence of Medication Tracking:** Without a digital system, owners must rely on memory for medication administration, increasing the risk of missed doses and negatively impacting a pet's recovery.
*   **Need for Reliable Advice & Trust:** A gap exists for a quick, trustworthy source of information. Users need to trust businesses, products, and services, which is difficult without community feedback or a clear verification process.

## 3. Proposed Solution & Core Features (MVP)

Pawtner will address these problems through a suite of interconnected features built upon a robust database architecture.

*   **Feature 1: Verified Local Business Discovery:** A searchable directory of pet businesses with verified badges to build trust.
*   **Feature 2: Multi-Store Integrated Marketplace:** An in-app marketplace with a separate shopping cart for each business.
*   **Feature 3: Centralized Service Booking:** A robust system for booking services with real-time capacity checks to prevent overbooking.
*   **Feature 4: Digital Pet Health Records & Prescriptions:** Digital pet profiles where vets can issue, and owners can view, permanent and accessible medication histories.
*   **Feature 5: Reviews and Rating System:** A comprehensive review system for businesses, products, and services to build a trusted community.
*   **Feature 6: AI-Powered Assistant (Client-Side):** A chat interface for general queries, escalating serious questions to professionals.
*   **Feature 7: Integrated & Secure Payment Gateway (Midtrans):** A fully integrated payment system allowing users to pay directly within the app, completing the business cycle and solidifying user trust.

## 4. Problem-Solution Mapping via Database Design

| Problem | Relevant Tables | Key Columns | Solution Mechanism |
| :--- | :--- | :--- | :--- |
| **Limited Emergency Access** | `businesses` | `emergency_phone`, `longitude`, `latitude`, `status_realtime` | Enables rapid, location-based filtering to display crucial contact details and real-time operational status. |
| **Complex E-commerce & Booking** | `shopping_carts`, `orders`, `bookings`, **`payments`** | `payments.snap_token`, `orders.status`, `bookings.status` | **A complete transactional flow.** A user checks out, an order/booking is created with `pending_payment` status, and Midtrans is called for a `snap_token`. A successful payment (confirmed via webhook) automatically updates the order status to `processing` or `confirmed`. |
| **Holiday Boarding Difficulty** | `services`, `bookings`, **`payments`** | `services.capacity_per_day`, `bookings.start_time`, `bookings.status` | Real-time availability is checked on booking request. The slot is only truly secured (`status = 'confirmed'`) **after a successful payment** via Midtrans, preventing "ghost" bookings from blocking capacity. |
| **Vulnerable Prescriptions** | `prescriptions`, `prescription_items` | `prescriptions.pet_id`, `prescription_items.medication_name` | Full digitalization of prescriptions, linked to a pet's profile, creating a permanent, secure, and easily accessible record. |
| **Lack of Medication Schedule** | `prescription_items` | `medication_name`, `dosage`, `frequency`, `duration_days` | Provides a detailed, reliable digital reference for each medication, solving the core problem of forgotten details. |
| **Need for Trusted Advice** | `reviews`, `businesses` | `reviews.rating`, `businesses.is_verified` | Users can make informed decisions based on community `reviews`. The `is_verified` badge, backed by an admin workflow, builds a foundational layer of trust. |

## 5. Detailed Database Schema & Explanation

This is the complete and optimized database schema for the MVP.

### 5.1. SQL Schema

```sql
-- Manages user accounts for both customers and business owners
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    code VARCHAR(255),
    code_expired TIMESTAMP,
    phone_number VARCHAR(50) UNIQUE,
    password_hash VARCHAR(255), -- Can be null for OAuth providers
    address TEXT,
    latitude DECIMAL(9, 6),
    longitude DECIMAL(9, 6),
    image_url VARCHAR(255),
    is_verified BOOLEAN DEFAULT false,
    role VARCHAR(50) NOT NULL CHECK (role IN ('business_owner', 'customer')),
    auth_provider VARCHAR(50) NOT NULL DEFAULT 'local',
    provider_id VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (auth_provider, provider_id)
);

-- Contains profiles for all service-providing businesses
CREATE TABLE businesses (
    id SERIAL PRIMARY KEY,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    business_email VARCHAR(255) UNIQUE,
    business_phone VARCHAR(50),
    business_image_url VARCHAR(255),
    certificate_image_url VARCHAR(255), -- For verification process
    emergency_phone VARCHAR(50),
    address TEXT,
    latitude DECIMAL(9, 6),
    longitude DECIMAL(9, 6),
    is_verified BOOLEAN DEFAULT false,  
    operation_hours JSONB,
    status_realtime VARCHAR(50) DEFAULT 'Closed', -- e.g., 'Accepting Patients', 'At Capacity', 'Closed'
    last_status_update TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Contains profiles for all customer-owned pets
CREATE TABLE pets (
    id SERIAL PRIMARY KEY,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    species VARCHAR(100),
    breed VARCHAR(100),
    birth_date DATE,
    image_url VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- A catalog of all physical products sold by businesses
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    business_id INT NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(255) NOT NULL CHECK (category IN ('food', 'toys', 'accessories', 'health', 'grooming_kit')),
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    image_url VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- A catalog of all services (e.g., boarding, grooming) offered by businesses
CREATE TABLE services (
    id SERIAL PRIMARY KEY,
    business_id INT NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    category VARCHAR(50) NOT NULL CHECK (category IN ('boarding', 'daycare', 'grooming', 'veterinary')),
    name VARCHAR(255) NOT NULL,
    service_image_url VARCHAR(255),
    base_price DECIMAL(10, 2) NOT NULL,
    requires_food_from_owner BOOLEAN DEFAULT false,
    can_accommodate_sick_pets BOOLEAN DEFAULT false,
    available_at VARCHAR(50) NOT NULL CHECK (available_at IN ('in_clinic', 'at_home', 'both')),    
    capacity_per_day INT, -- For real-time availability checks on services like boarding
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Represents a user's shopping cart for a specific business
CREATE TABLE shopping_carts (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    business_id INT NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    -- A customer can only have one active cart per business at a time
    UNIQUE (customer_id, business_id)
);

-- Contains the individual products within a shopping cart
CREATE TABLE cart_items (
    id SERIAL PRIMARY KEY,
    cart_id INT NOT NULL REFERENCES shopping_carts(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    quantity INT NOT NULL CHECK (quantity > 0),
    added_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    -- A product can only appear once in a specific cart; quantity should be updated instead
    UNIQUE (cart_id, product_id)
);

-- Represents a transaction receipt for product purchases
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES users(id),
    business_id INT NOT NULL REFERENCES businesses(id), -- Specifies which business the order is for
    order_number VARCHAR(255) UNIQUE NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    address TEXT,
    special_instructions TEXT,
    shipping_cost DECIMAL(10, 2) DEFAULT 0,
    status VARCHAR(50) NOT NULL DEFAULT 'pending_payment' CHECK (status IN ('pending_payment', 'processing', 'shipped', 'completed', 'cancelled', 'failed', 'refunded')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Junction table for many-to-many relationship between orders and products
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    price_at_purchase DECIMAL(10, 2) NOT NULL -- Logs the price at the time of sale
);

-- Represents a reservation for a specific service
CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES users(id),
    pet_id INT NOT NULL REFERENCES pets(id),
    service_id INT NOT NULL REFERENCES services(id),
    booking_number VARCHAR(255) UNIQUE NOT NULL,
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,
    special_instructions TEXT,
    total_price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'requested' CHECK (status IN ('requested', 'confirmed', 'cancelled', 'completed')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- [NEW & IMPORTANT] Records all payment transactions processed via Midtrans
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id) ON DELETE SET NULL,      -- Link to a product order
    booking_id INT REFERENCES bookings(id) ON DELETE SET NULL,  -- Link to a service booking
    payment_gateway_ref_id VARCHAR(255) UNIQUE NOT NULL,        -- The unique order_id from Midtrans
    amount DECIMAL(10, 2) NOT NULL,
    payment_method VARCHAR(100),                                -- e.g., 'gopay', 'bca_va', will be filled by webhook
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'settlement', 'capture', 'expire', 'failure', 'cancel')),
    snap_token VARCHAR(255),                                    -- The token used by the frontend to render the payment pop-up
    snap_redirect_url TEXT,
    payment_expiry_time TIMESTAMP WITH TIME ZONE,
    webhook_payload JSONB,                                      -- To store the full notification from Midtrans for auditing
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    -- A payment must be for EITHER an order OR a booking, not both.
    CONSTRAINT chk_payment_target CHECK (
        (CASE WHEN order_id IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN booking_id IS NOT NULL THEN 1 ELSE 0 END) = 1
    )
);

-- A digital record of veterinary prescriptions (the container)
CREATE TABLE prescriptions (
    id SERIAL PRIMARY KEY,
    pet_id INT NOT NULL REFERENCES pets(id),
    issuing_business_id INT NOT NULL REFERENCES businesses(id),
    issue_date DATE NOT NULL DEFAULT CURRENT_DATE,
    notes TEXT, -- General notes from the vet about the overall condition or prescription
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Individual medication items within a single prescription
CREATE TABLE prescription_items (
    id SERIAL PRIMARY KEY,
    prescription_id INT NOT NULL REFERENCES prescriptions(id) ON DELETE CASCADE,
    medication_name VARCHAR(255) NOT NULL,
    dosage VARCHAR(255) NOT NULL, -- e.g., '1 tablet', '5ml'
    frequency VARCHAR(255) NOT NULL, -- e.g., 'Twice a day', 'Every 8 hours'
    duration_days INT NOT NULL, -- e.g., 7, 14
    instructions TEXT -- e.g., 'With food', 'Before sleep'
);

-- Reviews for businesses, products, or services
CREATE TABLE reviews (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    business_id INT REFERENCES businesses(id) ON DELETE CASCADE,
    product_id INT REFERENCES products(id) ON DELETE CASCADE,
    service_id INT REFERENCES services(id) ON DELETE CASCADE,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    -- A review must be for EITHER a business, a product, OR a service, but not multiple.
    CONSTRAINT check_review_target CHECK (
        (CASE WHEN business_id IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN product_id IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN service_id IS NOT NULL THEN 1 ELSE 0 END) = 1
    )
);
```

### 5.2. Explanation of Key SQL Concepts

*   **`ON DELETE CASCADE`**: This rule ensures data integrity. For example, if a `user` deletes their account, all their `pets`, `bookings`, and `shopping_carts` are automatically deleted, preventing orphaned data.
*   **`UNIQUE` Constraints**: These are used to enforce business rules. For example, `UNIQUE (customer_id, business_id)` in `shopping_carts` ensures a user can only have one cart per store.
*   **`CHECK` Constraints**: These enforce valid values for a column, like ensuring a `rating` is between 1 and 5, or that a `role` is either 'customer' or 'business_owner'.
*   **`payments.chk_payment_target`**: This is a powerful constraint ensuring that a payment record is linked to *either* an order *or* a booking, but never both. It guarantees that the purpose of every payment is clear.
*   **`reviews.check_review_target`**: Similarly, this constraint ensures a review is specific and targets only one entity (a business, a product, or a service).

## 6. Implementation Flow and Core Workflows

### E-commerce Workflow (Add to Cart -> Successful Payment)
1.  **Add to Cart**: User adds a product. Backend manages `shopping_carts` and `cart_items`.
2.  **Checkout**:
    *   User clicks "Checkout". Backend creates an `orders` record with `status = 'pending_payment'`.
    *   Backend calls the **Midtrans API** to create a transaction, receiving a `snap_token`.
    *   Backend creates a `payments` record with `status = 'pending'` and saves the `snap_token`.
    *   The `snap_token` is sent back to the frontend.
3.  **Payment**: Frontend uses the `snap_token` to render the Midtrans Snap payment pop-up. User completes payment.
4.  **Confirmation (Webhook)**:
    *   Midtrans sends a webhook notification to our backend.
    *   Backend verifies the webhook, updates the `payments` status to `'settlement'`, and the `orders` status to `'processing'`.
    *   Finally, it decrements `products.stock_quantity` and clears the user's `shopping_cart`.

### Service Booking Workflow
1.  **Availability Check**: User selects a service and dates. Backend validates `services.capacity_per_day` against existing confirmed `bookings`.
2.  **Request Booking**: If available, the flow mirrors e-commerce: create a `bookings` record (`status = 'requested'`), call Midtrans, create a `payments` record, and return a `snap_token`.
3.  **Payment & Confirmation**: Upon successful payment webhook, the `bookings` status is changed to `'confirmed'`. The slot is now officially reserved.

## 7. Technical Specifications and Execution Plan

### a. Recommended Technology Stack
| Component | Technology |
|---|---|
| **Database** | PostgreSQL |
| **Backend** | Java with Spring Boot |
| **Frontend (Mobile)** | React Native |
| **Frontend (Web)** | React.js |
| **Authentication** | Spring Security with JWT |
| **Payment Gateway** | **Midtrans** |

### b. High-Level API Endpoint Design
- `AuthController`: `/api/auth/register`, `/api/auth/login`
- `BusinessController`: `GET /api/businesses`, `GET /api/businesses/{id}`
- `CartController`: `GET /api/carts`, `POST /api/carts/items`
- `OrderController`: `POST /api/orders/checkout`
- **`PaymentController`**:
    - `POST /api/orders/{orderId}/pay` -> Returns a `snap_token`.
    - `POST /api/bookings/{bookingId}/pay` -> Returns a `snap_token`.
    - `POST /api/payments/webhook` -> Public endpoint for Midtrans notifications.

### c. Realistic MVP Sprint Plan (Focused Scope)

To ensure high quality, the MVP will focus on validating **one core business cycle: E-commerce End-to-End**.

**Sprint 1 (Weeks 1-2): Foundation & Core E-commerce Logic**
- **Goal**: Build the complete backend logic for a user to buy a product.
- **Tasks**: Setup project, implement all entities, configure Spring Security, implement logic for `Carts` & `Orders`, and **critically, implement the `PaymentService` to integrate with the Midtrans API (create transaction, handle webhook).**

**Sprint 2 (Weeks 3-4): Frontend Development & Integration**
- **Goal**: Create the user-facing application for the e-commerce flow and connect it to the backend.
- **Tasks**:
    - **React Native (Customer App)**: Build screens for Login, Product Discovery, Cart, and **implement the Checkout screen with Midtrans Snap.js integration.**
    - **React.js (Business Dashboard)**: Build screens for Login, Business/Product Management, and viewing incoming orders.
    - **Integration**: Connect all frontend components to the backend APIs.

**Sprint 3 (Week 5): Testing, Refinement & Deployment**
- **Goal**: Ensure the application is stable, bug-free, and ready for deployment.
- **Tasks**: End-to-end testing of the entire user journey. **Crucially, test the payment flow thoroughly using the Midtrans Sandbox and webhook simulator.** Bug fixing, UI polishing, and deployment.

## 8. Roadmap Beyond MVP

### Phase 2: Service Expansion
-   **Service Booking System**: Implement the full booking workflow, leveraging the already-built payment logic.
-   **Digital Prescriptions**: Build the interface for Vets to issue, and customers to view, digital prescriptions.
-   **Enhanced Reviews**: Add features like replies to reviews and content flagging.

### Future Enhancements
-   **AI-Powered Assistant**: Implement the client-side AI chat for general advice.
-   **Business Intelligence Dashboard**: Provide analytics for business owners on sales and booking trends.
-   **Advanced Logistics**: Integrate with shipping APIs for real-time delivery tracking.
-   **Push Notifications**: Implement reminders for medication, appointments, and order status updates.
