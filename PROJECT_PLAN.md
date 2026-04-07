# Yash Pharma - Complete Development Plan

## 1. Project Overview

**Project Name:** Yash Pharma  
**Type:** Single-Vendor Medical/Pharma Web Application  
**Core Features:** Online medicine ordering, prescription management, doctor consultation, door delivery  
**Target Users:** Patients seeking medicines, pharmacy owner, delivery agents, doctors

---

## 2. Tech Stack Selection

### Frontend
- **Framework:** Next.js 14 (App Router) - SEO friendly, fast, server-side rendering
- **UI Library:** Tailwind CSS + shadcn/ui components
- **State Management:** Zustand (lightweight) or React Query for server state
- **Forms:** React Hook Form + Zod validation
- **Icons:** Lucide React

### Backend
- **Framework:** Node.js with Express.js (or Next.js API routes for MVP)
- **Language:** TypeScript (type safety)

### Database
- **Primary:** PostgreSQL (structured data, better for orders/transactions)
- **ORM:** Prisma (type-safe, great DX)
- **File Storage:** Cloudinary or AWS S3 (for prescriptions/images)

### Authentication
- **Service:** Clerk or NextAuth.js (supports OTP, role-based access)
- **Alternative:** Custom JWT with OTP verification

### Payments
- **Gateway:** Razorpay (India-focused, UPI support)

### Deployment
- **Vercel** (Frontend + Serverless functions)
- **Supabase** or **Neon** (PostgreSQL)

---

## 3. Database Schema

### Users
```
users
├── id (UUID, PK)
├── email (unique)
├── phone (unique)
├── password_hash (nullable for OTP-only)
├── role (ENUM: customer, admin, doctor, delivery_agent)
├── name
├── created_at
└── updated_at

addresses
├── id (UUID, PK)
├── user_id (FK)
├── label (home/work)
├── street, city, state, pincode
├── is_default
└── created_at
```

### Products (Medicines)
```
categories
├── id (UUID, PK)
├── name
├── slug
├── description
└── image_url

products
├── id (UUID, PK)
├── category_id (FK)
├── name
├── slug
├── composition
├── uses (TEXT)
├── side_effects (TEXT)
├── dosage_info
├── requires_prescription (BOOLEAN)
├── price (DECIMAL)
├── discount_percent
├── stock_quantity
├── image_url
├── is_active
└── created_at

product_reviews
├── id (UUID, PK)
├── product_id (FK)
├── user_id (FK)
├── rating (1-5)
├── comment
└── created_at
```

### Prescriptions
```
prescriptions
├── id (UUID, PK)
├── user_id (FK)
├── order_id (FK, nullable)
├── file_url
├── status (ENUM: pending, approved, rejected)
├── notes (admin feedback)
├── uploaded_at
└── reviewed_at
```

### Orders
```
orders
├── id (UUID, PK)
├── user_id (FK)
├── address_id (FK)
├── prescription_id (FK, nullable)
├── status (ENUM: pending, approved, packed, shipped, out_for_delivery, delivered, cancelled)
├── subtotal
├── discount
├── delivery_fee
├── total
├── payment_method (ENUM: upi, card, cod)
├── payment_status (ENUM: pending, paid, failed, refunded)
├── razorpay_order_id
├── delivery_agent_id (FK, nullable)
├── estimated_delivery
├── delivered_at
├── created_at
└── updated_at

order_items
├── id (UUID, PK)
├── order_id (FK)
├── product_id (FK)
├── quantity
├── unit_price
└── total_price

delivery_agent_assignments
├── id (UUID, PK)
├── order_id (FK)
├── agent_id (FK)
├── assigned_at
├── pickup_time
└── delivery_notes
```

### Doctor Consultation
```
doctors
├── id (UUID, PK)
├── user_id (FK)
├── specialization
├── experience_years
├── consultation_fee
├── rating
├── bio
├── available_from (TIME)
├── available_to (TIME)
└── is_available

appointments
├── id (UUID, PK)
├── doctor_id (FK)
├── user_id (FK)
├── scheduled_at (DATETIME)
├── status (ENUM: scheduled, in_progress, completed, cancelled)
├── type (ENUM: chat, video, phone)
├── notes
├── prescription (TEXT)
├── fee
└── created_at

chat_messages
├── id (UUID, PK)
├── appointment_id (FK)
├── sender_id (FK)
├── message (TEXT)
├── sent_at
└── read_at
```

### Coupons
```
coupons
├── id (UUID, PK)
├── code (unique)
├── discount_type (ENUM: percentage, fixed)
├── discount_value
├── min_order_value
├── max_discount
├── valid_from
├── valid_until
├── usage_limit
├── used_count
└── is_active
```

