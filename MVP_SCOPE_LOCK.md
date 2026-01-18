# MVP SCOPE LOCK
## AI-First Auto Shop Management Platform

**Target Launch:** 6 months from start
**Success Criteria:** 10 paying pilot shops using the platform daily

---

## IN SCOPE: MVP Features

### Authentication & User Management
- Email/password authentication (bcrypt hashed)
- User roles: Owner, Service Advisor, Technician, Front Desk
- Role-based permissions (view/create/edit ROs based on role)
- Password reset via email
- Simple user invite system

### Customer & Vehicle Database
- Create/read/update customer records (name, email, phone, address)
- Search customers by phone, email, or name (fuzzy matching)
- Create/read/update vehicle records (VIN, year, make, model, odometer)
- VIN decoder integration (auto-populate year/make/model/engine from VIN)
- View vehicle service history (list of past ROs)
- Basic duplicate detection (warn if phone/email already exists)

### Repair Order Lifecycle
- Create repair order (customer, vehicle, complaint, promised date)
- RO statuses: Draft → Estimate → Approved → In Progress → Completed → Closed
- Add line items (labor or parts) with description, quantity, unit price
- Manual line item entry (no AI assistance required for MVP)
- Assign RO to technician
- Update RO status (technicians can update via mobile)
- View all ROs (filterable by status, date, customer)
- Auto-calculate totals (line items + tax)
- Delete ROs (Draft status only)

### Digital Multi-Point Inspection (Mobile)
- Mobile-responsive inspection form
- Inspection template: 25 pre-defined check points (tires, brakes, fluids, belts, battery, lights, wipers)
- For each point: Good / Attention Needed / Urgent status
- Add notes to each inspection point
- Upload photos (max 3 per inspection point)
- Photo annotation (draw/markup on photos)
- Attach inspection to RO
- View completed inspections in web app

### Estimates & Customer Communication
- Generate estimate PDF from RO line items
- Send estimate to customer via SMS (link to view estimate)
- Customer views estimate on mobile-friendly page (read-only)
- SMS notifications:
  - Appointment reminder (1 day before)
  - "Your vehicle is ready for pickup"
  - Estimate ready for review
- Manual SMS send (service advisor can text customer)

### Invoicing & Payments
- Generate invoice PDF from completed RO
- Invoice includes: shop logo, customer info, vehicle info, line items, tax, total
- Email invoice to customer (PDF attachment)
- Record payment (cash, check, credit card)
- Stripe integration for credit card payments (online and in-person via terminal)
- Mark invoice as Paid/Partial/Unpaid
- View payment history per customer

### Scheduling (Basic)
- Calendar view (daily and weekly)
- Create appointment (customer, vehicle, date/time, estimated duration, services)
- Drag-and-drop to reschedule appointments
- Assign appointment to bay and technician
- View tech workload (list of assigned ROs)
- Appointment statuses: Scheduled → Checked In → Completed → No Show

### Inventory (Basic)
- Add inventory item (part number, description, quantity, cost, price)
- Search parts by part number or description
- Deduct from inventory when added to RO
- Manual stock adjustments (add/remove with reason)
- Low stock alert (below reorder point)
- View inventory list (sortable, filterable)

### Basic Reporting
- Date range selector (today, week, month, custom)
- Revenue report (total sales by date)
- RO count by status
- Top services (by revenue and count)
- Technician productivity (ROs completed, hours billed)
- Export reports to CSV

### AI Features (MVP Only)
- VIN decode (pull year/make/model/engine from VIN using NHTSA API)
- Smart customer search (fuzzy matching on name/phone/email)
- Complaint parsing: Customer enters complaint → AI suggests relevant inspection points and services
- AI estimate generation: Describe work needed → AI returns suggested line items with labor hours and parts
- Predictive maintenance: When vehicle checks in, show overdue services based on mileage and service history

### Technical Requirements
- Web app: responsive design (works on desktop, tablet, phone)
- Mobile inspection: works offline, syncs when online
- Page load time: <1 second for all pages
- Database: PostgreSQL (multi-tenant with shop_id on all tables)
- File storage: S3 for photos and PDFs
- Email: SendGrid for transactional emails
- SMS: Twilio for text messages
- Payment: Stripe for credit card processing
- Hosting: AWS (ECS or similar)
- Uptime: 99% (allows 7 hours downtime/month during MVP)

---

## OUT OF SCOPE: Explicit Exclusions

### Not in MVP (Deferred to Phase 2)
- ❌ Customer portal (self-service login)
- ❌ Two-way SMS messaging (only one-way notifications in MVP)
- ❌ Photo-based AI diagnostics (tire tread depth, brake pad wear measurement)
- ❌ Natural language reporting ("How much did we make last week?")
- ❌ Smart scheduling assistant (AI optimization of bay assignments)
- ❌ Parts ordering integration (NAPA, AutoZone APIs)
- ❌ Accounting integration (QuickBooks, Xero)
- ❌ Email marketing campaigns
- ❌ Review management (Google, Yelp)
- ❌ Advanced inventory (auto-reordering, purchase orders)
- ❌ Multi-location support
- ❌ Custom report builder
- ❌ Third-party API (no public API in MVP)
- ❌ Native mobile apps (iOS/Android)
- ❌ Real-time updates (WebSockets/live refresh)
- ❌ Advanced search (Elasticsearch)
- ❌ Single Sign-On (SSO)
- ❌ 2-factor authentication (2FA)
- ❌ White-labeling
- ❌ Customer sentiment analysis
- ❌ Churn prediction
- ❌ Dynamic pricing recommendations
- ❌ Commission calculation
- ❌ Payroll integration
- ❌ Time clock (clock in/out)
- ❌ Canned service packages/templates (will add in Phase 2)
- ❌ Fleet management
- ❌ Warranty tracking
- ❌ Shop performance benchmarking

