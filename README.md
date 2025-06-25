# Project Brief: Pawtner - The All-in-One Pet Care Platform

## 1. Executive Summary

**Pawtner** is a digital platform designed to serve as the ultimate partner for modern pet owners. It addresses critical daily challenges by consolidating pet-related needs—including product commerce, service booking, and health management—into a single, user-friendly mobile application. The platform connects pet owners (Customers) with local pet businesses (Veterinarians, Shops, Daycares) to create a seamless and integrated care ecosystem.

The initial development phase will focus on delivering a **Minimum Viable Product (MVP)** within an aggressive 10-day sprint. This MVP is laser-focused on solving the most pressing user problems to validate the core value proposition of the application.

## 2. Core Problem Statement

The current pet care landscape is fragmented and heavily reliant on analog processes, leading to significant frustrations for pet owners. Pawtner aims to solve the following key issues:

*   **Limited Access to Emergency Assistance:** Difficulty in quickly locating nearby, open veterinary clinics with available emergency contact information during urgent situations.
*   **Unpredictable E-commerce Costs:** The online purchasing experience for pet products is often marred by unexpected shipping fees, complicating budget management and obscuring local delivery options.
*   **Holiday Boarding Challenges:** During peak seasons, pet boarding facilities are frequently overbooked. Owners encounter inconsistent service standards, unclear policies (e.g., food provision), and a lack of real-time availability checking.
*   **Vulnerable Paper-Based Prescriptions:** Reliance on paper prescriptions creates a high risk of loss or damage, which can jeopardize a pet's treatment regimen and continuity of care.
*   **Absence of Medication Tracking:** Without a digital system, owners must rely on memory for medication administration, increasing the risk of missed doses and negatively impacting a pet's recovery.
*   **Need for Reliable Advice:** A gap exists for a quick, trustworthy source of information for general pet care questions, which can also intelligently escalate serious queries to professional help.

## 3. Proposed Solution & Core Features (MVP)

Pawtner will address these problems through a suite of interconnected features built upon a lean yet powerful database architecture.

*   **Feature 1: Local Business Discovery**
    *   **Solves:** Emergency Assistance, Boarding Challenges.
    *   **Description:** A searchable directory of pet businesses, filterable by local area (`ward`). Profiles will feature essential details like address, contact numbers, a dedicated emergency phone, and operating hours.

*   **Feature 2: Integrated Marketplace**
    *   **Solves:** Unpredictable E-commerce Costs.
    *   **Description:** An in-app marketplace for businesses to list products. The checkout process supports multi-item carts and transparently displays `shipping_cost` before purchase confirmation.

*   **Feature 3: Centralized Service Booking**
    *   **Solves:** Holiday Boarding Challenges.
    *   **Description:** A robust system for booking services (e.g., boarding, grooming). It will clearly display service policies (`requires_food_from_owner`, `can_accommodate_sick_pets`) and enforce a real-time capacity check to prevent overbooking, guaranteeing availability upon booking.

*   **Feature 4: Digital Pet Health Records**
    *   **Solves:** Vulnerable Prescriptions, Foundation for Medication Tracking.
    *   **Description:** Digital profiles for each pet where veterinarians can issue, and owners can view, digital `prescriptions`. This creates a permanent, accessible medication history, eliminating the risks associated with paper records.

*   **Feature 5: AI-Powered Assistant (Client-Side)**
    *   **Solves:** Need for Reliable Advice.
    *   **Description:** A chat interface for general care queries, using the pet's profile data for context. For urgent or serious questions, the AI will escalate by directing the user to the Local Business Discovery feature.

## 4. Problem-Solution Mapping via Database Design

This section details how the database schema is architected to solve each identified problem.

