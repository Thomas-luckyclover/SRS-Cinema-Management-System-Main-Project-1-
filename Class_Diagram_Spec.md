# CLASS DIAGRAM SPECIFICATION

_Input đầy đủ để vẽ Class Diagram — fields + methods + relationships_
_Traced từ SRS + field refactor đã chốt_

---

## PHẦN 1 — DANH SÁCH CLASSES + FIELDS + METHODS

---

### 1. User _(Base class — inheritance)_

```
User
─────────────────────────────────────────
- userId: String
- fullName: String
- email: String
- phone: String
- passwordHash: String
- role: ENUM(GUEST, CUSTOMER, CASHIER, CONCESSION_STAFF, CINEMA_MANAGER, SYSTEM_ADMIN)
- status: ENUM(ACTIVE, INACTIVE)
- failedLoginCount: int
- lockedUntil: Timestamp [nullable]
- createdAt: Timestamp
─────────────────────────────────────────
+ login(email, password): Token
+ logout(): void
+ resetPassword(otp, newPassword): void
```

---

### 2. Customer _(extends User)_

```
Customer
─────────────────────────────────
- dateOfBirth: Date [nullable]           ← do NGƯỜI DÙNG nhập (chưa verified)
- avatarUrl: String [nullable]
- birthdayDiscountUsedCount: int         ← reset mỗi năm
- birthdayDiscountUsedYear: int [nullable]  ← năm đã dùng để reset
- cccdNumber: String [nullable, UNIQUE]
- verifiedDob: Date [nullable]           ← DOB từ CCCD — authoritative cho birthday discount
- identityVerificationStatus: ENUM(NOT_VERIFIED, PENDING, VERIFIED, REJECTED)
- faceMatchScore: Decimal [nullable]
- faceMatchThreshold: Decimal = 0.85     ← minimum score to pass
- cccdVerifiedAt: Timestamp [nullable]
- cccdFrontImageUrl: String [nullable]   ← lưu để audit, không show UI
- cccdBackImageUrl: String [nullable]   ← lưu để audit, không show UI
- selfieImageUrl: String [nullable]      ← lưu để audit
─────────────────────────────────
+ submitIdentityVerification(cccdFront, selfie): VerificationResult
+ isBirthdayToday(): boolean             ← dùng verifiedDob, không dùng dateOfBirth
+ canUseBirthdayDiscountToday(): boolean
+ getMaxBirthdayDiscountThisYear(): int  ← trả về 1 hoặc 2 (Feb 29 rule)
```

---

### 3. Staff _(extends User — abstract)_

```
Staff
─────────────────────────────────────────
- assignedLocationIds: List<String>  [FK → CinemaLocation.locationId]
─────────────────────────────────────────
```

### 3a. Cashier _(extends Staff)_

```
Cashier
─────────────────────────────────────────
(không có field riêng)
─────────────────────────────────────────
+ sellTicketAtCounter(showtimeId, seats, payment): Booking
+ scanQRCheckIn(qrCode): CheckInResult
+ processRefund(bookingId, method): Refund
+ cancelBooking(bookingId): void
```

### 3b. ConcessionStaff _(extends Staff)_

```
ConcessionStaff
─────────────────────────────────────────
(không có field riêng)
─────────────────────────────────────────
+ viewPendingOrders(): List<FnBOrderItem>
+ fulfillOrder(orderId): void
```

### 3c. CinemaManager _(extends Staff)_

```
CinemaManager
─────────────────────────────────────────
(không có field riêng)
─────────────────────────────────────────
+ manageMovie(movie): Movie
+ manageShowtime(showtime): Showtime
+ manageRoom(room): ScreeningRoom
+ managePricing(config): void
+ moderateReview(reviewId, decision): void
+ viewDashboard(): DashboardData
```

### 3d. SystemAdmin _(extends Staff)_

```
SystemAdmin
─────────────────────────────────────────
(không có field riêng)
─────────────────────────────────────────
+ manageUserAccounts(): List<User>
+ configureSystemSettings(settings): void
+ viewReports(): ReportData
```

---

### 4. Movie

```
Movie
─────────────────────────────────────────
- movieId: String
- title: String
- description: String
- posterUrl: String
- trailerUrl: String [nullable]
- genre: String
- ageRating: ENUM(K, T13, T16, T18)
- durationMinutes: int
- status: ENUM(COMING_SOON, NOW_SHOWING, ENDED, HIDDEN)
- tmdbId: String [nullable]
- directorName: String
- castList: String
- createdBy: String [FK → User.userId]
─────────────────────────────────────────
+ publish(): void
+ hide(): void
+ importFromTMDB(tmdbId): Movie
```

