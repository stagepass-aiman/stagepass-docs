# ADR-004 — Money Type Representation

| Field             | Value                                                                                 |
|-------------------|---------------------------------------------------------------------------------------|
| **ID**            | ADR-004                                                                               |
| **Title**         | Money Type Representation                                                             |
| **Status**        | Accepted                                                                              |
| **Date**          | 2025-01-15                                                                            |
| **Author**        | StagePass Architecture                                                                |
| **Version**       | 1.0.0                                                                                 |
| **Traces To**     | PRD §7.3 (Revenue Split Model), PRD §9.1 (GST requirements), PRD §11 (Conventions),  |
|                   | NFR-REL-010, NFR-REL-012, NFR-REL-008, NFR-COMP-001,                                 |
|                   | ADR-003 §3.2.2 (REST wire format conventions)                                        |
| **Supersedes**    | —                                                                                     |
| **Superseded By** | —                                                                                     |

---

## Change Log

| Version | Date       | Author                  | Summary            |
|---------|------------|-------------------------|--------------------|
| 1.0.0   | 2025-01-15 | StagePass Architecture  | Initial acceptance |

---

## 1. Status

**Accepted.** This ADR is in effect from Phase 1. Every service that handles
monetary values must conform to this specification before its implementation PR
is merged. CI static analysis rules (Semgrep, ESLint, Ruff) enforce the
prohibitions below at every PR gate.

---

## 2. Context

### 2.1 Why this decision must be made before any code is written

StagePass is a financial platform. Every booking involves an exact monetary exchange
across three parties (Platform, Venue, Organiser) mediated by an escrow. Revenue
splits are computed once and become immutable ledger entries. Refunds are computed
from those stored entries, not by re-running the original arithmetic. Tax invoices
are generated per booking, itemising GST down to four decimal places.

The question of *how the platform represents money* is therefore not a library
preference — it is an integrity constraint. A single rounding error that drifts a
`venueShare` by ₹0.01 per booking will accumulate across thousands of bookings into
a reconciliation discrepancy that is invisible until disbursement time and
potentially impossible to audit back to its origin.

The platform is polyglot: Java 21 (Booking, Payment, Disbursement, Seat Inventory,
Auth), TypeScript (Event, Venue, Ticket, Check-in, Notification, Waitlist), Python
(Search, Recommendation, Chatbot, Fraud Detection, Analytics). Money flows across
all three runtimes and across REST and Kafka boundaries. Without a single,
non-negotiable contract for representation at every layer — in-memory, in the
database, on the wire, in Kafka payloads — values will silently corrupt at
language boundaries.

This decision eliminates an entire class of financial correctness bugs by
specification.

### 2.2 The root problem: IEEE 754 floating-point arithmetic

The fundamental reason floating-point is unusable for money is **IEEE 754 binary
representation**: not every decimal fraction has an exact binary representation.
The canonical example — `0.1 + 0.2` — evaluates to
`0.30000000000000004` in every IEEE 754 double-precision system, not `0.3`.

For a single transaction this looks trivial. At scale it is not. Consider a platform
fee of 5% applied to ₹1,500.00:

```
// IEEE 754 double — what you get
1500.0 * 0.05 = 75.00000000000001

// Exact decimal arithmetic — what you need
1500.0000 × 0.0500 = 75.0000
```

Multiply this across 10,000 bookings per day. The drift compounds. The ledger no
longer balances. GST remittance is wrong. Disbursement amounts do not reconcile
with bank statements. The Indian GST Act requires that tax invoices are accurate;
a systematic rounding error in tax computation is not a software bug — it is
regulatory non-compliance.

This is not a theoretical concern. Amazon, Stripe, and every serious payment
processor prohibit floating-point for money. The IEEE 754 section of Kleppmann
*DDIA* (§6, "The Trouble with Distributed Systems") and the Martin Fowler
*Money* pattern both articulate the same conclusion. NFR-REL-010 encodes this
as a non-negotiable constraint: zero occurrences of `float` or `double` in
money-handling code paths, enforced by CI static analysis.

### 2.3 The secondary problem: currency coupling

Storing an amount without its currency is an implicit assumption that the system
is single-currency. StagePass v1.0 is INR-only (PRD constraint C-07), which
makes that assumption locally true today. But a bare `amount` field with no
`currency` field:

1. Makes the currency assumption invisible — a reader of the code cannot tell
   if the omission was deliberate or accidental
2. Creates a migration burden if multi-currency is added later (every storage
   schema, every API, every test must change)
3. Allows a caller to accidentally compare or add amounts that are in different
   currencies if the system ever expands

The correct pattern, articulated in Martin Fowler's *Money* pattern and in Evans'
*Domain-Driven Design*, is: **amount and currency are a single value object and
are always stored and transmitted together**. There is no `amount` field that
is not accompanied by a `currency` field. This makes the INR assumption explicit
and auditable, and makes future multi-currency support a localised change.

### 2.4 The revenue split problem: immutability and the recomputation trap

The revenue split model (PRD §7.3) distributes every ticket sale across three
parties:

```
platformFee    = salePrice × platform_fee_rate
venueShare     = salePrice × venue_share_rate
organiserShare = salePrice − platformFee − venueShare
```

The rates (`platform_fee_rate`, `venue_share_rate`) are configurable. The Admin
can change the platform fee rate between events. The Organiser negotiates a new
Venue share for each VenueBooking. If a refund is computed by *re-running* this
formula at refund time, using the *current* rates, the refund amount will differ
from the original split if rates have changed in the interim. The ledger will
not balance.

The correct pattern is: compute the split *once*, at booking-confirm time, store
the three resulting amounts as an immutable `RevenueSplit` record, and compute
all refunds from those stored values. NFR-REL-012 makes this non-negotiable:
"No service may update a RevenueSplit record after creation." This ADR specifies
how those amounts are stored to ensure that the stored values can be used as
precise arithmetic inputs without re-introducing floating-point errors at refund
time.

### 2.5 The GST problem: display price vs. split base