### Intentionally Simple in MVP
- Manual pricing entry (no dynamic pricing)
- Basic tax calculation (single tax rate per shop, no jurisdiction lookup)
- No customizable inspection templates (use default 25-point inspection)
- No custom fields on customers/vehicles/ROs
- No bulk operations (edit one RO at a time)
- No advanced permissions (4 roles only, no custom roles)
- No audit logs (who changed what when)
- No data import tools (manual entry for pilot shops)
- No mobile apps (mobile-responsive web only)

---

## DEFINITION OF DONE: MVP Acceptance Criteria

The MVP is complete when ALL of the following are true:

### Functional Criteria
1. ✅ A service advisor can check in a walk-in customer, create an RO, and generate an estimate in <5 minutes
2. ✅ A technician can complete a 25-point inspection with photos on a mobile phone in <10 minutes
3. ✅ VIN decoder correctly populates vehicle data for 95% of VINs tested (2010-2025 vehicles)
4. ✅ AI estimate generation produces line items with labor/parts for common complaints (oil change, brakes, alignment, etc.) with 80%+ accuracy
5. ✅ Customer receives SMS notification when estimate is ready and when vehicle is ready for pickup
6. ✅ Customer can view estimate on mobile device and see all inspection photos
7. ✅ A customer can pay invoice via Stripe (credit card) and receive email receipt
8. ✅ Scheduling calendar allows drag-and-drop of appointments between days and techs
9. ✅ Inventory is automatically decremented when parts are added to an RO and saved
10. ✅ Revenue report shows daily sales totals for the current month
11. ✅ All pages load in <1 second on 4G mobile connection
12. ✅ Mobile inspection works offline and syncs photos when back online
13. ✅ 4 user roles (Owner, Service Advisor, Technician, Front Desk) have correct permissions enforced
14. ✅ Invoice PDFs include shop logo, formatted line items, and correct tax calculation

### Pilot Success Criteria
15. ✅ 10 pilot shops onboarded and actively using the platform
16. ✅ Each pilot shop creates at least 20 ROs in the first 30 days
17. ✅ 80%+ of pilot shops rate onboarding experience as "Good" or "Excellent"
18. ✅ Zero data loss incidents during pilot period
19. ✅ <5 critical bugs reported per pilot shop in first 30 days
20. ✅ Uptime >99% during pilot period (tracked via monitoring)

### Technical Criteria
21. ✅ Production deployment on AWS with automated CI/CD pipeline
22. ✅ Database backups running daily (automated, tested restore)
23. ✅ Monitoring and alerting configured (uptime, errors, performance)
24. ✅ All API endpoints have input validation and error handling
25. ✅ Security: TLS encryption, bcrypt password hashing, CSRF protection, rate limiting

---

## PHASE 2: Post-MVP Roadmap

**Timing:** Months 7-12 after MVP launch

### High-Priority Phase 2 Features
- Photo-based AI diagnostics (measure tire tread depth, brake pad wear from photos)
- Customer portal (view service history, approve estimates, pay invoices online)
- Two-way SMS messaging (conversations with customers)
- Parts ordering integration (NAPA API for live pricing and ordering)
- Canned service packages (pre-built packages: "60K service", "brake job front", etc.)
- Custom inspection templates (shops can create their own checklists)
- Purchase orders and receiving workflow
- Accounting integration (QuickBooks sync for invoices and payments)
- Email campaigns (service reminders, promotions)
- Advanced reporting (custom date ranges, more metrics, drill-downs)

### Medium-Priority Phase 2 Features
- Time clock (employee clock in/out)
- Commission calculation (flat rate vs. hourly tracking)
- Review requests (automated Google/Yelp review requests after RO completion)
- Multi-location support (one account, multiple shop locations)
- Customer tags and segments (VIP, fleet, etc.)
- Custom fields (add shop-specific fields to customers/vehicles)
- Data import tools (CSV import for migration from other systems)
- Audit logs (track who changed what)
- 2FA (two-factor authentication)

### Low-Priority Phase 2 Features
- Native mobile apps (iOS/Android with offline-first architecture)
- Natural language reporting ("Show me brake jobs last month")
- Smart scheduling assistant (AI suggests optimal bay/tech assignments)
- Public API (for third-party integrations)
- White-label options (custom branding for chains)
- SSO (SAML for enterprise customers)

---

## SCOPE CHANGE PROCESS

Any feature not listed in "IN SCOPE" above requires:
1. Written approval from Product Owner
2. Impact analysis (timeline, resources)
3. Explicit removal of equal-sized feature from MVP
4. Update to this document (version controlled)

**No exceptions. No "quick adds." No "while we're at it."**

---

**Version:** 1.0
**Last Updated:** 2026-01-18
**Approved By:** [CTO, VP Engineering, VP Product]

**Pilot Shop Onboarding Starts:** Month 5
**Target MVP Launch:** Month 6