---

### 5. ScreeningRoom

```
ScreeningRoom
─────────────────────────────────────────
- roomId: String
- roomName: String
- roomType: ENUM(2D, 3D, IMAX, 4DX, DOLBY_ATMOS)
- totalCapacity: int
- status: ENUM(ACTIVE, MAINTENANCE)
- surchargeAmount: Decimal [nullable]
- locationId: String [FK → CinemaLocation.locationId]
─────────────────────────────────────────
+ setMaintenance(): void
+ setActive(): void
+ getSeatLayout(): List<RoomSeat>
```

---

### 6. RoomSeat _(Template — per room)_

```
RoomSeat
─────────────────────────────────────────
- seatId: String
- roomId: String [FK → ScreeningRoom.roomId]
- rowLabel: String
- columnNumber: int
- seatTypeId: String [FK → SeatType.seatTypeId]   ← thay ENUM cũ
- pairedSeatId: String [FK → RoomSeat.seatId, nullable]   ← chỉ có giá trị khi
- isPhysicallyBlocked: boolean
─────────────────────────────────────────
+ markBlocked(): void
+ markActive(): void
+ isPartOfPair(): boolean
```

---

### 7. Showtime

```
Showtime
─────────────────────────────────────────
- showtimeId: String
- movieId: String [FK → Movie.movieId]
- roomId: String [FK → ScreeningRoom.roomId]
- startTime: DateTime
- endTime: DateTime  ← calculated: startTime + duration + 15min buffer
- languageFormat: ENUM(SUBTITLED, DUBBED, ORIGINAL)
- status: ENUM(SCHEDULED, PUBLISHED, SOLD_OUT, CLOSED, CANCELLED)
─────────────────────────────────────────
+ publish(): void
+ generateSeatMap(): List<ShowtimeSeat>   ← UC28 Step 6, Hướng A
+ hasOverlap(room, startTime, endTime): boolean
+ isFrozen(): boolean   ← CBR-09
```

---

### 8. ShowtimeSeat _(Dynamic — per showtime)_

```
ShowtimeSeat
─────────────────────────────────────────
- showtimeSeatId: String
- showtimeId: String [FK → Showtime.showtimeId]
- roomSeatId: String [FK → RoomSeat.seatId]
- status: ENUM(AVAILABLE, LOCKED, SOLD, BLOCKED)
- lockedBySessionId: String [FK → BookingSession.sessionId, nullable]
- soldBookingId: String [FK → Booking.bookingId, nullable]
─────────────────────────────────────────
+ lock(sessionId): void
+ release(): void
+ sell(bookingId): void
+ block(): void
+ unblock(): void
```

---

### 9. BookingSession _(Cache — Redis, không phải DB chính)_

```
BookingSession
─────────────────────────────────────────
- sessionId: String
- customerId: String [FK → Customer.userId]
- showtimeId: String [FK → Showtime.showtimeId]
- selectedSeatIds: List<String>
- timerStartedAt: Timestamp
- lockExpiresAt: Timestamp   ← timerStartedAt + 10 phút (CBR-01)
- fnbItems: List<FnBOrderItem>
- voucherCode: String [nullable]
- pointsToRedeem: int
- totalAmount: Decimal
─────────────────────────────────────────
+ addSeat(seatId): void
+ removeSeat(seatId): void
+ applyVoucher(code): void
+ calculateTotal(): Decimal
+ convertToBooking(): Booking   ← điểm chuyển đổi sang Booking
+ expire(): void
```

---

### 10. Booking

