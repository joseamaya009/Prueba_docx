
## Functional Domain Analysis — `modelo_postgresql.sql`

### Domain 1: Geography and Reference Data

**Tables:** `time_zone`, `continent`, `country`, `state_province`, `city`, `district`, `address`, `currency`

This is the foundation of the entire model. It forms a strict geographical hierarchy: `continent → country → state_province → city → district → address`. The `address` table is a critical convergence point, used by both `airport` and `maintenance_provider`. `time_zone` is associated with `city`, enabling local time handling in flight operations. `currency` is referenced by billing, payments, fares, and loyalty programs.

---

### Domain 2: Airline

**Tables:** `airline`

Although it is a single table, it is the **central axis** of the model. Virtually all operational domains depend on it: `aircraft`, `flight`, `customer`, `loyalty_program`, `fare`. It stores IATA/ICAO codes with exact-length validations enforced through `CHECK` constraints.

---

### Domain 3: Identity

**Tables:** `person_type`, `document_type`, `contact_type`, `person`, `person_document`, `person_contact`

Manages the real identity of any individual in the system, regardless of their role (customer, employee, passenger). `person` is a polymorphic base entity: a single record can act as a customer (`customer`), a system user (`user_account`), or a passenger (`reservation_passenger`). Documents and contacts are normalized into separate tables to allow multiple records per person.

---

### Domain 4: Security

**Tables:** `user_status`, `security_role`, `security_permission`, `user_account`, `user_role`, `role_permission`

Implements a full RBAC (Role-Based Access Control) scheme. `user_account` links to `person` in a 1:1 relationship. Roles are assigned to users via `user_role`, and permissions to roles via `role_permission`. There is traceability of who assigned each role (`assigned_by_user_id`). This domain is also cross-referenced in operations: `check_in` and `boarding_validation` both record the user who performed the action.

---

### Domain 5: Customers and Loyalty

**Tables:** `customer_category`, `benefit_type`, `loyalty_program`, `loyalty_tier`, `customer`, `loyalty_account`, `loyalty_account_tier`, `miles_transaction`, `customer_benefit`

`customer` links `person` to `airline` (a customer is airline-specific). The loyalty program has tiers (`loyalty_tier`) and the tier history is stored in `loyalty_account_tier` as a separate table, avoiding transitive dependencies. Miles transactions (`miles_transaction`) are immutable by design (types EARN/REDEEM/ADJUST, delta is never zero). Benefits are granted at the individual customer level.

---

### Domain 6: Airport

**Tables:** `airport`, `terminal`, `boarding_gate`, `runway`, `airport_regulation`

Physical hierarchy: `airport → terminal → boarding_gate`. Runways are independent of terminals. `airport_regulation` allows recording active regulations per airport with effective date ranges. `airport` depends on `address` from the geographic domain, giving it a concrete location. `boarding_gate` is referenced in `boarding_validation`, closing the operational cycle.

---

### Domain 7: Aircraft

**Tables:** `aircraft_manufacturer`, `aircraft_model`, `cabin_class`, `aircraft`, `aircraft_cabin`, `aircraft_seat`, `maintenance_provider`, `maintenance_type`, `maintenance_event`

Models the fleet along two axes: the physical structure of the aircraft and its maintenance history. The cabin hierarchy is `aircraft → aircraft_cabin → aircraft_seat`, where each seat has boolean properties (window, aisle, exit row). `maintenance_event` records the technical history of each aircraft. `aircraft_seat` is directly referenced in `seat_assignment` to enforce seat uniqueness per flight segment.

---

### Domain 8: Flight Operations

**Tables:** `flight_status`, `delay_reason_type`, `flight`, `flight_segment`, `flight_delay`

`flight` represents a flight instance (airline + number + date). `flight_segment` models each individual leg with origin/destination airports and scheduled vs. actual times, supporting multi-stop itineraries. Delays are recorded in `flight_delay` linked to the segment with a typed reason. `CHECK` constraints ensure origin ≠ destination and that arrival is always after departure.

---

### Domain 9: Sales and Reservations

**Tables:** `reservation_status`, `sale_channel`, `fare_class`, `fare`, `ticket_status`, `reservation`, `reservation_passenger`, `sale`, `ticket`, `ticket_segment`, `seat_assignment`, `baggage`

This is the most complex domain. The flow is: `reservation → sale → ticket → ticket_segment`. A reservation can have multiple passengers (`reservation_passenger`). Each passenger gets a ticket, and each ticket is broken down into flight segments (`ticket_segment`), which serves as the bridge table between the ticket and `flight_segment`. `seat_assignment` uses a composite FK on `(ticket_segment_id, flight_segment_id)` to enforce consistency. Baggage is recorded per ticket segment.

---

### Domain 10: Boarding

**Tables:** `boarding_group`, `check_in_status`, `check_in`, `boarding_pass`, `boarding_validation`

Sequential flow: `check_in → boarding_pass → boarding_validation`. A ticket segment can have at most one check-in (uniqueness constraint), one check-in produces at most one boarding pass, and each boarding pass can have multiple validation records (attempts). `boarding_validation` records the result (APPROVED/REJECTED/MANUAL_REVIEW), the gate, and the validating user.

---

### Domain 11: Payments

**Tables:** `payment_status`, `payment_method`, `payment`, `payment_transaction`, `refund`

`payment` is linked to `sale`, not directly to the ticket. `payment_transaction` records each individual movement (AUTH, CAPTURE, VOID, REFUND, REVERSAL), providing full traceability of the payment cycle. Refunds (`refund`) are recorded against the original payment with both request and processing timestamps.

---

### Domain 12: Billing

**Tables:** `tax`, `exchange_rate`, `invoice_status`, `invoice`, `invoice_line`

`invoice` is generated from a `sale`. Invoice lines (`invoice_line`) do not persist derived totals, preserving 3NF as noted in the script's technical comments. `tax` has date-based validity periods. `exchange_rate` handles currency conversions by specific date, supporting multi-currency operations.

---

### Most Relevant Cross-Domain Relationships

* `person` is the shared identity entity across `user_account`, `customer`, and `reservation_passenger`.
* `airline` connects fleet, flights, fares, customers, and loyalty programs.
* `address` is shared across geography, airports, and maintenance providers.
* `flight_segment` is the operational core linking airports, aircraft, tickets, seats, baggage, check-in, and delays.
* `sale` is the financial bridge between the reservation, payments, and the invoice.
