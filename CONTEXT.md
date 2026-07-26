# Event Management Platform

A full-stack web application that allows Event Organizers to create and promote events, while Customers can browse, purchase tickets, and attend those events.

## Language

### Roles & Identity

**Customer**:
A registered user who browses events, purchases tickets, and attends events.
_Avoid_: Attendee (until they've actually attended), buyer, user (too vague)

**Organizer**:
A registered user who creates events, manages transactions, and creates promotions.
_Avoid_: Admin, event creator, host

**Role**:
One of exactly two values: `CUSTOMER` or `ORGANIZER`. Assigned at registration and cannot be changed.
_Avoid_: User type, account type

### Events

**Event**:
A scheduled gathering created by an Organizer with a name, category, location, date range, and ticket capacity.
_Avoid_: Activity, gathering, show

**Category**:
A pre-seeded classification for events (e.g., Music, Technology, Sports). Cannot be created or modified by users.
_Avoid_: Tag, genre, type

**TicketType**:
An optional named tier within an Event (e.g., "VIP", "Regular", "Early Bird"), each with its own price and quota. When absent, the Event uses a single flat price.
_Avoid_: Ticket class, ticket tier, pricing tier

### Transactions & Payments

**Transaction**:
A record of a Customer purchasing tickets for an Event. Has exactly six possible statuses.
_Avoid_: Order, purchase, booking (we use "booking code" as an output, but the record itself is a Transaction)

**TransactionStatus**:
One of exactly six values: `WAITING_FOR_PAYMENT`, `WAITING_FOR_ADMIN_CONFIRMATION`, `DONE`, `REJECTED`, `EXPIRED`, `CANCELED`.
_Avoid_: Order status, payment status

**BookingCode**:
A unique alphanumeric code (format: `EVT-YYYY-XXXXXX`) generated when a Transaction reaches `DONE` status. Sent via email and used for attendance verification.
_Avoid_: Ticket code, confirmation code, order number

**PaymentProof**:
An image uploaded by the Customer as evidence of payment. Must be uploaded within 2 hours of Transaction creation.
_Avoid_: Receipt, payment screenshot

### Promotions & Rewards

**Voucher**:
A discount code created by an Organizer, applicable to specific events they organize. Can be percentage-based or fixed amount. Has a usage limit and validity period.
_Avoid_: Promo code, discount code (when referring to the entity)

**Coupon**:
A system-generated 10% discount reward given to a Customer who registers using a referral code. Single-use, expires 3 months after creation.
_Avoid_: Reward coupon, referral discount

**PointLedger**:
An immutable record of point credits and debits for a Customer. Each credit has its own expiry date (3 months from creation). Debits follow FIFO order (oldest-expiring points spent first).
_Avoid_: Points balance, point history, wallet

**ReferralCode**:
A unique alphanumeric code (format: `REF-XXXXXX`) auto-generated for every new user at registration. Cannot be changed. When used by another user during registration, the referrer earns 10,000 points and the referee gets a Coupon.
_Avoid_: Invite code, referral link

### Reviews

**Review**:
A rating (1–5 stars) and optional text comment left by a Customer for an Event they attended. Requires Transaction status `DONE` and `isAttended = true`.
_Avoid_: Feedback, testimonial, comment
