# PostgreSQL Database Schema
## AI-First Auto Shop Management Platform - MVP

**Version:** 1.0
**Last Updated:** 2026-01-18
**Implementation:** PostgreSQL 15+

---

## 1. Entities Overview

| Entity | Purpose | Relationships |
|--------|---------|---------------|
| **shops** | Tenant/shop account (multi-tenant isolation) | One shop has many users, customers, vehicles, ROs, etc. |
| **users** | Authentication and staff accounts | Belongs to shop, has role, creates ROs and inspections |
| **customers** | Customer contact information | Belongs to shop, has many vehicles and ROs |
| **vehicles** | Vehicle records tied to customers | Belongs to customer, has many ROs |
| **repair_orders** | Core workflow entity (RO lifecycle) | Belongs to vehicle/customer, has many line items, inspections, invoices |
| **line_items** | Labor/parts on an RO | Belongs to repair order |
| **inspections** | Multi-point inspection records | Belongs to repair order, has many inspection points |
| **inspection_points** | Individual check items in inspection | Belongs to inspection, has many photos |
| **photos** | Images for inspections and ROs | Belongs to inspection point or repair order |
| **appointments** | Scheduled service appointments | Belongs to shop, links to customer/vehicle/RO |
| **bays** | Service bay/lift assignments | Belongs to shop, used in appointments and ROs |
| **inventory_items** | Parts inventory tracking | Belongs to shop, has many stock transactions |
| **stock_transactions** | Inventory movements (in/out/adjust) | Belongs to inventory item |
| **invoices** | Billing records for completed work | Belongs to repair order, has many payments |
| **payments** | Payment records (cash/card/check) | Belongs to invoice |
| **notifications** | SMS/email notification log | Belongs to shop, links to customer/RO |
| **ai_predictions** | AI model outputs and tracking | Belongs to shop, links to various entities |

---

## 2. Table Definitions

### 2.1 Multi-Tenant: shops

```sql
CREATE TABLE shops (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL, -- URL-friendly identifier

    -- Contact
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(2),
    zip VARCHAR(10),

    -- Settings
    tax_rate DECIMAL(5,4) DEFAULT 0.0000, -- e.g., 0.0825 for 8.25%
    labor_rate DECIMAL(10,2) DEFAULT 120.00, -- $/hour
    timezone VARCHAR(50) DEFAULT 'America/New_York',

    -- Branding
    logo_url TEXT,

    -- Subscription
    plan VARCHAR(50) DEFAULT 'trial', -- trial, starter, professional, enterprise
    subscription_status VARCHAR(50) DEFAULT 'active', -- active, past_due, canceled
    trial_ends_at TIMESTAMP,

    -- Metadata
    settings JSONB DEFAULT '{}', -- flexible settings storage
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP -- soft delete
);

CREATE INDEX idx_shops_slug ON shops(slug);
CREATE INDEX idx_shops_deleted_at ON shops(deleted_at) WHERE deleted_at IS NULL;
```

### 2.2 Authentication: users

```sql
CREATE TYPE user_role AS ENUM ('owner', 'service_advisor', 'technician', 'front_desk', 'parts_manager');

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Authentication
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL, -- bcrypt

    -- Profile
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    role user_role NOT NULL,

    -- Employment
    employee_id VARCHAR(50),
    hourly_rate DECIMAL(10,2),
    hire_date DATE,

    -- Status
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,

    UNIQUE(shop_id, email)
);

CREATE INDEX idx_users_shop_id ON users(shop_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(shop_id, role) WHERE deleted_at IS NULL;
```

### 2.3 CRM: customers

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Contact
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(20) NOT NULL, -- primary contact method
    alternate_phone VARCHAR(20),

    -- Address
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(2),
    zip VARCHAR(10),

    -- Preferences
    preferred_contact VARCHAR(20) DEFAULT 'sms', -- sms, email, phone

    -- Metadata
    notes TEXT,
    tags TEXT[], -- ['fleet', 'vip', 'commercial']
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

CREATE INDEX idx_customers_shop_id ON customers(shop_id);
CREATE INDEX idx_customers_phone ON customers(shop_id, phone); -- fast lookup
CREATE INDEX idx_customers_email ON customers(shop_id, email);
CREATE INDEX idx_customers_name ON customers(shop_id, last_name, first_name);
CREATE INDEX idx_customers_deleted_at ON customers(shop_id) WHERE deleted_at IS NULL;
```

### 2.4 CRM: vehicles

```sql
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,

    -- Identification
    vin VARCHAR(17) UNIQUE, -- VIN is globally unique
    license_plate VARCHAR(20),

    -- Vehicle Info (from VIN decode or manual entry)
    year INTEGER NOT NULL,
    make VARCHAR(100) NOT NULL,
    model VARCHAR(100) NOT NULL,
    engine VARCHAR(100),
    transmission VARCHAR(100),
    drivetrain VARCHAR(50), -- FWD, RWD, AWD, 4WD
    color VARCHAR(50),

    -- Service Info
    current_odometer INTEGER, -- last known mileage

    -- Metadata
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,

    UNIQUE(shop_id, license_plate)
);