```
Booking
─────────────────────────────────────────
- bookingId: String
- pendingPaymentExpiresAt
- guardianConfirmedOnline
- guardianVerifiedAtCheckIn
- customerId: String [FK → Customer.userId, nullable]
- source: ENUM(ONLINE, OFFLINE)
- status: ENUM(PENDING_PAYMENT, CONFIRMED, FAILED, EXPIRED, CANCELLED, CHECKED_IN)
- cancellationReason: ENUM(PERSONAL_CONFLICT, WRONG_SHOWTIME, DUPLICATE_BOOKING, HEALTH_ISSUE, LATE_ARRIVAL, TECHNICAL_ISSUE, OTHER) [nullable]
- cancellationReasonText: String [nullable]   ← required khi reason = OTHER
- showtimeId: String [FK → Showtime.showtimeId]
- paymentMethod: ENUM(VNPAY, MOMO, VISA, MASTERCARD, CASH, CARD_POS, QR_TRANSFER)
- totalAmount: Decimal
- voucherCode: String [nullable]
- discountAmount: Decimal
- loyaltyPointsEarned: int
- loyaltyPointsRedeemed: int [nullable]
- ageVerified: boolean [nullable]
- studentDiscountClaimed: boolean
 verifiedBy: String [FK → Cashier.userId, nullable]  ← age check at UC-19; student discount verified by Cashier at UC-16
- createdBy: String [FK → Cashier.userId, nullable]
- createdAt: Timestamp
- checkedInAt: Timestamp [nullable]
- cancelledAt: Timestamp [nullable]
- pendingPaymentStartedAt: Timestamp [nullable]
- reminderSent: boolean
- reminderSentAt: Timestamp [nullable]
- emailRetryCount
─────────────────────────────────────────
+ confirm(): void
+ expire(): void
+ cancel(): void
+ checkIn(qrCode): void
+ calculateRefund(): Decimal
```

---

### 11. Refund

```
Refund
─────────────────────────────────────────
- refundId: String
- bookingId: String [FK → Booking.bookingId]
- amount: Decimal
- status: ENUM(NO_REFUND_DUE, PROCESSING, MANUAL_FOLLOWUP, REFUNDED)
- method: ENUM(GATEWAY_REVERSAL, CASH, BANK_TRANSFER) [nullable]
- processedBy: String [FK → Cashier.userId, nullable]
- requestedAt: Timestamp
- completedAt: Timestamp [nullable]
─────────────────────────────────────────
+ process(): void
+ markManualFollowup(): void
+ complete(cashierId): void
```

---

### 12. FnBMenuItem _(Catalog)_

```
FnBMenuItem
─────────────────────────────────────────
- menuItemId: String
- name: String
- category: String
- description: String
- price: Decimal
- imageUrl: String [nullable]
- status: ENUM(AVAILABLE, UNAVAILABLE)
─────────────────────────────────────────
+ setAvailable(): void
+ setUnavailable(): void
```

---

### 13. FnBOrderItem _(Per-booking order)_

```
FnBOrderItem
─────────────────────────────────────────
- orderItemId: String
- bookingId: String [FK → Booking.bookingId]
- menuItemId: String [FK → FnBMenuItem.menuItemId]
- quantity: int
- unitPrice: Decimal   ← locked at order time
- fulfillmentStatus: ENUM(PENDING, FULFILLED)
- netPriceExVat: Decimal
- vatRate: Decimal    ← 8% cho F&B
- vatAmount: Decimal
─────────────────────────────────────────
+ fulfill(): void
```

---

### 14. LoyaltyAccount

```
LoyaltyAccount
─────────────────────────────────────────
- accountId: String
- customerId: String [FK → Customer.userId]
- balance: int
- tier: ENUM(STANDARD, SILVER, GOLD)
- tierReviewedAt: Timestamp
- lastActivityAt: Timestamp   ← BR-UC24-01 expiry check
─────────────────────────────────────────
+ earn(points, bookingId): void
+ redeem(points, bookingId): void
+ reverse(points, bookingId): void   ← CBR-13
+ reinstate(points, bookingId): void   ← CBR-13 partial cancel
+ checkExpiry(): void   ← BR-UC24-01
```

---

### 15. LoyaltyTransaction

```
LoyaltyTransaction
─────────────────────────────────────────
- transactionId: String
- accountId: String [FK → LoyaltyAccount.accountId]
- bookingId: String [FK → Booking.bookingId, nullable]
- type: ENUM(EARNED, REDEEMED, REVERSED, REINSTATED, EXPIRED)
- points: int
- description: String
- createdAt: Timestamp
─────────────────────────────────────────
(chỉ read — không có mutating methods, append-only log)
```

---

### 16. Voucher

