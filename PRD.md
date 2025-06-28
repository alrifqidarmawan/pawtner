```md
# Project Brief: Pawtner - The All-in-One Pet Care Application

## 1. Executive Summary

**Pawtner** is a mobile application designed to be the ultimate partner for modern pet owners. It aims to solve several critical, everyday challenges by consolidating pet-related needs into a single, user-friendly platform. Pawtner connects pet owners (Customers) with local pet businesses (vets, shops, daycares) to facilitate product purchases, service bookings, and pet health record management.

The initial development phase (Phase 1) will focus on delivering a **Minimum Viable Product (MVP)** within a **10-day timeframe**, directly addressing the most pressing user issues.

## 2. Core Problem Statement

Today's pet owners face a fragmented and often analog ecosystem for pet care, leading to several key frustrations that Pawtner will solve:

*   **Limited Access to Emergency Assistance:** Difficulty in quickly finding nearby, open veterinary clinics with available emergency contact information during urgent situations.
*   **High and Unpredictable Shipping Costs:** Uncertainty surrounding shipping fees when ordering pet products online, which complicates budgeting.
*   **Difficulty Boarding Pets During Holidays:** Fully booked boarding facilities during peak seasons, coupled with inconsistent service policies and no way to check real-time availability.
*   **Vulnerable Paper-Based Prescriptions:** Veterinary prescriptions issued on paper are easily lost or damaged, jeopardizing a pet’s treatment plan.
*   **Lack of Medication Scheduling:** Owners must rely on memory to administer medication, increasing the risk of missed doses and negatively impacting a pet's recovery.
*   **Need for Reliable Advice:** A need for a quick, trustworthy source of information for general care questions, as well as clear guidance toward professional help for serious issues.

## 3. Proposed Solution & Core Features (MVP)

Pawtner will solve these problems through a set of interconnected features built on a lean but powerful database.

*   **Feature 1: Local Business Discovery**
    *   **Solves:** Emergency Assistance & Holiday Boarding.
    *   **Description:** Allows customers to search for pet businesses filtered by their local area (`ward`), displaying comprehensive profiles with addresses, emergency contacts, and hours of operation.

*   **Feature 2: Integrated E-commerce**
    *   **Solves:** High Shipping Costs.
    *   **Description:** An in-app marketplace where businesses can list products. A transparent checkout process will clearly display the `shipping_cost` before payment.

*   **Feature 3: Centralized Service Booking System**
    *   **Solves:** Holiday Boarding Difficulties.
    *   **Description:** A system for booking services like boarding and grooming. It will clearly display service policies (`requires_food_from_owner`, `can_accommodate_sick_pets`) and use a **real-time capacity check** to prevent overbooking.

*   **Feature 4: Digital Pet Health Records**
    *   **Solves:** Lost Prescriptions & Medication Scheduling Foundation.
    *   **Description:** A digital profile for each pet where vets can issue digital `prescriptions`, creating a permanent, easily accessible record of medication history.

*   **Feature 5: AI-Powered Assistant (Frontend Only)**
    *   **Solves:** Need for Reliable Advice.
    *   **Description:** A chat interface for general questions. The AI will use the pet's profile for context and will be programmed to direct users to professionals for serious queries.

## 4. Database-Driven Problem Solving

The proposed database schema is the backbone for each solution.

| Problem to Solve                            | Key Tables & Columns                                                                       | How the Database Solves It                                                                                                                                                                             |
| ------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Limited Access to Emergency Assistance**  | `businesses` (`emergency_phone`, `ward`, `operation_hours`)                                | Users can filter businesses by location (`ward`) and immediately access critical data like the `emergency_phone` and operating hours for a quick response.                                             |
| **High and Unpredictable Shipping Costs**   | `orders` (`shipping_cost`, `total_amount`), `order_items`                                  | The dedicated `shipping_cost` column ensures that delivery fees are calculated and displayed transparently upfront, eliminating hidden costs and building trust.                                        |
| **Difficulty Boarding Pets During Holidays**| `services` (`capacity_per_day`, `requires_food_...`), `bookings`                             | Service policies are clearly displayed. The system automatically validates availability by checking `capacity_per_day` against existing `bookings` to prevent overbooking.                                |
| **Vulnerable Paper-Based Prescriptions**    | `prescriptions`, `pets`, `businesses`                                                      | Prescriptions are stored digitally and permanently linked to the pet's profile (`pet_id`), creating a secure and loss-proof record.                                                                   |
| **Lack of Medication Scheduling**           | `prescriptions` (`dosage`, `frequency`, `instructions`)                                    | (Phase 1) Provides a digital "source of truth" for treatment instructions, solving the problem of forgetting details and laying the groundwork for future automated reminders.                          |

## 5. Database Schema (MVP)

This schema is designed for rapid development and efficiency.

```sql
-- Manages application users (customers and business owners)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(50) UNIQUE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('business_owner', 'customer')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Profiles for all business entities (shops, clinics, daycares)
CREATE TABLE businesses (
    id SERIAL PRIMARY KEY,
    owner_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    business_email VARCHAR(255),
    business_phone VARCHAR(50),
    emergency_phone VARCHAR(50),
    address TEXT,
    ward VARCHAR(100) NOT NULL,
    operation_hours JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Catalog of physical products sold by businesses
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

-- Catalog of offered services (boarding, grooming, etc.)
CREATE TABLE services (
    id SERIAL PRIMARY KEY,
    business_id INT NOT NULL REFERENCES businesses(id),
    category VARCHAR(50) NOT NULL CHECK (category IN ('boarding', 'daycare', 'grooming', 'veterinary')),
    name VARCHAR(255) NOT NULL,
    base_price DECIMAL(10, 2) NOT NULL,
    requires_food_from_owner BOOLEAN DEFAULT false,
    can_accommodate_sick_pets BOOLEAN DEFAULT false,
    capacity_per_day INT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Records each product purchase transaction
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES users(id),
    order_number VARCHAR(255) UNIQUE NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    shipping_cost DECIMAL(10, 2) DEFAULT 0,
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'paid', 'shipped', 'completed', 'cancelled')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Junction table for line items within a single order
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id),
    product_id INT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    price_at_purchase DECIMAL(10, 2) NOT NULL
);