| Masalah (Problem) | Tabel Terkait (Relevant Tables) | Kolom Kunci (Key Columns) | Cara Penyelesaian (Solution Mechanism) |
| :--- | :--- | :--- | :--- |
| **Akses Darurat Terbatas** | `businesses` | `emergency_phone`, `ward`, `operation_hours` | Enables rapid, location-based filtering (`ward`) to display crucial contact details (`emergency_phone`) and operational status (`operation_hours`). |
| **Biaya Pengiriman Tidak Terduga** | `orders`, `order_items` | `orders.shipping_cost` | The `shipping_cost` is an explicit, calculated field, ensuring full cost transparency for the user before payment commitment. |
| **Kesulitan Penitipan Saat Libur** | `services`, `bookings` | `services` (policy booleans, `capacity_per_day`) | Policies are displayed upfront for informed decisions. Real-time availability is enforced by cross-referencing new booking requests against existing `bookings` and the service's `capacity_per_day`. |
| **Resep Rentan Hilang** | `prescriptions`, `pets` | All columns in `prescriptions` | Digitalizes the entire prescription record, linking it immutably to a pet's profile (`pet_id`). This creates a permanent, secure, and easily accessible source of truth. |
| **Kurangnya Jadwal Obat** | `prescriptions` | `dosage`, `frequency`, `instructions` | Provides a reliable, digital reference for administration instructions, solving the foundational issue of forgotten details and laying the groundwork for future automated reminders. |

## 5. Database Schema (MVP)

This is the optimized database schema designed for the 10-day development sprint.

```sql
-- Manages user accounts for both customers and business owners
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(50) UNIQUE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('business_owner', 'customer')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Contains profiles for all service-providing businesses
CREATE TABLE businesses (
    id SERIAL PRIMARY KEY,
    owner_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    business_email VARCHAR(255),
    business_phone VARCHAR(50),
    emergency_phone VARCHAR(50),
    address TEXT,
    ward VARCHAR(100) NOT NULL, -- Key for location-based filtering
    operation_hours JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- A catalog of all physical products sold by businesses
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    business_id INT NOT NULL REFERENCES businesses(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    image_url VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- A catalog of all services (e.g., boarding, grooming) offered by businesses
CREATE TABLE services (
    id SERIAL PRIMARY KEY,
    business_id INT NOT NULL REFERENCES businesses(id),
    category VARCHAR(50) NOT NULL CHECK (category IN ('boarding', 'daycare', 'grooming', 'veterinary')),
    name VARCHAR(255) NOT NULL,
    base_price DECIMAL(10, 2) NOT NULL,
    requires_food_from_owner BOOLEAN DEFAULT false,
    can_accommodate_sick_pets BOOLEAN DEFAULT false,
    capacity_per_day INT, -- For real-time availability checks
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Represents a transaction receipt for product purchases
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES users(id),
    order_number VARCHAR(255) UNIQUE NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    shipping_cost DECIMAL(10, 2) DEFAULT 0,
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'paid', 'shipped', 'completed', 'cancelled')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Junction table to facilitate many-to-many relationship between orders and products
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id),
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
    total_price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) NOT NULL CHECK (status IN ('requested', 'confirmed', 'cancelled', 'completed')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Contains profiles for all customer-owned pets
CREATE TABLE pets (
    id SERIAL PRIMARY KEY,
    owner_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    species VARCHAR(100),
    breed VARCHAR(100),
    birth_date DATE,
    image_url VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- A digital record of veterinary prescriptions
CREATE TABLE prescriptions (
    id SERIAL PRIMARY KEY,
    pet_id INT NOT NULL REFERENCES pets(id),
    issuing_business_id INT NOT NULL REFERENCES businesses(id),
    medication_name VARCHAR(255) NOT NULL,
    dosage VARCHAR(255) NOT NULL,
    frequency VARCHAR(255) NOT NULL,
    duration_days INT NOT NULL,
    instructions TEXT,
    issue_date DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```
## 6. Implementation Flow and Core Workflows

### User Onboarding
A new user registers via the React Native mobile app, creating a record in the `users` table. Based on their selected `role`, they are directed to either:
- Create `pet` profiles (for customers)
- Create a `business` profile (for business owners)