CREATE INDEX idx_vehicles_shop_id ON vehicles(shop_id);
CREATE INDEX idx_vehicles_customer_id ON vehicles(customer_id);
CREATE INDEX idx_vehicles_vin ON vehicles(vin);
CREATE INDEX idx_vehicles_license_plate ON vehicles(shop_id, license_plate);
```

### 2.5 Core: repair_orders

```sql
CREATE TYPE ro_status AS ENUM (
    'draft',           -- Being created
    'estimate',        -- Waiting for customer approval
    'approved',        -- Customer approved, ready for work
    'in_progress',     -- Being worked on
    'completed',       -- Work done, ready for pickup
    'closed',          -- Picked up and paid
    'declined'         -- Customer declined work
);

CREATE TYPE ro_priority AS ENUM ('low', 'normal', 'high', 'urgent');

CREATE TABLE repair_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- RO Number (human-readable, auto-increment per shop)
    ro_number VARCHAR(50) NOT NULL,

    -- Relationships
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE RESTRICT,
    assigned_technician_id UUID REFERENCES users(id) ON DELETE SET NULL,
    bay_id UUID REFERENCES bays(id) ON DELETE SET NULL,

    -- Status
    status ro_status NOT NULL DEFAULT 'draft',
    priority ro_priority DEFAULT 'normal',

    -- Vehicle Info (snapshot at time of service)
    odometer INTEGER,

    -- Customer Complaint & Notes
    customer_complaint TEXT,
    technician_notes TEXT,
    internal_notes TEXT, -- not visible to customer

    -- Dates
    check_in_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    promised_date TIMESTAMP, -- when customer expects it done
    completed_date TIMESTAMP,
    closed_date TIMESTAMP,

    -- Totals (calculated from line_items)
    labor_total DECIMAL(10,2) DEFAULT 0.00,
    parts_total DECIMAL(10,2) DEFAULT 0.00,
    subtotal DECIMAL(10,2) DEFAULT 0.00,
    tax DECIMAL(10,2) DEFAULT 0.00,
    discount DECIMAL(10,2) DEFAULT 0.00,
    total DECIMAL(10,2) DEFAULT 0.00,

    -- Metadata
    created_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,

    UNIQUE(shop_id, ro_number)
);

CREATE INDEX idx_ro_shop_id ON repair_orders(shop_id);
CREATE INDEX idx_ro_customer_id ON repair_orders(customer_id);
CREATE INDEX idx_ro_vehicle_id ON repair_orders(vehicle_id);
CREATE INDEX idx_ro_status ON repair_orders(shop_id, status) WHERE deleted_at IS NULL;
CREATE INDEX idx_ro_assigned_tech ON repair_orders(assigned_technician_id) WHERE status IN ('approved', 'in_progress');
CREATE INDEX idx_ro_dates ON repair_orders(shop_id, check_in_date DESC);
CREATE INDEX idx_ro_number ON repair_orders(shop_id, ro_number);
```

### 2.6 Core: line_items

```sql
CREATE TYPE line_item_type AS ENUM ('labor', 'part', 'fee', 'discount');
CREATE TYPE line_item_status AS ENUM ('quoted', 'approved', 'declined', 'completed');

CREATE TABLE line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    repair_order_id UUID NOT NULL REFERENCES repair_orders(id) ON DELETE CASCADE,

    -- Type
    type line_item_type NOT NULL,
    status line_item_status DEFAULT 'quoted',

    -- Item Details
    description TEXT NOT NULL,
    part_number VARCHAR(100), -- for parts only

    -- Pricing
    quantity DECIMAL(10,2) DEFAULT 1.00,
    unit_price DECIMAL(10,2) NOT NULL,
    total DECIMAL(10,2) NOT NULL, -- quantity * unit_price
    cost DECIMAL(10,2), -- shop's cost (for margin calculation)

    -- Labor Specific
    labor_hours DECIMAL(5,2), -- for labor items

    -- Inventory Link
    inventory_item_id UUID REFERENCES inventory_items(id) ON DELETE SET NULL,

    -- Tracking
    completed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    time_spent_minutes INTEGER, -- actual time technician spent

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_line_items_shop_id ON line_items(shop_id);
CREATE INDEX idx_line_items_ro_id ON line_items(repair_order_id);
CREATE INDEX idx_line_items_type ON line_items(repair_order_id, type);
CREATE INDEX idx_line_items_status ON line_items(repair_order_id, status);
CREATE INDEX idx_line_items_inventory ON line_items(inventory_item_id);
```

### 2.7 Inspections: inspections

```sql
CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    repair_order_id UUID NOT NULL REFERENCES repair_orders(id) ON DELETE CASCADE,

    -- Template Used
    template_name VARCHAR(100) DEFAULT '25-point', -- MVP uses default template

    -- Completion
    is_complete BOOLEAN DEFAULT false,
    completed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    completed_at TIMESTAMP,

    -- Summary counts (cached for quick display)
    total_points INTEGER DEFAULT 0,
    good_count INTEGER DEFAULT 0,
    attention_count INTEGER DEFAULT 0,
    urgent_count INTEGER DEFAULT 0,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inspections_shop_id ON inspections(shop_id);
