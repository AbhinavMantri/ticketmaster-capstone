# Ticketmaster Capstone Notes

## Overview

This capstone is built as a microservices-based event ticketing platform. The current backend is split across these main services:

- `user-service`
- `event-management-service`
- `seats-allocation-service`
- `booking-service`
- `payment-service`

Supporting infrastructure in the workspace:

- `kafka` for async inventory/event messaging
- PostgreSQL-backed persistence per service

At a high level, the platform supports:

- user registration, login, profile access, and admin user management
- venue and event setup by organisers
- seat inventory creation and seat locking
- checkout and payment initiation
- booking finalization and ticket issuance
- ticket lookup and ticket scanning

## Services And Responsibilities

### user-service

Owns authentication and user identity.

- register/login/logout/refresh token flows
- profile read/update
- admin user listing, role updates, status updates
- JWT issuance for downstream protected APIs

### event-management-service

Owns venue and event management.

- create venues and sections
- generate venue seats
- create and update events
- configure pricing
- initialize inventory
- publish events for customer discovery

### seats-allocation-service

Owns event-level seat inventory and seat state transitions.

- expose seat availability
- lock seats during checkout
- confirm seats after successful payment
- release seats after failed/cancelled payment
- maintain lock details for downstream consumers

### booking-service

Owns checkout orchestration, bookings, and tickets.

- start checkout
- coordinate with seats-allocation for lock/confirm/release
- coordinate with payment-service for payment initiation
- finalize booking after successful payment
- store booking items
- issue tickets
- provide booking and ticket retrieval APIs

### payment-service

Owns payment lifecycle.

- initiate payment
- verify payment
- cancel payment
- process provider webhook/internal notifications
- notify booking-service about payment outcomes

## Main Actors

- Customer
- Organiser
- Admin
- Payment Provider

## End-To-End Platform Flows

## 1. User Authentication Flow

Handled by `user-service`.

### Main APIs

- `POST /user-service/v1/auth/register`
- `POST /user-service/v1/auth/login`
- `POST /user-service/v1/auth/refresh`
- `POST /user-service/v1/auth/logout`
- `GET /user-service/v1/users/me`
- `PATCH /user-service/v1/users/me`

### Flow

1. User registers with email, password, and profile details.
2. User logs in and receives access and refresh tokens.
3. Access token is used to call protected APIs in other services.
4. Refresh token is used to obtain a new access token when needed.
5. Logout revokes the current refresh token/session.

### Outcome

- authenticated user identity is available across the platform
- customer and organiser journeys can proceed on protected endpoints

## 2. Admin User Management Flow

Handled by `user-service`.

### Main APIs

- `GET /user-service/v1/admin/users`
- `PATCH /user-service/v1/admin/users/{id}/status`
- `PATCH /user-service/v1/admin/users/{id}/role`

### Flow

1. Admin lists users with filters/pagination.
2. Admin updates user status such as active or disabled.
3. Admin updates user role such as customer or organiser/admin.

### Outcome

- operational control over platform users

## 3. Venue Setup Flow

Handled by `event-management-service`.

### Main APIs

- `POST /venues`
- `GET /venues`
- `GET /venues/{venueId}`
- `POST /venues/{venueId}/sections`
- `POST /venues/{venueId}/sections/{sectionId}/seats/generate`

### Flow

1. Organiser creates a venue.
2. Organiser adds sections to the venue.
3. Organiser generates physical seats inside each section.

### Outcome

- venue layout exists and can be used by events

## 4. Event Creation And Configuration Flow

Handled by `event-management-service`.

### Main APIs

- `POST /organiser/events`
- `PATCH /organiser/events/{eventId}`
- `POST /organiser/events/{eventId}/pricing`
- `POST /organiser/events/{eventId}/inventory/init`
- `POST /organiser/events/{eventId}/publish`

### Flow

1. Organiser creates an event in draft state.
2. Organiser updates event details such as metadata and schedule.
3. Organiser configures pricing for event sections/seats.
4. Organiser initializes inventory for the event.
5. Organiser publishes the event.

### Outcome

- event becomes ready for sale and public discovery

## 5. Inventory Provisioning Flow

Handled jointly by `event-management-service` and `seats-allocation-service`.

### Flow