PRD §9.1 specifies:

- **Customer-facing display:** GST-inclusive price (what the customer pays)
- **Revenue split computation:** computed on the GST-exclusive base price
- **Tax invoice:** must separately itemise base price, GST rate, GST amount, and
  total (GST-inclusive) price

This means the platform must track *two* prices per booking line:

1. `displayPrice` — GST-inclusive; what the customer sees and pays
2. `basePrice` — GST-exclusive; the input to revenue split arithmetic

And a third derived value:

3. `gstAmount` = `displayPrice` − `basePrice`

All three must be stored with full precision in the `RevenueSplit` record. The
`gstAmount` is remitted separately and is not included in the Organiser or Venue
revenue share calculation. This ADR specifies the exact field structure for the
`RevenueSplit` record to accommodate this accounting model.

> **Scope boundary:** The exact GST rate (18% on platform fee? 28% on
> entertainment? compound GST on multi-state events?) is deferred to ADR-008
> (Revenue Split and Disbursement Model). This ADR specifies how GST amounts are
> *stored and represented* once they are computed. The rate computation logic is
> ADR-008's responsibility.

### 2.6 Scope of this decision

This ADR decides:
- The in-memory money type in each language runtime
- The JSON wire format for all REST and Kafka payloads
- The database column types for PostgreSQL (relational services) and MongoDB
  (document services)
- The Money value object structure: fields, coupling, and constraints
- The RevenueSplit record schema: field names, types, and immutability contract
- Arithmetic rules: rounding mode, scale discipline, where rounding is permitted
- The GST tracking model in the RevenueSplit record

This ADR does not decide:
- GST rate values or the legal treatment of GST across parties (→ ADR-008)
- The Razorpay payment gateway integration or escrow model (→ ADR-008)
- The Booking saga step sequence (→ ADR-005)
- Currency conversion or multi-currency (out of scope for v1.0, PRD C-07)

---

## 3. Decision

### 3.1 Core constraint: exact decimal types only, everywhere

**Floating-point types are unconditionally prohibited for monetary values.**

| Runtime    | Prohibited          | Required                          |
|------------|---------------------|-----------------------------------|
| Java 21    | `float`, `double`   | `java.math.BigDecimal`            |
| TypeScript | `number`            | `Decimal` from `decimal.js`       |
| Python     | `float`             | `decimal.Decimal`                 |
| PostgreSQL | `FLOAT`, `REAL`, `DOUBLE PRECISION` | `NUMERIC(19,4)`   |
| MongoDB    | `Number` (Double)   | `Decimal128`                      |
| JSON wire  | JSON `number`       | JSON `string` (e.g. `"1250.0000"`) |

This constraint is enforced by CI static analysis. Any PR that introduces a
prohibited type in a money-handling code path fails the CI gate:
- **Java:** Semgrep rule matching `float`/`double` type declarations in
  `*/payment/**`, `*/booking/**`, `*/disbursement/**`, `*/seat_inventory/**`
- **TypeScript:** ESLint `no-restricted-syntax` rule matching `number` type
  annotations in money-related modules; Pact contract tests validate wire format
- **Python:** Ruff custom rule + Pydantic validators at all service boundaries

### 3.2 In-memory money type per runtime

#### 3.2.1 Java 21 — `java.math.BigDecimal`

```java
// Money is a value record. amount and currency are always coupled.
// BigDecimal is immutable. Arithmetic always specifies scale and rounding mode.
public record Money(BigDecimal amount, Currency currency) {

    // Canonical scale: 4 decimal places throughout all internal computation.
    private static final int SCALE = 4;
    private static final RoundingMode ROUNDING = RoundingMode.HALF_UP;

    // Constructor normalises scale on entry. No bare BigDecimal flows through
    // the domain layer without this normalisation.
    public Money {
        Objects.requireNonNull(amount, "amount must not be null");
        Objects.requireNonNull(currency, "currency must not be null");
        // Normalise to canonical scale immediately. This prevents scale drift
        // where 75 and 75.0000 are equal by value but not by toString().
        amount = amount.setScale(SCALE, ROUNDING);
    }

    // Arithmetic — always returns a new Money; BigDecimal is immutable.
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount).setScale(SCALE, ROUNDING), currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.subtract(other.amount).setScale(SCALE, ROUNDING), currency);
    }

    // Multiply by a rate (e.g. 0.05 for 5%). The rate itself is a BigDecimal.
    // HALF_UP rounding applied at 4dp immediately — no intermediate accumulation
    // at higher precision (which would defer and concentrate rounding error).
    public Money multiply(BigDecimal rate) {
        return new Money(
            this.amount.multiply(rate).setScale(SCALE, ROUNDING),
            currency
        );
    }

    // Output boundary: human-readable display. Rounds to 2dp for INR display.
    // This is the ONLY place 2dp rounding is permitted — at the display layer.
    public String toDisplayString() {
        return currency.getSymbol() + " " + amount.setScale(2, ROUNDING);
    }

    // Wire format serialisation: amount as STRING, currency as ISO-4217 code.
    public MoneyDto toDto() {
        return new MoneyDto(amount.toPlainString(), currency.getCurrencyCode());
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(this.currency, other.currency);
        }
    }
}

// Why no Lombok? System prompt: "No Lombok." Java records provide equals,
// hashCode, toString, and accessors natively. Records are also naturally
// immutable — no setter can be accidentally added.
```

**Why `BigDecimal` and not `long` (paise)?**
See §5.2 (Alternatives Considered). Short answer: BigDecimal expresses rates and
split arithmetic naturally; paise-integer requires dividing by 100 to compute a
5% fee, which reintroduces a division and rounding step that must be explicitly
managed and is a source of bugs. The rounding is not eliminated — it is hidden.

**Why `HALF_UP`?**
It is the rounding mode required by Indian GST regulations for tax computation.
It is the rounding mode used by Razorpay and every serious INR payment processor.
It is also the intuitive "school arithmetic" rounding mode (0.5 rounds up), which
means computed splits are auditable by any reviewer without understanding IEEE
754 banker's rounding.