CREATE INDEX idx_inspections_ro_id ON inspections(repair_order_id);
CREATE INDEX idx_inspections_completed_by ON inspections(completed_by_user_id);
```

### 2.8 Inspections: inspection_points

```sql
CREATE TYPE inspection_status AS ENUM ('good', 'attention', 'urgent');

CREATE TABLE inspection_points (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    inspection_id UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,

    -- Point Definition
    category VARCHAR(50) NOT NULL, -- tires, brakes, fluids, battery, belts, lights, etc.
    item VARCHAR(100) NOT NULL, -- 'Front Left Tire', 'Brake Fluid', etc.
    sort_order INTEGER DEFAULT 0, -- display order in UI

    -- Assessment
    status inspection_status NOT NULL DEFAULT 'good',
    notes TEXT,
    measurement VARCHAR(50), -- '3/32"', '50%', '12.6V' for specific measurements

    -- Recommendations
    recommend_service BOOLEAN DEFAULT false,
    recommended_action TEXT, -- 'Replace within 1000 miles', 'Monitor', etc.

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_inspection_points_shop_id ON inspection_points(shop_id);
CREATE INDEX idx_inspection_points_inspection_id ON inspection_points(inspection_id);
CREATE INDEX idx_inspection_points_status ON inspection_points(inspection_id, status);
CREATE INDEX idx_inspection_points_category ON inspection_points(inspection_id, category);
```

### 2.9 Media: photos

```sql
CREATE TYPE photo_context AS ENUM ('inspection_point', 'repair_order', 'vehicle');

CREATE TABLE photos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Context (what is this photo for?)
    context photo_context NOT NULL,

    -- Relationships (only one should be set based on context)
    inspection_point_id UUID REFERENCES inspection_points(id) ON DELETE CASCADE,
    repair_order_id UUID REFERENCES repair_orders(id) ON DELETE CASCADE,
    vehicle_id UUID REFERENCES vehicles(id) ON DELETE CASCADE,

    -- File Info
    url TEXT NOT NULL, -- S3 URL
    thumbnail_url TEXT, -- optimized thumbnail
    filename VARCHAR(255),
    file_size INTEGER, -- bytes
    mime_type VARCHAR(50),

    -- Image Data
    width INTEGER,
    height INTEGER,

    -- Annotations (JSONB for markup data)
    annotations JSONB DEFAULT '{}', -- {circles: [], arrows: [], text: []}

    -- AI Analysis (if available)
    ai_analysis JSONB, -- {detected: 'brake_pad', condition: 'worn', confidence: 0.92}

    -- Metadata
    uploaded_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Constraint: exactly one relationship must be set
    CONSTRAINT photo_has_one_parent CHECK (
        (inspection_point_id IS NOT NULL)::integer +
        (repair_order_id IS NOT NULL)::integer +
        (vehicle_id IS NOT NULL)::integer = 1
    )
);

CREATE INDEX idx_photos_shop_id ON photos(shop_id);
CREATE INDEX idx_photos_inspection_point ON photos(inspection_point_id);
CREATE INDEX idx_photos_ro ON photos(repair_order_id);
CREATE INDEX idx_photos_vehicle ON photos(vehicle_id);
CREATE INDEX idx_photos_context ON photos(shop_id, context);
```

### 2.10 Scheduling: bays

```sql
CREATE TYPE bay_type AS ENUM ('lift', 'alignment', 'tire', 'general');
CREATE TYPE bay_status AS ENUM ('available', 'occupied', 'maintenance', 'offline');

CREATE TABLE bays (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Bay Info
    name VARCHAR(50) NOT NULL, -- 'Bay 1', 'Alignment Rack', etc.
    type bay_type NOT NULL DEFAULT 'general',
    status bay_status DEFAULT 'available',

    -- Current Occupancy
    current_ro_id UUID REFERENCES repair_orders(id) ON DELETE SET NULL,

    -- Equipment/Capabilities
    equipment_tags TEXT[], -- ['2-post-lift', 'tire-changer', 'alignment-rack']

    -- Metadata
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(shop_id, name)
);

CREATE INDEX idx_bays_shop_id ON bays(shop_id);
CREATE INDEX idx_bays_status ON bays(shop_id, status) WHERE is_active = true;
CREATE INDEX idx_bays_current_ro ON bays(current_ro_id);
```

### 2.11 Scheduling: appointments

```sql
CREATE TYPE appointment_status AS ENUM ('scheduled', 'checked_in', 'in_progress', 'completed', 'no_show', 'cancelled');

CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Relationships
    customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE RESTRICT,
    repair_order_id UUID REFERENCES repair_orders(id) ON DELETE SET NULL, -- linked when checked in

    -- Assignment
    bay_id UUID REFERENCES bays(id) ON DELETE SET NULL,
    assigned_technician_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- Timing
    scheduled_start TIMESTAMP NOT NULL,
    scheduled_end TIMESTAMP NOT NULL,
    actual_start TIMESTAMP,
    actual_end TIMESTAMP,

    -- Status
    status appointment_status DEFAULT 'scheduled',

    -- Service Info
    services_requested TEXT[], -- ['Oil Change', 'Tire Rotation', 'Brake Inspection']
    customer_notes TEXT,

    -- Reminders
    reminder_sent_at TIMESTAMP,

    -- Metadata
    created_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