1. Event-management prepares event seat and pricing data.
2. Event/inventory data is emitted through Kafka.
3. Seats-allocation consumes inventory initialization messages.
4. Seats-allocation creates event-specific seat inventory records and context.

### Outcome

- venue seats are transformed into event seats that can be sold

## 6. Public Event Discovery Flow

Handled by `event-management-service`.

### Main APIs

- `GET /api/v1/events`
- `GET /api/v1/events/{eventId}`

### Flow

1. Customer browses published events.
2. Customer fetches details for a specific event.
3. Frontend uses event data to drive the booking journey.

### Outcome

- customer can discover what is available to book

## 7. Seat Discovery And Availability Flow

Handled by `seats-allocation-service`.

### Main APIs

- `GET /events/{eventId}/seats`
- `GET /events/{eventId}/seats/availability`

### Flow

1. Customer opens an event.
2. Frontend fetches seats and current availability.
3. Available seats are presented for selection.

### Outcome

- customer can choose valid seats before checkout

## 8. Seat Lock Flow

Handled by `seats-allocation-service`, usually triggered by `booking-service`.

### Main APIs

Public:

- `POST /events/{eventId}/locks`
- `POST /events/{eventId}/locks/release`

Internal:

- `POST /internal/seats/{eventId}/locks`
- `GET /internal/locks?bookingId=...`

### Flow

1. Customer selects seats and starts checkout.
2. Booking-service asks seats-allocation to lock those seats.
3. Seats move from available to locked for a specific lock/booking reference.

### Outcome

- seats are temporarily reserved so parallel customers cannot buy them

## 9. Checkout Flow

Handled by `booking-service`.

### Main APIs

- `POST /booking-service/v1/bookings/checkout`

### Flow

1. Customer sends checkout request with event, seat ids, and payment provider.
2. Booking-service validates the request context.
3. Booking-service locks seats through seats-allocation internal APIs.
4. Booking-service builds booking/lock identifiers.
5. Booking-service calls payment-service to create a payment.
6. Booking-service returns checkout response containing booking/payment context.

### Outcome

- frontend receives payment details needed to continue payment

## 10. Payment Initiation Flow

Handled by `payment-service`.

### Main APIs

- `POST /payments/initiate`
- `GET /payments/{paymentId}`

### Flow

1. Booking-service sends initiation request with user id, event id, lock id, amount, currency, and provider.
2. Payment-service creates an internal payment record.
3. Payment-service calls the provider integration layer.
4. Payment-service stores provider order/payment references.
5. Payment-service returns payment summary back to booking-service.

### Outcome

- payment exists both internally and at provider side
- frontend gets provider order details to complete payment

## 11. Payment Verification / Webhook Flow

Handled by `payment-service`.

### Main APIs

- `POST /payments/{paymentId}/verify`
- internal webhook/notification handling in `InternalPaymentService`

### Flow

1. Payment provider completes or fails the transaction.
2. Payment-service receives either:
   - client-side verification request, or
   - provider webhook/internal notification
3. Payment-service validates the provider signature/event.
4. Payment status is updated to success/failure/cancelled state.

### Outcome

- payment-service becomes the source of truth for payment result

## 12. Payment Outcome To Booking Flow

Handled jointly by `payment-service` and `booking-service`.

### Internal APIs

- `POST /internal/bookings/payment-outcomes`

### Flow

1. Payment-service decides the final payment outcome.
2. Payment-service sends the outcome to booking-service.
3. Booking-service branches based on payment status.

Success path:

1. Booking-service fetches lock details from seats-allocation.
2. Booking-service confirms seats.
3. Booking-service finalizes booking.
4. Booking items and tickets are created.

Failure/cancel path:

1. Booking-service releases the locked seats.
2. No final booking is created for a failed payment.

### Outcome

- booking and seat state stay consistent with payment state

## 13. Seat Confirmation Flow

Handled by `seats-allocation-service`, driven by `booking-service`.

### Internal APIs

- `POST /internal/seats/confirm`

### Flow

1. Booking-service sends event id, booking id, payment id, and seat ids.
2. Seats-allocation validates the lock and seat ownership.
3. Seats move from locked to confirmed.

### Outcome

- sold seats are no longer available for future purchases

## 14. Seat Release Flow

Handled by `seats-allocation-service`, driven by `booking-service`.

### Internal APIs