#### 3.2.2 TypeScript — `Decimal` from `decimal.js`

```typescript
// decimal.js provides arbitrary-precision decimal arithmetic.
// It is the standard choice in the Node/TypeScript ecosystem for money.
// Never use the native `number` type for monetary values.

import Decimal from 'decimal.js';

// Configure once at application startup — before any monetary arithmetic.
// Scale 4, HALF_UP (Decimal.ROUND_HALF_UP = 4 in decimal.js enum).
Decimal.set({ precision: 20, rounding: Decimal.ROUND_HALF_UP });

// Money value object for TypeScript services.
// Immutable by design: readonly fields, no setters.
export class Money {
  readonly amount: Decimal;
  readonly currency: string; // ISO-4217, e.g. "INR"

  constructor(amount: Decimal | string, currency: string) {
    // Accept string from wire format or Decimal from internal computation.
    // Normalise to 4dp immediately.
    this.amount = new Decimal(amount).toDecimalPlaces(4, Decimal.ROUND_HALF_UP);
    this.currency = currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount.plus(other.amount), this.currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount.minus(other.amount), this.currency);
  }

  // Rate is a Decimal representing a fraction (e.g. new Decimal('0.05') for 5%).
  multiply(rate: Decimal): Money {
    return new Money(this.amount.times(rate), this.currency);
  }

  // Wire serialisation. Returns the DTO shape consumed by JSON.stringify().
  toDto(): { amount: string; currency: string } {
    // toFixed(4) produces the 4dp string format mandated by §3.3.
    return { amount: this.amount.toFixed(4), currency: this.currency };
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
  }
}

// Why decimal.js and not big.js or bignumber.js?
// decimal.js supports the full set of rounding modes (including ROUND_HALF_UP)
// and is configured with a global precision and rounding mode via Decimal.set().
// big.js is simpler but does not support configurable rounding modes.
// bignumber.js is equivalent; decimal.js is more widely used in the TypeScript
// ecosystem for financial applications and has better TypeScript typings.
```

#### 3.2.3 Python — `decimal.Decimal`

```python
from decimal import Decimal, ROUND_HALF_UP, getcontext
from dataclasses import dataclass

# Configure the decimal context once at module load.
# Precision 20 gives ample headroom for intermediate arithmetic before rounding.
# ROUND_HALF_UP matches the Java and TypeScript rounding mode.
getcontext().prec = 20
getcontext().rounding = ROUND_HALF_UP

# Canonical scale quantiser. Used to normalise all amounts to 4dp.
QUANTIZE_4DP = Decimal('0.0001')

@dataclass(frozen=True)  # frozen=True: Python equivalent of an immutable value object
class Money:
    amount: Decimal
    currency: str  # ISO-4217, e.g. "INR"

    def __post_init__(self) -> None:
        # Pydantic or dataclass validators are not enough: the normalisation
        # must happen at construction, not just at API boundary validation.
        # object.__setattr__ required because frozen=True blocks direct assignment.
        object.__setattr__(
            self, 'amount',
            Decimal(str(self.amount)).quantize(QUANTIZE_4DP)
        )
        if not self.currency:
            raise ValueError("currency must not be empty")

    def add(self, other: 'Money') -> 'Money':
        self._assert_same_currency(other)
        return Money(self.amount + other.amount, self.currency)

    def subtract(self, other: 'Money') -> 'Money':
        self._assert_same_currency(other)
        return Money(self.amount - other.amount, self.currency)

    def multiply(self, rate: Decimal) -> 'Money':
        return Money(self.amount * rate, self.currency)

    def to_dto(self) -> dict:
        # Wire format: amount as STRING, 4dp.
        return {"amount": str(self.amount), "currency": self.currency}

    def _assert_same_currency(self, other: 'Money') -> None:
        if self.currency != other.currency:
            raise ValueError(f"Currency mismatch: {self.currency} vs {other.currency}")

# Pydantic validator used at all FastAPI boundary layers.
# This is the parser that converts the string-amount wire format into the
# internal Money type. No float ever enters the domain layer.
from pydantic import BaseModel, validator

class MoneySchema(BaseModel):
    amount: str   # Received as string from wire; parsed into Decimal
    currency: str

    @validator('amount')
    def must_be_valid_decimal(cls, v: str) -> str:
        try:
            Decimal(v)
        except Exception:
            raise ValueError(f"amount must be a valid decimal string, got: {v!r}")
        return v

    def to_money(self) -> Money:
        return Money(Decimal(self.amount), self.currency)
```

### 3.3 JSON wire format — all REST APIs and all Kafka payloads

**The monetary amount is always serialised as a JSON string, never as a JSON number.**

```json
{
  "amount": "1250.0000",
  "currency": "INR"
}
```

**Rules:**
- `amount` field: always a `string`, always exactly 4 decimal places
  (i.e. trailing zeros are not stripped — `"75.0000"` not `"75"` or `"75.0"`)
- `currency` field: always a `string`, always ISO-4217 three-character code
  (`"INR"` in v1.0)
- These two fields always appear together. There is no API field named `amount`
  that is not accompanied by a `currency` field in the same object

**Why string, not number?**

JSON numbers are parsed by every JSON parser as IEEE 754 doubles. A receiving
service that parses `1250.0001` as a JSON number and converts it to a
`double` has already corrupted the value before any business logic runs. The
corruption happens in the JSON deserialiser, not in application code — it is
invisible, and it is not guarded by any type system check.

By using a string, the contract is:
- The serialiser is responsible for producing a valid decimal string with 4dp
- The deserialiser parses the string into the appropriate exact decimal type
  (`BigDecimal`, `Decimal`, `decimal.Decimal`)
- No information is lost at the boundary, regardless of the receiving language

This is aligned with ADR-003 §3.2.2 (REST conventions) which specifies this
format for all monetary fields in inter-service REST calls.