-- Records each reservation for a service
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

-- Profiles for the customers' pets
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

-- Digital records for veterinary prescriptions
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

## 6. Key Implementation Flows

*   **Onboarding & Profiling:** New users sign up (`users`). Customers then create `pets` profiles, while business owners create `businesses` profiles and populate their `products` and `services` catalogs.
*   **Service Booking Flow:** A customer searches for a service by `ward`. The app displays relevant services with their policies. When a date range is selected, the system validates availability by comparing existing `bookings` against the `services.capacity_per_day` before confirming.
*   **Product Order Flow:** A customer fills their shopping cart. At checkout, the system creates one entry in `orders` (for the transaction total and shipping) and multiple entries in `order_items` (for each product purchased).
*   **Prescription Management Flow:** A veterinarian (`business_owner`) creates a new entry in the `prescriptions` table via their dashboard, which becomes immediately visible to the customer on their pet's profile.

## 7. Next Steps & Technical Recommendations

### a. Recommended Technology Stack
*   **Database:** **PostgreSQL** – Ideal for its `JSONB` support and robust, production-ready nature.
*   **Backend:** **Node.js** (with Express.js/NestJS) – Fast for API development and highly efficient.
*   **Frontend (Mobile):** **Flutter** or **React Native** – Accelerates cross-platform (iOS & Android) development.
*   **Authentication:** **JWT (JSON Web Tokens)** – The secure standard for managing user sessions.

### b. Proposed 10-Day MVP Sprint Plan
*   **Days 1-2: Setup & Backend Foundation.**
    *   Initialize Git repository, set up the database, and run schema migrations.
    *   Implement user authentication endpoints (register & login).
*   **Days 3-5: Core Feature Backend Development.**
    *   Build APIs for CRUD (Create, Read, Update, Delete) operations on `businesses`, `products`, and `services`.
    *   Develop the critical logic for product ordering and service booking (including the availability check).
*   **Days 6-8: Frontend Development.**
    *   Build the UI for all primary user flows: authentication, business discovery, product browsing, shopping cart, booking, and pet profiles.
*   **Days 9-10: Integration, Testing, & Finalization.**
    *   Connect all frontend screens to the backend APIs.
    *   Conduct end-to-end testing of core user journeys.
    *   Fix bugs and prepare for deployment.

## 8. Development Roadmap (Beyond MVP)

After a successful MVP launch, Pawtner can be enhanced with the following features:

*   **Advanced Medication Management (Phase 2):**
    *   **Feature:** Add a `medication_schedules` table linked to `prescriptions` to auto-generate schedules.
    *   **Benefit:** Enables **push notifications** for dose reminders and allows users to log compliance, which can be shared with their vet.

*   **Business Analytics Dashboard:**
    *   **Feature:** A web-based dashboard for business owners to track sales trends, booking analytics, and manage inventory.
    *   **Benefit:** Provides significant value, opening up a potential SaaS subscription model.

*   **Community & Review Features:**
    *   **Feature:** A rating and review system for businesses and services.
    *   **Benefit:** Builds trust and community, driving user engagement and informed decision-making.

*   **Payment & Logistics Integration:**
    *   **Feature:** Integrate with payment gateways (e.g., Stripe) for in-app transactions and logistics APIs for real-time shipping calculations.
    *   **Benefit:** Automates and streamlines the payment and delivery processes.
```