```
Voucher
─────────────────────────────────────────
- voucherId: String
- code: String [UNIQUE]
- discountType: ENUM(PERCENTAGE, FIXED)
- discountValue: Decimal
- validFrom: DateTime
- validTo: DateTime
- maxUsage: int
- usageCount: int
- minTier: ENUM(STANDARD, SILVER, GOLD) [nullable]
- applicableMovieIds: List<String> [nullable]
- applicableSlots: List<String> [nullable]
- applicableDays: List<int> [nullable]
- applicableSeatTypes: List<String> [nullable]
- createdBy: String [FK → CinemaManager.userId]
─────────────────────────────────────────
+ validate(booking): ValidationResult   ← CBR-12
+ apply(booking): Decimal
+ incrementUsage(): void
```

---

### 17. Campaign _(Tier 2 Promotion)_

```
Campaign
─────────────────────────────────────────
- campaignId: String
- name: String
- discountPercentage: Decimal
- allowsVoucherStack: boolean   ← CBR-11
- createdBy: String [FK → CinemaManager.userId]
- validFrom: Date
- validTo: Date
- maxRedemptionsTotal: int [nullable]
- maxRedemptionsPerPhone: int [nullable]   ← chống voucher abuse
- applicableLocationIds: List<String> [nullable]   ← BP-2 tỉnh lẻ
─────────────────────────────────────────
+ isActiveToday(): boolean
+ apply(booking): Decimal
```

---

### 18. MovieReview

```
MovieReview
─────────────────────────────────────────
- reviewId: String
- movieId: String [FK → Movie.movieId]
- customerId: String [FK → Customer.userId]
- bookingId: String [FK → Booking.bookingId]   ← chỉ CHECKED_IN mới review được
- rating: int   ← 1-5
- content: String
- status: ENUM(PENDING, APPROVED, REJECTED)
- moderatedBy: String [FK → CinemaManager.userId, nullable]
- createdAt: Timestamp
- moderatedAt
- rejectionReason
─────────────────────────────────────────
+ approve(managerId): void
+ reject(managerId): void
```

---

### 19. CinemaLocation

```
CinemaLocation
─────────────────────────────────
- locationId: String
- name: String
- address: String
- city: String
- province: String
- status: ENUM(ACTIVE, INACTIVE)
─────────────────────────────────
+ getRooms(): List<ScreeningRoom>
```

### 20. BookingSeat

```
BookingSeat
─────────────────────────────────
- bookingSeatId: String
- bookingId: String [FK → Booking]
- showtimeSeatId: String [FK → ShowtimeSeat]
- seatTypeId: String [FK → SeatType.seatTypeId]   ← thay ENUM cũ
- basePrice: Decimal
- seatMultiplier: Decimal
- roomSurcharge: Decimal
- discountApplied: Decimal
- netPriceExVat: Decimal      ← giá chưa VAT
- vatRate: Decimal            ← 5%
- vatAmount: Decimal
- finalPrice: Decimal         ← inclusive VAT
- cancelledAt: Timestamp [nullable]  ← partial cancellation support
─────────────────────────────────
+ cancel(): void   ← partial cancel CBR-05
```

### 21. AuditLog

```
AuditLog
─────────────────────────────────
- logId: String
- timestamp: Timestamp
- actorId: String [FK → User.userId]
- actorRole: String
- action: ENUM(CREATE, UPDATE, DELETE, LOGIN, LOGOUT, LOGIN_FAILED)
- entityType: String       ← "Booking", "User", "Showtime"...
- entityId: String
- detail: String           ← "Service fee 5000 → 6000"
- ipAddress: String
─────────────────────────────────
(append-only — không có mutating methods)
```

### 22. SystemSetting

```
SystemSetting
─────────────────────────────────
- settingKey: String [PK]
- settingValue: String
- dataType: ENUM(INT, DECIMAL, BOOLEAN, STRING)
- description: String
- updatedBy: String [FK → SystemAdmin.userId]
- updatedAt: Timestamp
─────────────────────────────────
(append-only log riêng, không edit-in-place)
```

### 23. IdentityVerificationService <<external>>

```

IdentityVerificationService
─────────────────────────────────
(eKYC provider — VNPay eKYC / ViettelPay ID / FPT.AI)
─────────────────────────────────
+ verifyCCCD(cccdImage): CCCDData
+ matchFace(cccdPhoto, selfie): FaceMatchResult

```

### 24. PriceConfig