**Example: a `RevenueSplit` event payload on Kafka**

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "booking.revenue-split-created",
  "bookingId": "550e8400-e29b-41d4-a716-446655440000",
  "eventId":   "7e9d3c2b-18ae-4f1a-b73e-8dcc5432a100",
  "salePrice": {
    "amount": "1475.0000",
    "currency": "INR"
  },
  "basePrice": {
    "amount": "1250.0000",
    "currency": "INR"
  },
  "gstAmount": {
    "amount": "225.0000",
    "currency": "INR"
  },
  "platformFee": {
    "amount": "62.5000",
    "currency": "INR"
  },
  "venueShare": {
    "amount": "250.0000",
    "currency": "INR"
  },
  "organiserShare": {
    "amount": "937.5000",
    "currency": "INR"
  },
  "platformFeeRate": "0.0500",
  "venueShareRate": "0.2000",
  "gstRate": "0.1800",
  "computedAt": "2025-01-15T10:30:00Z"
}
```

Note: `platformFeeRate`, `venueShareRate`, and `gstRate` are stored alongside
the computed amounts so that refund computations can reconstruct the percentages
without needing to look up current Admin configuration (which may have changed).

### 3.4 Database storage types

#### 3.4.1 PostgreSQL — `NUMERIC(19,4)` for amount, `CHAR(3)` for currency

```sql
-- Canonical column definition for any monetary amount in PostgreSQL.
-- Used in: Booking (write model), Payment, Disbursement, Ticket.

-- NUMERIC(19,4):
--   Precision 19 total digits.
--   Scale 4: exactly 4 decimal places stored.
--   This covers amounts up to 999,999,999,999,999.9999 — sufficient for any
--   realistic ticket sale or disbursement amount in INR.
--   PostgreSQL NUMERIC is an exact decimal type — no floating-point.

-- Why precision 19?
--   The maximum credible single transaction on this platform is a sold-out
--   stadium (50,000 seats) at a premium price (₹10,000 per seat):
--     50,000 × 10,000 = ₹500,000,000 (9 digits before the decimal)
--   Adding 4 decimal places = 13 significant digits needed.
--   Precision 19 gives comfortable headroom (6 extra digits) for aggregate
--   queries (SUM across all bookings in a disbursement batch).

-- Why CHAR(3) for currency?
--   ISO-4217 currency codes are always exactly 3 characters.
--   CHAR(3) enforces this at the database level — no variable-length padding.
--   VARCHAR(3) would also work but CHAR(3) signals the fixed nature of the code.

CREATE TABLE revenue_split (
    id                  UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id          UUID            NOT NULL REFERENCES booking(id),
    event_id            UUID            NOT NULL,

    -- The price the Customer paid (GST-inclusive display price).
    sale_price_amount   NUMERIC(19, 4)  NOT NULL,
    sale_price_currency CHAR(3)         NOT NULL DEFAULT 'INR',

    -- The GST-exclusive base price used for split computation.
    base_price_amount   NUMERIC(19, 4)  NOT NULL,
    base_price_currency CHAR(3)         NOT NULL DEFAULT 'INR',

    -- GST amount = sale_price - base_price. Tracked separately for invoice.
    gst_amount          NUMERIC(19, 4)  NOT NULL,
    gst_currency        CHAR(3)         NOT NULL DEFAULT 'INR',
    gst_rate            NUMERIC(5, 4)   NOT NULL,   -- e.g. 0.1800 for 18%

    -- The three split amounts. These are the immutable ledger values.
    platform_fee_amount NUMERIC(19, 4)  NOT NULL,
    platform_fee_currency CHAR(3)       NOT NULL DEFAULT 'INR',
    platform_fee_rate   NUMERIC(5, 4)   NOT NULL,   -- e.g. 0.0500 for 5%

    venue_share_amount  NUMERIC(19, 4)  NOT NULL,
    venue_share_currency CHAR(3)        NOT NULL DEFAULT 'INR',
    venue_share_rate    NUMERIC(5, 4)   NOT NULL,   -- e.g. 0.2000 for 20%

    organiser_share_amount   NUMERIC(19, 4) NOT NULL,
    organiser_share_currency CHAR(3)        NOT NULL DEFAULT 'INR',

    -- Disbursement tracking. NULL until disbursed.
    disbursed_at        TIMESTAMPTZ,
    disbursement_id     UUID,

    -- Immutability guard: no UPDATE is permitted on this table.
    -- Enforced by: (a) the service layer never calling UPDATE on this table,
    -- (b) a PostgreSQL row-level trigger (below) that raises an exception.
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT now()

    -- NOTE: there is deliberately no updated_at column. The absence of this
    -- column is a design signal: this record is append-only.
);

-- Immutability trigger: any UPDATE on revenue_split raises an exception.
-- This is a defence-in-depth measure. The service layer is the first line of
-- defence (NFR-REL-012). The trigger is the second.
CREATE OR REPLACE FUNCTION prevent_revenue_split_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    RAISE EXCEPTION
        'revenue_split records are immutable. '
        'bookingId=% attempted mutation.',
        OLD.booking_id
    USING ERRCODE = 'restrict_violation';
END;
$$;

CREATE TRIGGER revenue_split_immutability
    BEFORE UPDATE ON revenue_split
    FOR EACH ROW EXECUTE FUNCTION prevent_revenue_split_update();

-- Integrity constraint: the three split components must sum to base_price.
-- This catches arithmetic errors at write time.
ALTER TABLE revenue_split
    ADD CONSTRAINT split_sums_to_base_price CHECK (
        platform_fee_amount + venue_share_amount + organiser_share_amount
        = base_price_amount
    );