### Notifications
```
notifications
├── id (UUID, PK)
├── user_id (FK)
├── type (ENUM: order_update, prescription, reminder, promo)
├── title
├── message
├── is_read
├── data (JSON - for deep links)
└── created_at
```

---

## 4. API Architecture

### Authentication
```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/send-otp
POST   /api/auth/verify-otp
POST   /api/auth/forgot-password
PUT    /api/auth/reset-password
GET    /api/auth/me
POST   /api/auth/logout
```

### Products
```
GET    /api/products                    (list with filters)
GET    /api/products/:slug              (single product)
GET    /api/products/featured
GET    /api/products/search?q=
GET    /api/categories
GET    /api/categories/:slug/products
POST   /api/products (admin)
PUT    /api/products/:id (admin)
DELETE /api/products/:id (admin)
```

### Cart
```
GET    /api/cart
POST   /api/cart/items
PUT    /api/cart/items/:productId
DELETE /api/cart/items/:productId
POST   /api/cart/apply-coupon
DELETE /api/cart/coupon
```

### Orders
```
GET    /api/orders                      (user's orders)
GET    /api/orders/:id
POST   /api/orders
PUT    /api/orders/:id/cancel
POST   /api/orders/:id/verify-prescription (admin)
PUT    /api/orders/:id/status (admin)
```

### Prescriptions
```
POST   /api/prescriptions/upload
GET    /api/prescriptions
GET    /api/prescriptions/:id
PUT    /api/prescriptions/:id/status (admin)
```

### Payments
```
POST   /api/payments/create-order (Razorpay)
POST   /api/payments/verify
POST   /api/payments/webhook
```

### Doctor Consultation
```
GET    /api/doctors
GET    /api/doctors/:id
GET    /api/doctors/:id/available-slots
POST   /api/appointments
GET    /api/appointments
GET    /api/appointments/:id
PUT    /api/appointments/:id/cancel
POST   /api/appointments/:id/chat
GET    /api/appointments/:id/messages
POST   /api/appointments/:id/prescription
```

### Admin
```
GET    /api/admin/dashboard/stats
GET    /api/admin/orders
GET    /api/admin/products
GET    /api/admin/users
GET    /api/admin/inventory-alerts
GET    /api/admin/reports/sales
POST   /api/admin/doctors
PUT    /api/admin/doctors/:id
```

### Delivery Agent
```
GET    /api/delivery/assigned-orders
PUT    /api/delivery/orders/:id/pickup
PUT    /api/delivery/orders/:id/delivered
PUT    /api/delivery/orders/:id/failed
GET    /api/delivery/today-summary
```

### User
```
GET    /api/users/profile
PUT    /api/users/profile
GET    /api/users/addresses
POST   /api/users/addresses
PUT    /api/users/addresses/:id
DELETE /api/users/addresses/:id
POST   /api/users/reviews
```

---

## 5. Frontend Structure (Next.js App Router)

```
src/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   ├── register/
│   │   ├── forgot-password/
│   │   └── layout.tsx
│   ├── (shop)/
│   │   ├── page.tsx                 (Home)
│   │   ├── products/
│   │   │   ├── page.tsx             (Listing)
│   │   │   └── [slug]/page.tsx      (Detail)
│   │   ├── categories/
│   │   │   └── [slug]/page.tsx
│   │   ├── cart/
│   │   ├── checkout/
│   │   ├── orders/
│   │   │   ├── page.tsx             (List)
│   │   │   └── [id]/page.tsx        (Tracking)
│   │   └── prescriptions/
│   ├── (consultation)/
│   │   ├── doctors/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   └── appointments/
│   │       ├── page.tsx
│   │       └── [id]/
│   │           ├── page.tsx
│   │           └── chat/page.tsx
│   ├── (admin)/
│   │   ├── dashboard/
│   │   ├── products/
│   │   ├── orders/
│   │   ├── prescriptions/
│   │   ├── doctors/
│   │   ├── customers/
│   │   ├── reports/
│   │   └── settings/
│   ├── (delivery)/
│   │   └── dashboard/
│   ├── api/                         (API routes)
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ui/                          (shadcn components)
│   ├── layout/
│   │   ├── header.tsx
│   │   ├── footer.tsx
│   │   └── mobile-nav.tsx
│   ├── products/
│   │   ├── product-card.tsx
│   │   ├── product-grid.tsx
│   │   ├── product-filters.tsx
│   │   └── product-search.tsx
│   ├── cart/
│   │   ├── cart-drawer.tsx
│   │   ├── cart-item.tsx
│   │   └── cart-summary.tsx
│   ├── orders/
│   │   ├── order-card.tsx
│   │   └── order-timeline.tsx
│   ├── prescriptions/
│   │   ├── upload-modal.tsx
│   │   └── prescription-card.tsx
│   ├── consultation/
│   │   ├── doctor-card.tsx
│   │   ├── appointment-card.tsx
│   │   └── chat-interface.tsx
│   └── admin/
│       ├── sidebar.tsx
│       ├── stats-card.tsx
│       └── data-table.tsx
├── lib/
│   ├── db.ts                        (Prisma client)
│   ├── auth.ts                     (Auth config)
│   ├── razorpay.ts
│   ├── validations/                (Zod schemas)
│   └── utils.ts
├── hooks/
│   ├── use-cart.ts
│   ├── use-auth.ts
│   └── use-orders.ts
├── stores/
│   └── cart-store.ts
└── types/
    └── index.ts
```