```
PriceConfig
─────────────────────────────────────────
- configId: String
- locationId: String [FK → CinemaLocation.locationId]
- timeSlot: ENUM(MORNING, AFTERNOON, EVENING)
- basePrice: Decimal
- seatMultiplierStandard: Decimal = 1.0
- seatMultiplierVip: Decimal = 1.5
- seatMultiplierSweetbox: Decimal = 2.0
- roomSurchargeImax: Decimal [nullable]
- roomSurcharge4dx: Decimal [nullable]
- roomSurchargeDolby: Decimal [nullable]
- effectiveFrom: Date
- createdBy: String [FK → CinemaManager.userId]
- createdAt: Timestamp
─────────────────────────────────────────
+ getPrice(showtime: Showtime, seatType: ENUM): Decimal   ← implements CBR-10 Step 1-3
+ isEffectiveToday(): boolean
```

---

### 25. HolidayCalendar

```
HolidayCalendar
─────────────────────────────────────────
- holidayId: String
- date: Date
- name: String
- locationId: String [FK → CinemaLocation.locationId, nullable]   ← null = áp dụng toàn hệ thống
- createdBy: String [FK → SystemAdmin.userId]
- createdAt: Timestamp
- updatedBy: String [FK → SystemAdmin.userId, nullable]
- updatedAt: Timestamp [nullable]
─────────────────────────────────────────
+ isHoliday(date: Date, locationId: String): boolean   ← CBR-10 Step 1
+ getHolidaysByLocation(locationId: String): List<HolidayCalendar>
+ isNationalHoliday(date: Date): boolean   ← locationId = null
```

---

### 26. SeatType

```
RoomSeat
─────────────────────────────────────────
- seatId: String
- roomId: String [FK → ScreeningRoom.roomId]
- rowLabel: String
- columnNumber: int
- seatTypeId: String [FK → SeatType.seatTypeId]   ← thay ENUM cũ
- pairedSeatId: String [FK → RoomSeat.seatId, nullable]   ← chỉ có giá trị khi SeatType.seatsPerUnit > 1
- isPhysicallyBlocked: boolean
─────────────────────────────────────────
+ markBlocked(): void
+ markActive(): void
+ isPartOfPair(): boolean
```

---

## PHẦN 2 — RELATIONSHIPS

### Inheritance (Generalization)

```
User <|-- Customer
User <|-- Staff
Staff <|-- Cashier
Staff <|-- ConcessionStaff
Staff <|-- CinemaManager
Staff <|-- SystemAdmin
```

---

### Composition (vòng đời phụ thuộc hoàn toàn vào cha)

```
ScreeningRoom "1" *-- "1..*" RoomSeat
  → Room bị xóa thì RoomSeat cũng mất theo

Showtime "1" *-- "1..*" ShowtimeSeat
  → Showtime bị xóa thì ShowtimeSeat cũng mất theo

Booking "1" *-- "0..*" FnBOrderItem
  → Booking bị xóa thì order item cũng mất theo

Booking "1" *-- "0..1" Refund
  → Refund gắn chặt với Booking

Booking "1" *-- "1..*" BookingSeat
  → Booking bị xóa thì BookingSeat cũng mất theo



```

---

### Aggregation (tồn tại độc lập)

```
Customer "1" o-- "1" LoyaltyAccount
  → Customer xóa, LoyaltyAccount vẫn giữ lại để audit

LoyaltyAccount "1" o-- "0..*" LoyaltyTransaction
```

---

### Association (tham chiếu, không phụ thuộc)

```
Movie "1" --> "0..*" Showtime
  Showtime.movieId → Movie

ScreeningRoom "1" --> "0..*" Showtime
  Showtime.roomId → ScreeningRoom

RoomSeat "1" --> "0..*" ShowtimeSeat
  ShowtimeSeat.roomSeatId → RoomSeat (template snapshot)

Customer "1" --> "0..*" Booking
  Booking.customerId → Customer [nullable]

Showtime "1" --> "0..*" Booking
  Booking.showtimeId → Showtime

ShowtimeSeat "0..*" --> "0..1" Booking
  ShowtimeSeat.soldBookingId → Booking

Booking "0..*" --> "0..1" Voucher
  Booking.voucherCode → Voucher

FnBOrderItem "0..*" --> "1" FnBMenuItem
  FnBOrderItem.menuItemId → FnBMenuItem

Customer "1" --> "0..*" MovieReview
  MovieReview.customerId → Customer

Movie "1" --> "0..*" MovieReview
  MovieReview.movieId → Movie

Booking "1" --> "0..1" MovieReview
  MovieReview.bookingId → Booking

BookingSeat "0..*" --> "1" ShowtimeSeat
  BookingSeat.showtimeSeatId → ShowtimeSeat

  CinemaLocation "1" --> "0..*" ScreeningRoom
  ScreeningRoom.locationId → CinemaLocation

Staff "0..*" --> "1..*" CinemaLocation
  Staff.assignedLocationIds → CinemaLocation (many-to-many)
```