CREATE INDEX idx_appointments_shop_id ON appointments(shop_id);
CREATE INDEX idx_appointments_customer_id ON appointments(customer_id);
CREATE INDEX idx_appointments_vehicle_id ON appointments(vehicle_id);
CREATE INDEX idx_appointments_schedule ON appointments(shop_id, scheduled_start) WHERE deleted_at IS NULL;
CREATE INDEX idx_appointments_bay ON appointments(bay_id, scheduled_start);
CREATE INDEX idx_appointments_tech ON appointments(assigned_technician_id, scheduled_start);
CREATE INDEX idx_appointments_status ON appointments(shop_id, status) WHERE deleted_at IS NULL;
```

### 2.12 Inventory: inventory_items

```sql
CREATE TABLE inventory_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Part Identification
    part_number VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    category VARCHAR(100), -- oil, filters, brakes, belts, etc.
    manufacturer VARCHAR(100),

    -- Stock
    quantity_on_hand DECIMAL(10,2) DEFAULT 0.00,
    unit_of_measure VARCHAR(20) DEFAULT 'ea', -- ea, qt, gal, case
    reorder_point DECIMAL(10,2) DEFAULT 0.00,
    reorder_quantity DECIMAL(10,2) DEFAULT 0.00,

    -- Pricing
    cost DECIMAL(10,2) NOT NULL, -- what shop pays
    price DECIMAL(10,2) NOT NULL, -- what customer pays
    markup_percentage DECIMAL(5,2), -- calculated field

    -- Storage
    location VARCHAR(100), -- 'Shelf A3', 'Bin 12', etc.

    -- Core Charge (deposits for batteries, starters, etc.)
    core_charge DECIMAL(10,2) DEFAULT 0.00,

    -- Status
    is_active BOOLEAN DEFAULT true,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,

    UNIQUE(shop_id, part_number)
);

CREATE INDEX idx_inventory_shop_id ON inventory_items(shop_id);
CREATE INDEX idx_inventory_part_number ON inventory_items(shop_id, part_number);
CREATE INDEX idx_inventory_description ON inventory_items USING gin(to_tsvector('english', description));
CREATE INDEX idx_inventory_category ON inventory_items(shop_id, category);
CREATE INDEX idx_inventory_low_stock ON inventory_items(shop_id)
    WHERE is_active = true AND quantity_on_hand <= reorder_point;
```

### 2.13 Inventory: stock_transactions

```sql
CREATE TYPE transaction_type AS ENUM ('receipt', 'sale', 'adjustment', 'return');

CREATE TABLE stock_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    inventory_item_id UUID NOT NULL REFERENCES inventory_items(id) ON DELETE RESTRICT,

    -- Transaction Type
    type transaction_type NOT NULL,

    -- Quantity Change
    quantity DECIMAL(10,2) NOT NULL, -- positive for receipt/return, negative for sale
    cost DECIMAL(10,2), -- cost per unit at time of transaction

    -- Reference (why did this transaction happen?)
    repair_order_id UUID REFERENCES repair_orders(id) ON DELETE SET NULL, -- for sales
    line_item_id UUID REFERENCES line_items(id) ON DELETE SET NULL, -- specific line item
    reference_number VARCHAR(100), -- PO number, invoice number, etc.

    -- Details
    notes TEXT,

    -- Tracking
    performed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_stock_trans_shop_id ON stock_transactions(shop_id);
CREATE INDEX idx_stock_trans_item_id ON stock_transactions(inventory_item_id);
CREATE INDEX idx_stock_trans_date ON stock_transactions(shop_id, transaction_date DESC);
CREATE INDEX idx_stock_trans_ro ON stock_transactions(repair_order_id);
CREATE INDEX idx_stock_trans_type ON stock_transactions(shop_id, type);
```

### 2.14 Billing: invoices

```sql
CREATE TYPE invoice_status AS ENUM ('draft', 'sent', 'paid', 'partial', 'overdue', 'void');

CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    repair_order_id UUID NOT NULL REFERENCES repair_orders(id) ON DELETE RESTRICT,

    -- Invoice Number (human-readable)
    invoice_number VARCHAR(50) NOT NULL,

    -- Status
    status invoice_status DEFAULT 'draft',

    -- Dates
    issue_date DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date DATE,
    paid_date DATE,

    -- Amounts (mirrored from RO for historical accuracy)
    labor_total DECIMAL(10,2) DEFAULT 0.00,
    parts_total DECIMAL(10,2) DEFAULT 0.00,
    subtotal DECIMAL(10,2) DEFAULT 0.00,
    tax DECIMAL(10,2) DEFAULT 0.00,
    discount DECIMAL(10,2) DEFAULT 0.00,
    total DECIMAL(10,2) NOT NULL,

    -- Payment Tracking
    amount_paid DECIMAL(10,2) DEFAULT 0.00,
    amount_due DECIMAL(10,2) NOT NULL, -- total - amount_paid

    -- Terms
    payment_terms TEXT, -- 'Net 30', 'Due on receipt', etc.

    -- Notes
    notes TEXT,

    -- Files
    pdf_url TEXT, -- generated invoice PDF

    -- Metadata
    created_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(shop_id, invoice_number)
);