- `POST /internal/seats/{eventId}/locks/release`
- `POST /internal/seats/release`

### Flow

1. Payment fails, is cancelled, or otherwise does not complete.
2. Booking-service asks seats-allocation to release locked seats.
3. Seats move from locked back to available.

### Outcome

- abandoned or failed checkouts do not permanently block inventory

## 15. Booking Finalization Flow

Handled by `booking-service`.

### Internal APIs

- `POST /internal/bookings/finalize`

### Flow

1. Booking-service receives enough data to convert a lock into a booking.
2. It checks for existing booking by payment id or lock id to avoid duplicates.
3. It creates the booking in confirmed state.
4. It stores booking items for each purchased seat.
5. It issues tickets tied to the booking.

### Outcome

- customer now owns the seats as a confirmed booking with issued tickets

## 16. Booking Retrieval Flow

Handled by `booking-service`.

### Main APIs

- `GET /booking-service/v1/bookings`
- `GET /booking-service/v1/bookings/{bookingId}`
- `GET /booking-service/v1/bookings/{bookingId}/tickets`

### Flow

1. Customer requests booking history.
2. Booking-service returns bookings for the authenticated user.
3. Customer can fetch a single booking and its tickets.

### Outcome

- customer can view completed purchase history

## 17. Ticket Retrieval And Scan Flow

Handled by `booking-service`.

### Main APIs

- `GET /tickets/{ticketId}`
- `POST /tickets/scan`

### Flow

1. Ticket is fetched by id for display or validation.
2. Ticket scan endpoint validates usage/check-in state.
3. Ticket status is updated according to scan rules.

### Outcome

- issued tickets are usable for event entry flow

## Core Customer Purchase Sequence

This is the most important end-to-end business flow.

1. Customer logs in through `user-service`.
2. Customer browses published events through `event-management-service`.
3. Customer checks seat availability in `seats-allocation-service`.
4. Customer starts checkout in `booking-service`.
5. Booking-service locks seats in `seats-allocation-service`.
6. Booking-service initiates payment in `payment-service`.
7. Frontend completes provider payment.
8. Payment-service verifies or receives webhook outcome.
9. Payment-service notifies booking-service about payment result.
10. Booking-service confirms or releases seats using `seats-allocation-service`.
11. Booking-service finalizes the booking if payment succeeded.
12. Booking-service creates tickets.
13. Customer views booking and tickets.

## Core Organiser Setup Sequence

1. Organiser logs in through `user-service`.
2. Organiser creates a venue.
3. Organiser creates sections and generates seats.
4. Organiser creates an event.
5. Organiser configures pricing.
6. Organiser initializes inventory.
7. Organiser publishes the event.
8. Customers can now discover and book the event.

## State Transitions

### Seat State

- available
- locked
- confirmed
- released back to available

### Payment State

- created
- pending/provider-created
- success/captured
- failed
- cancelled

### Booking State

- not yet finalized during checkout
- confirmed after successful payment

### Ticket State

- issued
- scanned/used depending on scan flow

## Important Cross-Service Contracts

### Booking -> Payment

Booking sends:

- user id
- event id
- lock id
- amount
- currency
- provider
- idempotency key

### Payment -> Booking

Payment sends:

- payment id
- user id
- event id
- lock id
- total amount
- currency
- payment status

### Booking -> Seats Allocation

Booking sends:

- event id
- booking id / lock id
- user id
- payment id when applicable
- seat ids
- idempotency/reference keys

## Reliability / Safety Patterns In The Design

- idempotency on checkout/payment initiation paths
- lock-before-pay to reduce double booking
- confirm seats only after successful payment
- release seats on failed/cancelled payment
- internal booking finalization checks for duplicate booking creation
- ticket creation only after booking confirmation

## Current Practical Definition Of Done

For the backend, the main capstone scope appears to include:

- auth and user management
- organiser event setup
- seat inventory initialization
- public event browsing
- seat selection and lock
- checkout and payment initiation
- payment result handling
- booking confirmation
- ticket issuance and retrieval

## Suggested Next Documentation Additions

If this needs to evolve further, the next useful docs would be:

- environment setup and required env vars per service
- local run order for all services
- Kafka topics and payload contracts
- database schema summary by service
- API examples for the main happy path
- sequence diagrams for checkout and payment outcome handling