---

## 6. Development Phases

### Phase 1: Foundation (Week 1-2)
- [ ] Project setup (Next.js, TypeScript, Tailwind)
- [ ] Database schema & Prisma setup
- [ ] Authentication (Clerk or NextAuth)
- [ ] Basic layout components (Header, Footer)
- [ ] Database seed with sample data

### Phase 2: Customer Shop (Week 3-4)
- [ ] Product listing & search
- [ ] Product detail page
- [ ] Category pages
- [ ] Cart functionality
- [ ] Wishlist (optional)

### Phase 3: Checkout & Orders (Week 5-6)
- [ ] Address management
- [ ] Checkout flow
- [ ] Order placement
- [ ] Order confirmation
- [ ] Order history & tracking

### Phase 4: Prescription System (Week 7)
- [ ] Prescription upload (image/PDF)
- [ ] Admin prescription review
- [ ] Prescription linking to orders

### Phase 5: Payments (Week 8)
- [ ] Razorpay integration
- [ ] Payment verification
- [ ] COD option
- [ ] Refund flow

### Phase 6: Doctor Consultation (Week 9-10)
- [ ] Doctor profiles & listing
- [ ] Appointment booking
- [ ] Chat interface
- [ ] E-prescription generation

### Phase 7: Admin Panel (Week 11-12)
- [ ] Dashboard with analytics
- [ ] Product management (CRUD)
- [ ] Order management
- [ ] Inventory alerts
- [ ] User management
- [ ] Reports & exports

### Phase 8: Delivery Agent (Week 13)
- [ ] Agent dashboard
- [ ] Order assignment
- [ ] Status updates
- [ ] Delivery proof

### Phase 9: Polish & Deploy (Week 14)
- [ ] Notifications (Email/SMS)
- [ ] Error handling
- [ ] Loading states
- [ ] SEO optimization
- [ ] Performance optimization
- [ ] Deployment

---

## 7. UI/UX Design System

### Color Palette
```
Primary:        #2563EB (Blue - Trust)
Secondary:      #059669 (Green - Health)
Accent:         #F59E0B (Amber - Attention)
Background:     #FFFFFF (White)
Surface:        #F8FAFC (Slate-50)
Text Primary:   #0F172A (Slate-900)
Text Secondary: #64748B (Slate-500)
Error:          #DC2626 (Red-600)
Success:        #16A34A (Green-600)
```

### Typography
```
Font: Inter (Google Fonts)
Headings: 600-700 weight
Body: 400-500 weight
Scale: 14px base, modular scale 1.25
```

### Components Style
- Border radius: 8px (cards), 6px (buttons), 4px (inputs)
- Shadows: Subtle (sm for cards), Medium (for modals)
- Spacing: 4px base unit (4, 8, 12, 16, 24, 32, 48, 64)

---

## 8. MVP Scope (Launch in 4-6 weeks)

### Must Have
1. User registration/login (OTP)
2. Medicine catalog with search
3. Cart & wishlist
4. Checkout with address
5. Order placement
6. Admin: Product & order management
7. Prescription upload
8. Order tracking

### Can Wait
1. Doctor consultation
2. Payment gateway (COD first)
3. Delivery agent app
4. Reviews & ratings
5. Push notifications

---

## 9. Cost Estimation

### Development
- Self-development: ₹0
- Hire developer: ₹2-5 lakhs

### Infrastructure (Monthly)
- Hosting (Vercel): ₹0-2000
- Database (Neon): ₹0-1500
- Domain: ₹500-1000
- SMS (MSG91/Twilio): ₹500-2000
- Email (Resend/SendGrid): ₹0-500
- Storage (Cloudinary): ₹0-500

### One-time
- SSL: Free (Let's Encrypt/Vercel)
- Logo & Branding: ₹2000-5000

---

## 10. Next Steps

1. **Confirm Plan** - Review and finalize scope
2. **Design Mockups** - Figma/pen and paper
3. **Setup Project** - Initialize Next.js with all dependencies
4. **Database** - Create schema and seed data
5. **Start Building** - Phase 1 implementation

Ready to start? Let me know which phase you'd like to begin with!