CREATE INDEX idx_invoices_shop_id ON invoices(shop_id);
CREATE INDEX idx_invoices_ro_id ON invoices(repair_order_id);
CREATE INDEX idx_invoices_number ON invoices(shop_id, invoice_number);
CREATE INDEX idx_invoices_status ON invoices(shop_id, status);
CREATE INDEX idx_invoices_dates ON invoices(shop_id, issue_date DESC);
CREATE INDEX idx_invoices_overdue ON invoices(shop_id, due_date)
    WHERE status IN ('sent', 'partial') AND due_date < CURRENT_DATE;
```

### 2.15 Billing: payments

```sql
CREATE TYPE payment_method AS ENUM ('cash', 'check', 'credit_card', 'debit_card', 'ach', 'financing', 'other');
CREATE TYPE payment_status AS ENUM ('pending', 'completed', 'failed', 'refunded');

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,

    -- Payment Info
    amount DECIMAL(10,2) NOT NULL,
    method payment_method NOT NULL,
    status payment_status DEFAULT 'completed',

    -- Payment Processor Data
    transaction_id VARCHAR(255), -- Stripe charge ID, check number, etc.
    processor_fee DECIMAL(10,2) DEFAULT 0.00, -- Stripe fee
    net_amount DECIMAL(10,2), -- amount - processor_fee

    -- Details
    check_number VARCHAR(50),
    reference_number VARCHAR(100),
    notes TEXT,

    -- Dates
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    refunded_at TIMESTAMP,

    -- Tracking
    processed_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_shop_id ON payments(shop_id);
CREATE INDEX idx_payments_invoice_id ON payments(invoice_id);
CREATE INDEX idx_payments_date ON payments(shop_id, processed_at DESC);
CREATE INDEX idx_payments_method ON payments(shop_id, method);
CREATE INDEX idx_payments_transaction_id ON payments(transaction_id);
```

### 2.16 Communication: notifications

```sql
CREATE TYPE notification_type AS ENUM ('sms', 'email', 'push');
CREATE TYPE notification_event AS ENUM (
    'appointment_reminder',
    'estimate_ready',
    'vehicle_ready',
    'invoice_sent',
    'payment_received',
    'general_message'
);
CREATE TYPE notification_status AS ENUM ('pending', 'sent', 'delivered', 'failed', 'bounced');

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Recipient
    customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
    recipient_phone VARCHAR(20),
    recipient_email VARCHAR(255),

    -- Notification Details
    type notification_type NOT NULL,
    event notification_event NOT NULL,
    status notification_status DEFAULT 'pending',

    -- Content
    subject VARCHAR(255), -- for emails
    message TEXT NOT NULL,

    -- Context
    repair_order_id UUID REFERENCES repair_orders(id) ON DELETE SET NULL,
    appointment_id UUID REFERENCES appointments(id) ON DELETE SET NULL,
    invoice_id UUID REFERENCES invoices(id) ON DELETE SET NULL,

    -- Delivery Tracking
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    error_message TEXT,

    -- External IDs (Twilio, SendGrid)
    external_id VARCHAR(255),

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_shop_id ON notifications(shop_id);
CREATE INDEX idx_notifications_customer_id ON notifications(customer_id);
CREATE INDEX idx_notifications_status ON notifications(shop_id, status, created_at DESC);
CREATE INDEX idx_notifications_type ON notifications(shop_id, type);
CREATE INDEX idx_notifications_event ON notifications(shop_id, event);
CREATE INDEX idx_notifications_ro ON notifications(repair_order_id);
```

### 2.17 AI: ai_predictions

```sql
CREATE TYPE prediction_type AS ENUM (
    'estimate_generation',
    'complaint_analysis',
    'service_recommendation',
    'predictive_maintenance',
    'part_lookup',
    'photo_diagnosis'
);

CREATE TABLE ai_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,

    -- Type
    prediction_type prediction_type NOT NULL,

    -- Context (what triggered this prediction?)
    entity_type VARCHAR(50), -- 'repair_order', 'vehicle', 'inspection_point'
    entity_id UUID,

    -- Input
    input_data JSONB NOT NULL, -- what was sent to AI

    -- Output
    prediction JSONB NOT NULL, -- AI response
    confidence DECIMAL(5,4), -- 0.0000 to 1.0000

    -- Model Info
    model_name VARCHAR(100), -- 'gpt-4', 'claude-3', 'custom-vision-v1'
    model_version VARCHAR(50),

    -- Feedback Loop
    was_accepted BOOLEAN, -- did user accept/use this prediction?
    user_feedback TEXT,
    actual_outcome JSONB, -- what actually happened (for training)

    -- Performance
    inference_time_ms INTEGER, -- how long did prediction take

    -- Tracking
    created_by_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_ai_predictions_shop_id ON ai_predictions(shop_id);