```

**Why `NUMERIC(19,4)` and not `NUMERIC(20,4)` or `NUMERIC(15,2)`?**

- **Not 2 decimal places (`NUMERIC(15,2)`):** Indian GST is applied at 18% on a
  GST-exclusive base. A base price of ₹1,250.00 produces a GST amount of ₹225.00
  — clean. But a base price of ₹1,333.00 at 18% produces ₹239.94 — already 2dp.
  At a GST rate of 12%, ₹1,001.00 produces ₹120.12 — fine. But ₹999.99 at 18%
  produces ₹179.9982 — requiring 4dp before rounding. The platform must not round
  intermediate GST amounts before storing; 4dp gives a safe boundary.
- **Not arbitrary precision:** A fixed scale is required for `CHECK` constraints
  (the integrity check above requires known scale), for indexed numeric ranges,
  and for predictable comparison behaviour in SQL. Arbitrary precision (`NUMERIC`
  without scale) disables the `CHECK` constraint's equality test.
- **19 total digits:** See rationale in the SQL comment above. 9 significant
  integer digits (for a worst-case ₹500M transaction) + 4 scale digits + 6
  buffer digits = 19.

**Transaction isolation for monetary writes:** All writes to `revenue_split`,
`disbursement_record`, and the booking-payment state machine use `SERIALIZABLE`
isolation (NFR-REL-008). `READ COMMITTED` is insufficient for the revenue split
integrity check — a concurrent write could produce two `revenue_split` records
for the same `booking_id` before either transaction's `CHECK` constraint fires.
`SERIALIZABLE` eliminates this anomaly.

#### 3.4.2 MongoDB — `Decimal128` for all monetary fields

MongoDB's `Decimal128` type is an IEEE 754-2008 decimal128 floating-point type
— a 128-bit decimal (not binary) type with 34 significant decimal digits. It is
an exact decimal representation and is the only MongoDB BSON type suitable for
monetary storage.

```javascript
// Event Service (NestJS, TypeScript) — storing ticket price tiers in MongoDB.
// Venue Service (NestJS, TypeScript) — storing Venue share rates.
// Booking read model (CQRS read side) — MongoDB document for query.

import { Schema, Prop, SchemaFactory } from '@nestjs/mongoose';
import { Types } from 'mongoose';

// Mongoose schema for a PricingTier subdocument.
// All monetary fields use Decimal128 — never Number.
@Schema({ _id: false })
class PricingTierDocument {

  @Prop({ required: true })
  tierId: string;

  @Prop({ required: true })
  name: string;              // e.g. "PREMIUM", "STANDARD"

  // Decimal128 is stored as MongoDB's native decimal type.
  // On read, it is converted to a string before leaving the service
  // (to avoid any intermediate conversion to JavaScript number).
  @Prop({ type: Types.Decimal128, required: true })
  priceAmount: Types.Decimal128;

  @Prop({ required: true, enum: ['INR'] })
  priceCurrency: string;

  @Prop({ type: Types.Decimal128, required: true })
  surgePriceAmount: Types.Decimal128;    // Current effective price (may be surge)
}

