# 🎬 Cinema Management System — Software Requirements Specification

A complete Software Requirements Specification (SRS) for a Cinema Management System, covering online ticket booking, offline box office operations, F&B pre-ordering, loyalty program, identity verification (eKYC), AI-powered support, and back-office management — built using the F-Soft SRS template as part of a university Business Analyst coursework project.

## 📋 Overview

This repository contains the full requirements documentation for a cinema ticketing platform that supports **6 main actors** (Guest, Customer, Cashier, Concession Staff, Cinema Manager, System Admin) and **6 secondary actors** (Payment Gateway, Email/SMS Notification Service, Movie Content Provider, Google OAuth Service, AI Support, External Identity Verification Service), spanning **41 use cases** across **12 functional groups**.

The system covers:
- Online seat selection & booking with real-time seat locking and a configurable seat type catalog (incl. paired/Sweetbox seats)
- Offline box office (walk-in) ticket sales
- F&B pre-ordering and concession fulfillment
- QR code check-in with age/document verification denial handling and full refund override
- Loyalty points & voucher/promotion system (3-tier discount stacking, birthday leap-year edge case)
- eKYC identity verification (CCCD/CMND + facial matching)
- AI-powered customer support (movie recommendations, FAQ)
- Movie, showtime, room, seat type, and pricing management (5-step formula incl. VAT breakdown)
- Review moderation, business reporting, and immutable audit logging
- User account & role administration

## 📁 Document Structure

| Section | Content |
|---|---|
| 1. Introduction | Scope, purpose, definitions |
| 2.1 | Object Relationship Diagram (ORD) |
| 2.2 | Workflow Diagrams — Online Booking, Cancellation/Refund, and others |
| 2.3 | State Transition Diagrams — Booking, Seat, and Refund lifecycle |
| 2.4 | Use Case Diagrams (4 views: System Overview, Customer, Operations, Management) |
| 2.5 | Permission Matrix |
| 2.6 | Site Map (Customer-facing site, POS, Concession Display, Manager Dashboard, Admin Panel) |
| 3.x | 41 detailed Use Case Specifications + Common Business Rules (CBR-01 → CBR-17) |
| 4 | UI Mockups |
| 5 | Non-Functional Requirements (Performance, Availability, Security) |
| 6–9 | Common field controls, integrations, data migration, appendices |

## 🎭 Actors

**Main:** Guest · Customer · Cashier (Box Office Staff) · Concession Staff · Cinema Manager · System Admin

**Secondary:** Payment Gateway · Email/SMS Notification Service · Movie Content Provider (TMDB API) · Google OAuth Service · AI Support · External Identity Verification Service (eKYC)

## 📐 Diagrams Included

- `Context Diagram.drawio` — data flows between system and all actors/external systems
- `Use Case Diagram CMS.mdj` — use case relationships (4 views)
- `State Machine Diagram CMS.mdj` / `Seat State (State Machine Diagram).drawio` — booking, seat, and refund lifecycle
- `Workflow1.drawio` – `Workflow5.drawio` — key business process workflows (online booking, cancellation/refund, check-in/concession, etc.)
- `class-diagram-spec.md` — full data model (26 entity classes) backing the use case specifications
- Site Map (5 zones: Customer, POS, Concession, Manager Dashboard, Admin Panel)

## 🔧 Tools Used

- Microsoft Word (F-Soft SRS template)
- draw.io / StarUML — UML diagrams, workflows, state machines, site maps
- AI-assisted mockup generation for UI screens (Lovable)

## 👥 Team

University Business Analyst coursework project — Group [tên nhóm/môn học]
Course: [tên môn] · Semester [X]

## 📌 Status

🟡 v0.7.2 — post Internal Review 2. Core SRS, diagrams, and mockups complete; minor consistency fixes and final NFR/field-spec polish in progress ahead of formal Review 3.

---

## 📝 Changelog — v0.7.2 (vs. v0.7.1, post Internal Review 2)

### ✨ New Use Cases
- **UC-38: Chat with AI Support** — AI-powered chatbot for recommendations and FAQ, with strict data-privacy rule (only anonymized behavioral context shared, never PII)
- **UC-39: Automated Showtime Reminder** — system-initiated scheduled job (SMS reminder ~1h before showtime), exactly-once delivery guarantee
- **UC-40: Identity Verification** — eKYC flow (CCCD front/back + selfie → external verification service → facial match → `verifiedDob`)
- **UC-41: Manage Seat Types Catalog** — Cinema Manager can create/edit/deactivate seat types (no longer hardcoded to Standard/VIP/Sweetbox), with cascading updates to pricing and room layout

### 🔧 New / Updated Business Rules
- **CBR-17 (new): Audit Logging** — immutable, system-wide audit trail for all data-modifying, financial, and security-sensitive events
- **CBR-05 updated** — added *Check-in Denial Refund*: age-restriction or unverifiable-ID check-in failures now trigger a 100% refund, overriding the standard time-tiered policy
- **CBR-10 expanded** — added VAT breakdown (Step 5: Ticket 5%, F&B 8%, surcharge 5%, service fee 10%/manager-set), birthday discount redemption mechanics (incl. Feb 29 leap-year exception), and online-vs-offline student discount verification rules
- **CBR-11 updated** — voucher-cap edge case: if a voucher is reduced to 0% by the 39% stacking cap, the code is **not** marked as used and remains valid for future bookings

### 🐛 Bugs Found & Fixed
- **UC-22 (Cancel Booking)** — corrected an incorrect `<<extend>>` relationship: cancellation now correctly extends UC-15 (View Booking History), not UC-09, reflecting where the Cancel action is actually triggered from
- **UC-23 (Process Refund)** — resolved a Pre-condition/Main-Flow contradiction (refund record was described as both "already existing" and "newly created"); refund record creation is now clearly automatic at the UC-22→UC-23 trigger boundary
- **Permission Matrix** — corrected Cashier permission for UC-12 (Apply Voucher); clarified Cashier's indirect access to UC-11 (F&B) via `<<extend>>` from UC-16

### 🔒 New Restriction (with documented rationale)
- **UC-25 (Redeem Loyalty Points)** — confirmed restricted to the online channel only; offline/counter redemption is explicitly disallowed to prevent counter-side fraud (cash collected could be reduced without a customer-initiated digital confirmation trail)

### 🪑 Data Model Enhancements
- Added **`SeatType`** entity (replaces hardcoded enum), with `priceMultiplier`, `colorCode`, `seatsPerUnit` for paired seats (e.g. Sweetbox)
- Added **`pairedSeatId`** self-reference on `RoomSeat` to support 2-seat units sold/locked as a single atomic block
- Added **`HolidayCalendar`** entity (location-scoped or system-wide) feeding into CBR-10 pricing
- Added **`CinemaLocation`** and **`PriceConfig`** entities to support multi-room, location-aware configuration
- Added **`AuditLog`** entity (append-only, no mutating methods)

### 📚 Other
- Field-length and dropdown-source specifications added for key fields (Section 6.1)
- Out-of-scope clarifications documented: no drag-and-drop seat layout UI; no multi-cinema branch comparison reporting (treated as a separate Corporate BI concern outside this SRS's scope)

## 📄 License

This is an academic project for educational purposes only.