CREATE INDEX idx_ai_predictions_type ON ai_predictions(shop_id, prediction_type);
CREATE INDEX idx_ai_predictions_entity ON ai_predictions(entity_type, entity_id);
CREATE INDEX idx_ai_predictions_created_at ON ai_predictions(shop_id, created_at DESC);
CREATE INDEX idx_ai_predictions_feedback ON ai_predictions(shop_id, was_accepted)
    WHERE was_accepted IS NOT NULL;
```

---

## 3. Relationships Summary

### One-to-Many Relationships

```
shops (1) → (many) users
shops (1) → (many) customers
shops (1) → (many) vehicles
shops (1) → (many) repair_orders
shops (1) → (many) appointments
shops (1) → (many) bays
shops (1) → (many) inventory_items
shops (1) → (many) invoices

customers (1) → (many) vehicles
customers (1) → (many) repair_orders
customers (1) → (many) appointments

vehicles (1) → (many) repair_orders
vehicles (1) → (many) appointments

repair_orders (1) → (many) line_items
repair_orders (1) → (many) inspections
repair_orders (1) → (many) photos
repair_orders (1) → (many) stock_transactions
repair_orders (1) → (1) invoice

inspections (1) → (many) inspection_points

inspection_points (1) → (many) photos

inventory_items (1) → (many) stock_transactions
inventory_items (1) → (many) line_items

invoices (1) → (many) payments

users (1) → (many) repair_orders [created_by]
users (1) → (many) repair_orders [assigned_technician]
users (1) → (many) inspections [completed_by]
```

### Foreign Key Constraints

**CASCADE on shop deletion:**
- If a shop is deleted, ALL related data is deleted (full tenant cleanup)

**RESTRICT on customer/vehicle/RO deletion:**
- Cannot delete customer if they have ROs
- Cannot delete vehicle if it has ROs
- Cannot delete RO if it has an invoice
- Prevents accidental data loss

**SET NULL on user deletion:**
- If user is deleted, their assignments become NULL
- Preserves historical records even after employee leaves

---

## 4. Index Strategy

### Primary Indexes (Already Defined Above)

**Multi-Tenant Queries:**
- Every table has `shop_id` index for tenant isolation
- Composite indexes like `(shop_id, status)` for filtered queries

**Foreign Keys:**
- All foreign key columns are indexed for JOIN performance

**Search Fields:**
- Customer phone/email for quick lookup
- VIN and license plate for vehicle search
- RO number and invoice number for reference lookups

**Date Ranges:**
- `check_in_date`, `scheduled_start`, `transaction_date` for reports

**Full-Text Search:**
- Inventory description uses GIN index for text search

### Query Optimization Examples

**Find all in-progress ROs for a shop:**
```sql
-- Uses idx_ro_status (shop_id, status)
SELECT * FROM repair_orders
WHERE shop_id = ? AND status = 'in_progress' AND deleted_at IS NULL;
```

**Get customer's service history:**
```sql
-- Uses idx_ro_customer_id
SELECT ro.*, v.year, v.make, v.model
FROM repair_orders ro
JOIN vehicles v ON ro.vehicle_id = v.id
WHERE ro.customer_id = ?
ORDER BY ro.check_in_date DESC
LIMIT 10;
```

**Find low-stock items:**
```sql
-- Uses idx_inventory_low_stock (partial index)
SELECT * FROM inventory_items
WHERE shop_id = ? AND is_active = true
  AND quantity_on_hand <= reorder_point;
```

**Daily revenue report:**
```sql
-- Uses idx_ro_dates (shop_id, check_in_date)
SELECT DATE(check_in_date) as date, SUM(total) as revenue
FROM repair_orders
WHERE shop_id = ?
  AND check_in_date >= ?
  AND check_in_date < ?
  AND status IN ('completed', 'closed')
GROUP BY DATE(check_in_date)
ORDER BY date DESC;
```

---

## 5. Multi-Tenant Strategy

### Approach: Shared Database, Row-Level Isolation

**Every table has `shop_id`:**
```sql
shop_id UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE
```

### Enforcement Mechanisms

#### 1. Database-Level: Row-Level Security (RLS)

```sql
-- Enable RLS on all tenant tables
ALTER TABLE repair_orders ENABLE ROW LEVEL SECURITY;

-- Policy: users can only access their shop's data
CREATE POLICY shop_isolation_policy ON repair_orders
    USING (shop_id = current_setting('app.current_shop_id')::uuid);

-- Set context in application before queries
SET app.current_shop_id = '123e4567-e89b-12d3-a456-426614174000';
```

**Pros:**
- Database enforces tenant isolation (can't bypass in code)
- Extra security layer

**Cons:**
- Adds complexity
- Every query must set context
- Can impact performance

**Recommendation for MVP:** Start without RLS, add later for enterprise customers.

#### 2. Application-Level: Query Filters

**Every query includes `shop_id` filter:**

```typescript
// GOOD - Always filter by shop_id
const repairOrders = await db.query(
  'SELECT * FROM repair_orders WHERE shop_id = $1 AND status = $2',
  [shopId, status]
);