---

### Dependency (dùng tạm, không lưu FK)

```
BookingSession ..> Showtime
  Session tham chiếu Showtime trong cache

BookingSession ..> Customer
  Session tham chiếu Customer trong cache

BookingSession ..> ShowtimeSeat
  Session lock ShowtimeSeat tạm thời

BookingSession ..> Voucher
  Session validate Voucher tạm thời

BookingSession --> Booking
  Session.convertToBooking() tạo ra Booking (1 chiều, 1 lần)

Customer ..> IdentityVerificationService
```

---

## PHẦN 3 — MULTIPLICITY SUMMARY

| Relationship                        | Type        | Multiplicity |
| ----------------------------------- | ----------- | ------------ |
| ScreeningRoom → RoomSeat            | Composition | 1 → 1..\*    |
| Showtime → ShowtimeSeat             | Composition | 1 → 1..\*    |
| Booking → FnBOrderItem              | Composition | 1 → 0..\*    |
| Booking → Refund                    | Composition | 1 → 0..1     |
| Customer → LoyaltyAccount           | Aggregation | 1 → 1        |
| LoyaltyAccount → LoyaltyTransaction | Aggregation | 1 → 0..\*    |
| Movie → Showtime                    | Association | 1 → 0..\*    |
| ScreeningRoom → Showtime            | Association | 1 → 0..\*    |
| RoomSeat → ShowtimeSeat             | Association | 1 → 0..\*    |
| Customer → Booking                  | Association | 1 → 0..\*    |
| Showtime → Booking                  | Association | 1 → 0..\*    |
| ShowtimeSeat → Booking              | Association | 0..\* → 0..1 |
| Booking → Voucher                   | Association | 0..\* → 0..1 |
| FnBOrderItem → FnBMenuItem          | Association | 0..\* → 1    |
| Movie → MovieReview                 | Association | 1 → 0..\*    |
| Customer → MovieReview              | Association | 1 → 0..\*    |
| Booking → MovieReview               | Association | 1 → 0..1     |
| BookingSession → Booking            | Dependency  | 1 → 0..1     |
| Booking → BookingSeat               | Composition | 1 → 1..\*    |
| BookingSeat → ShowtimeSeat          | Association | 0..\* → 1    |
| CinemaLocation → ScreeningRoom      | Association | 1 → 0..\*    |
| Staff → CinemaLocation              | Association | 0.._ → 1.._  |
| SystemAdmin → SystemSetting         | Association | 0..\* → 1    |
| User → AuditLog                     | Association | 0..\* → 1    |

SeatType "1" --> "0..\*" RoomSeat
RoomSeat.seatTypeId → SeatType
(Association — RoomSeat tham chiếu SeatType, không sở hữu vòng đời nhau)

RoomSeat "0..1" --> "0..1" RoomSeat
RoomSeat.pairedSeatId → RoomSeat (self-reference, reflexive association)
Chỉ có giá trị khi seatTypeId.seatsPerUnit > 1; mỗi cặp là 2 chiều
(A.pairedSeatId = B.seatId VÀ B.pairedSeatId = A.seatId)

CinemaManager "1" --> "0..\*" SeatType
SeatType.createdBy → CinemaManager.userId

---

## PHẦN 4 — LƯU Ý KHI VẼ

1. **BookingSession** vẽ bằng class thường nhưng thêm note `<<cache>>` hoặc `<<transient>>` để phân biệt với các entity DB chính
2. **User hierarchy** vẽ tam giác kế thừa (không phải mũi tên thường) — UML generalization
3. **Composition** vẽ hình thoi đặc (◆), **Aggregation** vẽ hình thoi rỗng (◇)
4. **Dependency** vẽ mũi tên đứt nét (---->)
5. Các field nullable đánh dấu `[nullable]` hoặc `[0..1]` trong comment
6. Method `calculateRefund()` trên Booking là điểm trace về CBR-05 logic — nếu cô hỏi tại sao method ở đây, giải thích đây là business logic gắn với entity Booking, không phải UC23