// Serialisation helper: always convert Decimal128 to string when building DTOs.
// Never let a Decimal128 flow into JSON serialisation directly — the default
// serialisation produces { "$numberDecimal": "1250.0000" }, which is the MongoDB
// extended JSON format, not the wire format defined in §3.3.
function decimal128ToDto(d: Types.Decimal128): { amount: string; currency: string } {
  return {
    amount: d.toString(),   // Produces "1250.0000" — matches §3.3 wire format
    currency: 'INR',
  };
}
```

**Why `Decimal128` and not `Number` (JavaScript/BSON Double)?**

MongoDB's default numeric BSON type when using Mongoose with a `Number` field is
`BSON Double` — an IEEE 754 binary double, identical to JavaScript's `number`.
The same floating-point problem described in §2.2 applies. Storing ₹1,250.0001
as a BSON Double will silently corrupt the stored value. This is not a theoretical
concern: MongoDB's own documentation warns against using `Double` for monetary values
and recommends `Decimal128` explicitly.

The document services (Event, Venue) store ticket prices and revenue share rates —
both of which are inputs to split computation in the Booking Service. If these
values are corrupted in MongoDB, every downstream split computation inherits the
corruption. The corruption is invisible until reconciliation time.

**Why not store monetary values in MongoDB as strings?**

String storage in MongoDB loses the ability to do range queries
(`db.events.find({ price: { $gte: 500, $lte: 2000 } }`) without converting
every stored string to a number at query time — which brings floating-point back.
`Decimal128` provides both exact storage and native numeric query capability.

### 3.5 Money value object: coupling rule

**There is no field named `amount` in any service's domain model, API schema,
database column definition, or Kafka payload that is not accompanied by a
`currency` field in the same object.**

The canonical shape in all contexts:

```
{ amount: "<decimal string>", currency: "<ISO-4217>" }
```

Rationale: INR is the only currency in v1.0 (PRD C-07). However, hardcoding
`INR` implicitly would mean that every service assumes the currency without
expressing it. The `currency` field makes the assumption explicit and auditable.
If multi-currency is introduced in v2.0, the schema change is additive: existing
records already carry the currency code.

**Anti-pattern (prohibited):**
```json
{ "ticketPrice": "1250.0000" }
```

**Correct:**
```json
{ "ticketPrice": { "amount": "1250.0000", "currency": "INR" } }
```

### 3.6 Revenue split: compute once, store immutably

The revenue split computation is performed **exactly once**: in the Booking Service,
at the moment the booking transitions from `PAYMENT_CONFIRMED` to `SEATS_COMMITTED`.

```
Step 1. Read salePrice (GST-inclusive) from the booking line item.
Step 2. Read gstRate from the Event (set by Admin via platform config).
Step 3. Compute basePrice = salePrice / (1 + gstRate). Round to 4dp HALF_UP.
Step 4. Compute gstAmount = salePrice - basePrice. Store this — do not recompute.
Step 5. Read platformFeeRate from Admin configuration (snapshot at booking time).
Step 6. Read venueShareRate from the VenueBooking record (locked at ACCEPTED state).
Step 7. Compute platformFee = basePrice × platformFeeRate. Round to 4dp HALF_UP.
Step 8. Compute venueShare = basePrice × venueShareRate. Round to 4dp HALF_UP.
Step 9. Compute organiserShare = basePrice - platformFee - venueShare. [No rounding
        needed: this is the remainder, making the three components sum exactly.]
Step 10. Write RevenueSplit record. This write and the booking state transition
         happen in the same SERIALIZABLE transaction.
Step 11. The RevenueSplit record is never updated again.
```

**Why compute `organiserShare` as a remainder (Step 9) rather than as a rate?**

Arithmetic rounding on rates always produces a residual. If all three components
are computed by multiplying rates against basePrice and all three are rounded to
4dp independently, they will not sum exactly to `basePrice` due to rounding at
each step. This is the *split rounding residual* problem.

The solution: assign the platform fee and venue share by rate, then assign
the organiser share as the mathematical remainder. The remainder is always exact
(it is a subtraction, not a multiplication) and the three components always sum
exactly to `basePrice`. The Organiser bears the rounding residual — a difference
of at most ₹0.0001 per ticket, which is an acceptable and explicitly documented
choice.

**Refund computation (NFR-REL-012):**

```
refundSalePrice = refundPolicy.refundableAmount(salePrice, cancellationTimestamp)
refundRatio     = refundSalePrice / salePrice      [Decimal arithmetic, 4dp]

refundPlatformFee    = storedPlatformFee × refundRatio
refundVenueShare     = storedVenueShare × refundRatio
refundOrganiserShare = storedOrganiserShare × refundRatio

// Again: platform and venue by ratio, organiser as remainder.
refundOrganiserShare = refundSalePrice
    - refundPlatformFee
    - refundVenueShare
```

The `storedPlatformFee`, `storedVenueShare`, `storedOrganiserShare` values read
here are the amounts written at booking time — never recomputed from current rates.
This is the enforcement of NFR-REL-012 at the code level.

### 3.7 GST tracking in the RevenueSplit record

The GST treatment in the split model follows PRD §9.1:

```
salePrice  = displayPrice (GST-inclusive — what the Customer pays)
basePrice  = salePrice / (1 + gstRate)      (GST-exclusive base)
gstAmount  = salePrice - basePrice

revenue split is computed on basePrice, not on salePrice
gstAmount is tracked separately and remitted to tax authorities
```

**Fields stored in the RevenueSplit record:**

| Field              | Type          | Description                                    |
|--------------------|---------------|------------------------------------------------|
| `salePriceAmount`  | NUMERIC(19,4) | GST-inclusive price (Customer's charge)        |
| `basePriceAmount`  | NUMERIC(19,4) | GST-exclusive base (split computation input)   |
| `gstAmount`        | NUMERIC(19,4) | gstAmount = salePrice − basePrice              |
| `gstRate`          | NUMERIC(5,4)  | Snapshot of GST rate at booking time           |
| `platformFeeAmount`| NUMERIC(19,4) | platformFee = basePrice × platformFeeRate      |
| `platformFeeRate`  | NUMERIC(5,4)  | Snapshot of Admin-configured rate              |
| `venueShareAmount` | NUMERIC(19,4) | venueShare = basePrice × venueShareRate        |
| `venueShareRate`   | NUMERIC(5,4)  | Snapshot from VenueBooking (locked at ACCEPTED)|
| `organiserShareAmount` | NUMERIC(19,4) | basePrice − platformFee − venueShare      |

Rates are stored as snapshots at booking time so that the split can be
audited and refund computations can be verified without depending on
current Admin configuration (which may have changed since booking).

**The GST amount is not part of any party's disbursement.** It is collected
from the Customer as part of the GST-inclusive sale price and remitted to
tax authorities separately. It appears on the tax invoice as a separate line
item (NFR-COMP-001) but does not flow through the three-party split.

> **Scope note:** The question of which entity (Platform, Organiser, or Venue)
> is the GST-registered supplier, how the GST is remitted, and whether the
> platform fee itself attracts a separate GST charge is deferred to ADR-008.
> This ADR only specifies that the GST amount is tracked as a separate stored
> field in the RevenueSplit record.

### 3.8 Arithmetic rules — rounding discipline

1. **Scale throughout:** all intermediate arithmetic is performed at 4 decimal
   places. Do not reduce precision mid-computation. The only place where scale
   may be reduced is at the display boundary (2dp for Customer-facing prices).

2. **Rounding mode:** `HALF_UP` everywhere, without exception.
   Do not use `HALF_EVEN` (banker's rounding), `FLOOR`, `CEILING`, or
   `TRUNCATE` for any monetary value.

3. **Rounding at boundaries only:** rounding is permitted only at:
   - **Write to database:** amount normalised to 4dp before `INSERT`
   - **Serialise to wire:** `toFixed(4)` / `toPlainString()` at DTO construction
   - **Display to user:** reduce to 2dp for Customer UI display only

4. **Never round intermediate:** do not round a sub-total before using it in the
   next computation step. The revenue split steps in §3.6 are a pipeline; only
   the final result of each step (platformFee, venueShare, organiserShare) is
   rounded to 4dp before storage. Do not round basePrice before computing
   platformFee.

5. **Rate representation:** percentage rates (e.g. 5%) are stored and used in
   computation as decimal fractions (`0.0500`), not as percentages (`5`). This
   prevents a class of bugs where a rate of `5` is inadvertently used as a
   multiplier (producing `basePrice × 5` instead of `basePrice × 0.05`).

---

## 4. Consequences

### 4.1 Positive

**Financial correctness is guaranteed at all boundaries.** Every entry into and
exit from the domain layer is guarded: the Money constructor normalises scale,
the DTO serialiser always produces 4dp strings, the PostgreSQL `NUMERIC(19,4)`
column rejects any value outside scale, and the `Decimal128` MongoDB type stores
exact decimal values. There is no point in the system where a floating-point value
can silently enter a monetary field.

**Audit and reconciliation are straightforward.** The `RevenueSplit` table is an
immutable ledger. Every refund can be traced to its originating RevenueSplit record.
The three split components sum exactly to `basePrice` (enforced by the `CHECK`
constraint). The stored rate snapshots mean the computation can be reconstructed
from the ledger alone, without querying current Admin configuration.

**GST compliance is structurally supported.** The separation of `salePrice`,
`basePrice`, and `gstAmount` in the RevenueSplit record means that generating a
GST-compliant tax invoice (PRD §9.1, NFR-COMP-001) requires reading a single record
— no on-the-fly computation at invoice generation time.

**Polyglot safety.** Money flowing from a Python Recommendation Service's
price-filter API into a TypeScript Event Service's MongoDB document and then into
a Java Booking Service's PostgreSQL revenue split record passes through three
language runtimes, two database engines, and REST wire formats. The string-based
wire format means no runtime's JSON parser can silently corrupt a value. The
consistent Money value object contract means every runtime produces and consumes
values in the same shape.

**The Currency field future-proofs multi-currency.** When v2.0 introduces USD or
GBP events for international artists, the currency is already a first-class field
everywhere. No schema migration is required to add it — it is already there.

### 4.2 Negative / costs

**Slightly more verbose API payloads.** `{ "amount": "1250.0000", "currency": "INR" }`
is heavier than a bare `1250.00`. Over thousands of concurrent WebSocket seat-map
updates, this is negligible — seat state changes do not carry monetary amounts.
Revenue payloads are low-volume and the verbosity is acceptable.

**Additional developer discipline required.** A developer unfamiliar with this
convention may instinctively write `const price: number = 1250`. Code review,
ESLint rules, and the CI Semgrep gate are the enforcement mechanisms. Onboarding
documentation must make this rule visible immediately.

**Slightly more complex PostgreSQL schema.** Storing `amount` and `currency` as
separate columns (rather than a single JSON column) is more verbose. The benefit —
typed numeric operations, range queries, aggregate `SUM`, the `CHECK` constraint —
outweighs the verbosity cost.

**`Decimal128` MongoDB serialisation quirk.** The default Mongoose serialiser
emits `{ "$numberDecimal": "1250.0000" }` for a `Decimal128` field — the MongoDB
extended JSON format. Service code must explicitly convert `Decimal128` to string
at the DTO layer (the `decimal128ToDto` helper in §3.2.2). This must be part of
every TypeScript service's review checklist.

### 4.3 Risks and mitigations

| Risk | Mitigation |
|------|------------|
| Developer adds `float` or `number` to a money field | Semgrep / ESLint rule blocks the PR at CI gate |
| Decimal128 serialised as MongoDB extended JSON leaks to API clients | Code review checklist; integration tests assert wire format is plain string |
| `organiserShare` rounding residual accumulates to a visible discrepancy | Residual is at most ₹0.0001 per ticket; documented explicitly; integration test asserts CHECK constraint holds |
| RevenueSplit record mutated despite the constraint | PostgreSQL trigger raises exception; service layer logs and alerts; no silent update possible |
| New developer uses `HALF_EVEN` rounding from Java's default BigDecimal | Code review; explicit `ROUND_HALF_UP` in all Money arithmetic methods; linting rule that flags `.setScale(N)` without a RoundingMode argument |

---

## 5. Alternatives Considered

### 5.1 Alternative: IEEE 754 `double` / `float` / `number`

**What it is:** Use the native floating-point type in each language runtime:
`double` in Java, `number` in TypeScript, `float` in Python.

**Why it seems appealing:** Zero additional dependencies. Native JSON
serialisation (JSON `number` type). Familiar to most developers.

**Why it is rejected:**

The IEEE 754 binary representation of decimal fractions is not exact. The
canonical example:

```python
>>> 0.1 + 0.2
0.30000000000000004

>>> 1500.0 * 0.05  # 5% platform fee on ₹1,500
75.00000000000001
```

This error is not a rounding error that HALF_UP can fix — it is an
**exact representation error** in the binary significand. No rounding mode
applied after the fact can produce the correct decimal value, because the
information has already been lost.

For a financial platform with thousands of daily transactions, split
computations, GST calculations, refund computations, and disbursement aggregates,
this drift is not theoretical — it is systematic and cumulative. The GST Act
requires accuracy to 2dp on tax invoices; a systematic binary representation
error of `0.00000000000001` per multiplication will aggregate into visible
discrepancies at scale.

**Verdict:** Unconditionally rejected. NFR-REL-010 encodes this rejection as a
non-negotiable constraint enforced by CI static analysis.

### 5.2 Alternative: Integer minor units (paise)

**What it is:** Store all monetary values as integers in the smallest currency
unit. For INR, the smallest unit is the paisa (1 INR = 100 paise). A price of
₹1,250.00 is stored as `125000` (paise).

**Why it seems appealing:** Integer arithmetic is exact. No decimal library
required. PostgreSQL `BIGINT` is efficient. JSON integer type is parsed exactly
by all JSON parsers. This is the approach used by Stripe's API.

**Why it is a valid alternative, and why we reject it for this platform:**

This approach is correct and widely used. It eliminates floating-point entirely
by making the unit of storage the smallest indivisible unit. Integer arithmetic
on paise values is exact.

**The problem for StagePass specifically:**

Rate-based computations require division. A platform fee of 5% on ₹1,250.00
(125000 paise) requires:

```
platformFee_paise = 125000 × 5 / 100 = 6250 paise (exact — this works)
```

But what if the rate is 7.5% (a valid GST rate in some categories)?

```
platformFee_paise = 125000 × 75 / 1000 = 9375 paise (exact — this still works)
```

And at 18% GST, computing the GST-exclusive base from the GST-inclusive price?

```
base_paise = 125000 / 1.18 = 105932.203... paise — NOT an integer
```

The GST-exclusive base is not expressible as an exact integer in paise. The
division `/ 1.18` introduces a fractional result that must be rounded. The
rounding happens immediately at the first division — there is no way to defer
it. A 4dp `NUMERIC` approach defers rounding to the storage boundary, giving
more precise intermediate arithmetic.

Additionally, StagePass stores revenue share rates with up to 4dp (e.g.
venue share of 17.5%, stored as `0.1750`). In integer-unit arithmetic, this
requires careful management of the multiplier (multiply by 1750, divide by
10000) and is a source of arithmetic bugs when developers forget the conversion.

**Verdict:** Valid approach; rejected in favour of `NUMERIC(19,4)` / `Decimal`
because the GST-exclusive base computation introduces a fractional result that
cannot be represented exactly in integer paise, and because decimal arithmetic
with explicit rates is more auditable than integer-scaled arithmetic with
implicit rate conversions. If multi-currency (with sub-paisa denominators) were
required, this decision would be revisited.

### 5.3 Alternative: Amount as string only, no typed Money object

**What it is:** Store and transmit amounts as strings everywhere (matching the
wire format in §3.3), with no shared Money type. Each service parses the string
into whatever it needs at point of use.

**Why it seems appealing:** Simple. No library dependency. Works natively across
all runtimes.

**Why it is rejected:** Without a shared Money type, arithmetic happens ad-hoc
at point of use. There is no enforcement of scale, rounding mode, or currency
coupling. A developer can add two string-amounts by `parseFloat(a) + parseFloat(b)`,
reintroducing the floating-point problem. The string-only approach relies entirely
on developer discipline with no structural guardrail. The Money value object is
the guardrail.

### 5.4 Alternative: PostgreSQL `MONEY` type

**What it is:** PostgreSQL has a native `MONEY` type that stores a monetary
amount with 2 decimal places of fixed scale.

**Why it is rejected:** The PostgreSQL `MONEY` type has several well-documented
limitations: it is locale-dependent (output format changes with the database
locale setting), it does not support arithmetic with `NUMERIC` without casting
(introducing precision loss), and it is limited to 2 decimal places — insufficient
for the 4dp scale required by GST intermediate computation. It is generally
recommended against by PostgreSQL documentation for new applications. `NUMERIC(19,4)`
is the correct choice.

---

## 6. References

- Kleppmann, M. *Designing Data-Intensive Applications*, Chapter 6
  ("The Trouble with Distributed Systems") — floating-point reliability
- Fowler, M. *Money Pattern* — https://martinfowler.com/eaaCatalog/money.html
- Evans, E. *Domain-Driven Design* — Value Objects chapter
- IEEE 754-2008 standard — binary floating-point arithmetic
- IEEE 754-2008 §3.5 decimal128 — the standard underlying MongoDB's `Decimal128`
- decimal.js documentation — https://mikemcl.github.io/decimal.js/
- PostgreSQL documentation — NUMERIC type
  https://www.postgresql.org/docs/current/datatype-numeric.html
- MongoDB documentation — Decimal128 BSON type
  https://www.mongodb.com/docs/manual/reference/bson-types/#decimal128
- Indian GST Act, 2017 — invoicing requirements under Section 31
- PRD §7.3 (Revenue Split Model)
- PRD §9.1 (GST requirements)
- NFR-REL-010 (exact decimal types — non-negotiable constraint)
- NFR-REL-012 (revenue split immutability)
- NFR-REL-008 (SERIALIZABLE isolation for money operations)
- NFR-COMP-001 (GST-inclusive display, separate GST itemisation on invoice)
- ADR-003 §3.2.2 (REST wire format — amount as STRING)

---

## 7. Quick Self-Check

Before implementing any money-handling code, answer these questions without
looking at this document. If you cannot answer them, re-read the relevant section.

**Q1: A developer on the Ticket Service (TypeScript / Fastify) receives a REST
request body with `{ "price": 1250.5 }` as a JSON number. What is wrong, what
has already happened, and what must the code do?**

> The JSON number `1250.5` has already been parsed by the JavaScript/TypeScript
> JSON parser as an IEEE 754 double. If the value were `1250.7` (or any decimal
> fraction without an exact binary representation), the parsed value would already
> be corrupted before the code runs. The code must reject this payload — the
> contract is that `price` must be an object `{ "amount": "<string>", "currency":
> "INR" }`. The correct action is to return a 400 response with Problem Details
> (RFC 9457) indicating that `price.amount` must be a string and `price.currency`
> is required. The Pydantic/Zod schema validation at the boundary enforces this
> before any domain logic executes.

**Q2: A junior developer writes the platform fee computation as:**
`const platformFee = basePrice * platformFeeRate / 100;`
**Name every mistake in this line.**

> Three mistakes: (1) `basePrice` and `platformFeeRate` are `number` types, not
> `Decimal` — this is floating-point arithmetic; the result is not exact. The
> code should be `basePrice.multiply(platformFeeRate)` on `Decimal` instances.
> (2) `platformFeeRate` is divided by 100, which means the caller is storing
> the rate as a percentage integer (e.g. `5`) rather than as a decimal fraction
> (`0.0500`). This ADR requires rates to be stored and used as decimal fractions
> (§3.8, rule 5) — dividing by 100 at computation time is an implicit convention
> that is a source of bugs (forget the `/ 100` and the fee is 100× too large).
> The rate should be stored as `new Decimal('0.0500')` and used as a direct
> multiplier. (3) The result is a `number`, not a `Money` value object — the
> currency is lost. The result must be a `Money` instance with the same currency
> as `basePrice`.

**Q3: The Booking Service has just written a RevenueSplit record to PostgreSQL.
Thirty days later, the Admin changes the platform fee rate from 5% to 6%.
A Customer then cancels their 30-day-old booking. How is the refund platform fee
computed, and where does the code get the rate?**

> The refund platform fee is computed from the stored `platform_fee_amount` in
> the RevenueSplit record for that booking, multiplied by the `refundRatio`
> (refundable amount ÷ original sale price). The code reads the stored
> `platform_fee_amount` from the immutable RevenueSplit record. It does NOT read
> the current Admin platform fee rate (which is now 6%). It does NOT query the
> Admin configuration service. The stored `platform_fee_rate` field (which holds
> the snapshot value `0.0500` from booking time) may be used to verify the
> computation but the authoritative input is the stored `platform_fee_amount`
> value. This is the enforcement of NFR-REL-012 in code: the split amounts are
> immutable and are the sole source of truth for all downstream computations.