// BAD - Missing shop_id filter (would leak data!)
const repairOrders = await db.query(
  'SELECT * FROM repair_orders WHERE status = $1',
  [status]
);
```

**ORM Approach (Prisma example):**

```typescript
// Middleware to auto-inject shop_id
prisma.$use(async (params, next) => {
  if (params.model && params.action === 'findMany') {
    params.args.where = {
      ...params.args.where,
      shop_id: currentShopId
    };
  }
  return next(params);
});
```

#### 3. Connection Pooling

**Separate connection pools per tenant (optional):**
- For large tenants, dedicated connection pool
- Prevents one tenant from monopolizing connections
- Use PgBouncer with multiple pools

#### 4. Data Partitioning (Future)

**When scale demands:**
```sql
-- Partition repair_orders by shop_id range
CREATE TABLE repair_orders_partition_1
  PARTITION OF repair_orders
  FOR VALUES FROM ('00000000-...') TO ('50000000-...');

CREATE TABLE repair_orders_partition_2
  PARTITION OF repair_orders
  FOR VALUES FROM ('50000000-...') TO ('ffffffff-...');
```

**Benefits:**
- Query performance (only scan relevant partition)
- Easier to archive old tenant data
- Can move partitions to different storage

### Multi-Tenant Security Checklist

✅ Every table has `shop_id` foreign key
✅ All queries filter by `shop_id`
✅ ORM middleware enforces `shop_id` injection
✅ API authentication extracts `shop_id` from JWT
✅ Unit tests verify tenant isolation
✅ No global queries (admin tools must be explicit)
✅ Backup strategy allows per-tenant restore
✅ Monitoring tracks queries missing `shop_id` filter

---

## 6. Additional Constraints & Rules

### Unique Constraints

```sql
-- RO numbers are unique per shop (not globally)
UNIQUE(shop_id, ro_number)

-- Invoice numbers are unique per shop
UNIQUE(shop_id, invoice_number)

-- Part numbers are unique per shop
UNIQUE(shop_id, part_number)

-- User emails are unique per shop (same email can exist in different shops)
UNIQUE(shop_id, email)

-- VIN is globally unique (one vehicle, many shops can service it)
UNIQUE(vin)

-- License plates are unique per shop (same plate in different states)
UNIQUE(shop_id, license_plate)
```

### Check Constraints

```sql
-- Quantities must be positive for receipts
ALTER TABLE stock_transactions
ADD CONSTRAINT positive_quantity_for_receipts
CHECK (type != 'receipt' OR quantity > 0);

-- Line item total = quantity * unit_price
ALTER TABLE line_items
ADD CONSTRAINT total_equals_quantity_times_price
CHECK (total = quantity * unit_price);

-- Invoice amount_due = total - amount_paid
ALTER TABLE invoices
ADD CONSTRAINT amount_due_calculation
CHECK (amount_due = total - amount_paid);

-- Appointment end must be after start
ALTER TABLE appointments
ADD CONSTRAINT end_after_start
CHECK (scheduled_end > scheduled_start);

-- Payment amount must be positive
ALTER TABLE payments
ADD CONSTRAINT positive_payment_amount
CHECK (amount > 0);
```

### Default Values

```sql
-- Auto-generate UUIDs
DEFAULT gen_random_uuid()

-- Auto-timestamp
DEFAULT CURRENT_TIMESTAMP

-- Sensible defaults for status fields
DEFAULT 'draft'
DEFAULT 'active'
DEFAULT 'pending'

-- Zero for money fields
DEFAULT 0.00
```

### Triggers (Auto-Update)

```sql
-- Auto-update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_customers_updated_at BEFORE UPDATE ON customers
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_vehicles_updated_at BEFORE UPDATE ON vehicles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_repair_orders_updated_at BEFORE UPDATE ON repair_orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Apply to all tables with updated_at column
```

```sql
-- Auto-update RO totals when line items change
CREATE OR REPLACE FUNCTION update_ro_totals()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE repair_orders
    SET
        labor_total = (
            SELECT COALESCE(SUM(total), 0)
            FROM line_items
            WHERE repair_order_id = NEW.repair_order_id
              AND type = 'labor'
              AND status != 'declined'
        ),
        parts_total = (
            SELECT COALESCE(SUM(total), 0)
            FROM line_items
            WHERE repair_order_id = NEW.repair_order_id
              AND type = 'part'
              AND status != 'declined'
        )
    WHERE id = NEW.repair_order_id;

    -- Update subtotal and total
    UPDATE repair_orders
    SET
        subtotal = labor_total + parts_total,
        total = subtotal + tax - discount
    WHERE id = NEW.repair_order_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_ro_totals_on_line_item_change
    AFTER INSERT OR UPDATE OR DELETE ON line_items
    FOR EACH ROW EXECUTE FUNCTION update_ro_totals();