Business owners will primarily use a separate React.js web dashboard to manage their profiles, `products`, and `services` catalogs.

### Service Booking Workflow (Mobile App)
1. Customer searches for a `boarding` service within their `ward`
2. React Native app calls Spring Boot API, which:
   - Queries `businesses` by location
   - Joins with `services` to retrieve relevant offerings
3. Results display service policies transparently
4. Upon date range selection:
   - Backend validates availability by counting conflicting `bookings` against service's `capacity_per_day`
   - Successful validation → new record in `bookings` table
   - Failure → returns "at capacity" error

### E-commerce Workflow (Mobile App)
1. Customer adds items to cart (state managed in React Native)
2. At checkout:
   - Backend receives cart data
   - Calculates `total_amount` including `shipping_cost`
3. Creates:
   - Parent record in `orders` table
   - Child records in `order_items` for each product

### Prescription Management Workflow (Web Dashboard)
1. Veterinarian (`business_owner`) logs into React.js dashboard
2. Accesses patient's `pet` profile
3. Submits form → creates new record in `prescriptions` table
4. Customer instantly views digital record in React Native app

## 7. Technical Specifications and Execution Plan

### a. Recommended Technology Stack
| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Database** | PostgreSQL | Robust, reliable, native `JSONB` support works well with JPA/Hibernate |
| **Backend** | Java with Spring Boot | Enterprise-grade REST APIs with Spring Data JPA, Spring Web, Spring Security |
| **Frontend (Mobile)** | React Native | Cross-platform for iOS/Android |
| **Frontend (Web)** | React.js | Web dashboard for business owners |
| **Authentication** | Spring Security with JWT | Industry-standard stateless API security |

### b. High-Level API Endpoint Design
**Core Controllers:**
- `AuthController`:
  - `POST /api/auth/register`
  - `POST /api/auth/login`
- `UserController`:
  - `GET /api/users/me`
- `BusinessController`:
  - `GET /api/businesses`
- `ProductController`:
  - Product catalog management
- `ServiceController`:
  - Service catalog browsing
- `OrderController`:
  - `POST /api/orders`
- `BookingController`:
  - `POST /api/bookings/check-availability`
- `PetController`:
  - Pet profile management
- `PrescriptionController`:
  - Veterinary prescriptions

### c. 10-Day MVP Sprint Plan
**Days 1-2: Foundation & Backend Setup**
- Initialize Git repos (Backend, Mobile App, Web Dashboard)
- Set up Spring Boot project
- Configure PostgreSQL connection
- Define JPA entities
- Implement Spring Security (JWT auth)

**Days 3-5: Backend Core Logic**
- Build REST controllers/services/repositories for:
  - `businesses`
  - `products`
  - `services`
  - `pets`
- Implement:
  - Order creation logic
  - Booking availability check

**Days 6-8: Frontend Development**
- **React Native Team:**
  - Login screen
  - Business Directory
  - Product Catalogs
  - Pet Profiles
  - Cart/Checkout
- **React.js Team:**
  - Login screen
  - Business Profile Management
  - Product/Service Management

**Days 9-10: Integration & Deployment**
- Connect frontends to backend APIs
- End-to-end testing:
  - Customer journey (mobile)
  - Business journey (web)
- Bug fixes & UI refinements
- Deploy backend to cloud (Heroku/AWS)

## 8. Roadmap Beyond MVP
### Phase 2
- **Enhanced Medication Management**
  - `@Scheduled` tasks for dose reminders
  - Push notification system

### Future Enhancements
- **Business Intelligence Dashboard**
  - Chart.js analytics
  - Sales/booking trends
- **Community Features**
  - Reviews/ratings system
- **Financial Integration**
  - Payment gateway (Stripe/Midtrans)
  - `PaymentController`
- **Advanced Features**
  - ML recommendations (Weka/Deeplearning4j)
  - Mapping/shipping API integration