```

```sql
-- Auto-decrement inventory when line item added
CREATE OR REPLACE FUNCTION decrement_inventory()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.type = 'part' AND NEW.inventory_item_id IS NOT NULL THEN
        -- Reduce inventory
        UPDATE inventory_items
        SET quantity_on_hand = quantity_on_hand - NEW.quantity
        WHERE id = NEW.inventory_item_id;

        -- Log transaction
        INSERT INTO stock_transactions (
            shop_id, inventory_item_id, type, quantity,
            cost, repair_order_id, line_item_id,
            performed_by_user_id
        ) VALUES (
            NEW.shop_id, NEW.inventory_item_id, 'sale', -NEW.quantity,
            NEW.cost, NEW.repair_order_id, NEW.id,
            NEW.completed_by_user_id
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER decrement_inventory_on_part_usage
    AFTER INSERT ON line_items
    FOR EACH ROW EXECUTE FUNCTION decrement_inventory();
```

---

## 7. Migration Strategy

### Initial Schema Creation

```sql
-- migrations/001_initial_schema.sql

-- 1. Create ENUMs
CREATE TYPE user_role AS ENUM (...);
CREATE TYPE ro_status AS ENUM (...);
-- ... all enums

-- 2. Create core tables (no foreign keys yet)
CREATE TABLE shops (...);
CREATE TABLE users (...);
CREATE TABLE customers (...);

-- 3. Add foreign keys
ALTER TABLE users ADD CONSTRAINT fk_users_shop_id
    FOREIGN KEY (shop_id) REFERENCES shops(id) ON DELETE CASCADE;
-- ... all foreign keys

-- 4. Create indexes
CREATE INDEX idx_users_shop_id ON users(shop_id);
-- ... all indexes

-- 5. Create triggers
CREATE TRIGGER update_customers_updated_at ...
-- ... all triggers
```

### Sample Data for Testing

```sql
-- migrations/002_seed_test_data.sql (development only)

-- Create test shop
INSERT INTO shops (id, name, slug, tax_rate, labor_rate) VALUES
    ('550e8400-e29b-41d4-a716-446655440000', 'Test Auto Shop', 'test-auto-shop', 0.0825, 120.00);

-- Create test user
INSERT INTO users (shop_id, email, password_hash, first_name, last_name, role) VALUES
    ('550e8400-e29b-41d4-a716-446655440000', 'admin@testshop.com', '$2b$12$...', 'Admin', 'User', 'owner');

-- Create test customer
INSERT INTO customers (shop_id, first_name, last_name, email, phone) VALUES
    ('550e8400-e29b-41d4-a716-446655440000', 'John', 'Doe', 'john@example.com', '555-0100');
```

---

## 8. Performance Considerations

### Query Performance

**Expected Query Times (95th percentile):**
- Customer lookup by phone: <10ms
- Load RO with line items: <20ms
- Daily revenue report: <50ms
- Load inspection with photos: <30ms

### Scaling Thresholds

**When to add read replicas:**
- >1000 queries/second
- >100 concurrent shops actively using platform
- Reports impacting OLTP performance

**When to partition tables:**
- repair_orders >10M rows
- photos >50M rows
- stock_transactions >50M rows

### Caching Strategy

**Redis caching candidates:**
```
Key: shop:{shop_id}:settings
TTL: 1 hour
Value: JSONB of shop settings

Key: user:{user_id}:profile
TTL: 30 minutes
Value: User profile data

Key: vehicle:{vin}:decode
TTL: 7 days
Value: VIN decode results (rarely changes)

Key: shop:{shop_id}:revenue:today
TTL: 5 minutes
Value: Cached daily revenue total
```

### Archival Strategy

**Archive old ROs after 2 years:**
```sql
-- Move to archive table
INSERT INTO repair_orders_archive
SELECT * FROM repair_orders
WHERE shop_id = ? AND check_in_date < NOW() - INTERVAL '2 years';

-- Delete from active table
DELETE FROM repair_orders
WHERE shop_id = ? AND check_in_date < NOW() - INTERVAL '2 years';
```

---

## 9. Implementation Checklist

### Phase 1: Core Schema
- [ ] Create all ENUMs
- [ ] Create base tables (shops, users, customers, vehicles)
- [ ] Create RO tables (repair_orders, line_items)
- [ ] Add foreign key constraints
- [ ] Add basic indexes
- [ ] Test with sample data

### Phase 2: Inspections & Media
- [ ] Create inspection tables
- [ ] Create photos table
- [ ] Add S3 URL fields
- [ ] Test photo upload flow

### Phase 3: Scheduling
- [ ] Create bays table
- [ ] Create appointments table
- [ ] Add scheduling indexes
- [ ] Test drag-and-drop scenarios

### Phase 4: Inventory
- [ ] Create inventory_items table
- [ ] Create stock_transactions table
- [ ] Add inventory triggers
- [ ] Test stock decrement on sale

### Phase 5: Billing
- [ ] Create invoices table
- [ ] Create payments table
- [ ] Add payment indexes
- [ ] Test Stripe integration

### Phase 6: Communication & AI
- [ ] Create notifications table
- [ ] Create ai_predictions table
- [ ] Add notification indexes
- [ ] Test SMS/email logging

### Phase 7: Optimization
- [ ] Add all triggers (updated_at, totals calculation)
- [ ] Add check constraints
- [ ] Performance test with 10K ROs
- [ ] Optimize slow queries

### Phase 8: Production Readiness
- [ ] Set up automated backups
- [ ] Configure replication
- [ ] Enable query monitoring
- [ ] Document runbook for common operations

---

**End of Database Schema Document**

Ready for implementation with PostgreSQL 15+.
