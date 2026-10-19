# LP03: Black-Box Testing, Part 1

**Student:** Jose Horta  
**Course:** AD361 - Application Testing  
**Application:** North Vacation Booking

## 1. Test Analysis

The test basis is the reservation requirements in the supplied [LP03 assignment](LP03.md). The test design uses equivalence partitioning to select representative inputs and determine expected behavior without examining application code.

The four classifications are:

1. **Number of nights:** a calculated value that determines whether the reservation duration is valid.
2. **Number of guests:** an input that determines reservation validity and whether an additional guest fee applies.
3. **Check-in and checkout dates:** inputs whose ordering must be valid and whose difference determines the number of reserved nights.
4. **Availability:** a condition that must hold for every requested night.

### Requirements used to determine expected results

| ID | Requirement |
| --- | --- |
| R01 | A reservation must contain 1-31 nights, inclusive. |
| R02 | A reservation must contain 1-12 guests, inclusive. |
| R03 | The base price is $200 per night. |
| R04 | Each valid reservation includes one $50 cleaning fee. |
| R05 | Each guest over five adds $25 per reserved night. |
| R06 | Check-in must be before checkout. |
| R07 | Nights are calculated as checkout date minus check-in date. |
| R08 | The checkout date is not a reserved night. |
| R09 | Every night in the requested reservation must be available. |
| R10 | Invalid reservation information raises `ValueError`. |
| R11 | An unavailable reservation raises `UnavailableError`. |

### Test-data conventions

- Dates use `YYYY-MM-DD` notation, and the selected guest counts are whole numbers. These are conventions for this test data.
- Number of nights is calculated from the two dates; it is not a separate reservation input.
- All calendar dates are available unless a test identifies a specific unavailable date.
- Cases targeting availability use otherwise valid reservation information. The requirements do not define exception precedence when invalid information and unavailability occur together.
- A valid partition in one classification does not by itself make the entire reservation valid. Acceptance also depends on the other requirements.

### Pricing oracle

For a valid, available reservation:

```text
nights = checkout date - check-in date
extra guests = max(number of guests - 5, 0)
base cost = nights * $200
additional guest fee = nights * extra guests * $25
total = base cost + additional guest fee + $50
```

The $50 cleaning fee is added once. The additional guest fee is charged for every guest above five on every reserved night.

For example, check-in on `2026-11-10` and checkout on `2026-11-13` reserve three nights: November 10, 11, and 12. With three guests, the expected total is `(3 * $200) + $50 = $650.00`.

## 2. Equivalence Partition Tables

The partitions within each classification are non-empty and non-overlapping. Separate classifications can describe the same reservation: for example, equal dates also produce zero calculated nights.

### 2.1 Number of Nights

| Partition ID | Partition | Range or condition | Representative | Expected result | Test case |
| --- | --- | --- | --- | --- | --- |
| N01 | Invalid low | Fewer than 1 calculated night | 0 nights: `2026-11-10` to `2026-11-10` | Raise `ValueError`. | TC02 |
| N02 | Valid duration | 1-31 nights, inclusive | 3 nights: `2026-11-10` to `2026-11-13` | Duration is valid; accept if all other requirements are satisfied. | TC01 |
| N03 | Invalid high | More than 31 nights | 32 nights: `2026-11-10` to `2026-12-12` | Raise `ValueError`. | TC04 |

### 2.2 Number of Guests

| Partition ID | Partition | Range or condition | Representative | Expected result | Test case |
| --- | --- | --- | --- | --- | --- |
| G01 | Invalid low | Fewer than 1 guest | 0 guests | Raise `ValueError`. | TC05 |
| G02 | Valid, standard rate | 1-5 guests, inclusive | 5 guests | Guest count is valid; no additional guest fee. | TC06 |
| G03 | Valid, additional fee | 6-12 guests, inclusive | 6 guests | Guest count is valid; add $25 per guest above five, per night. | TC07 |
| G04 | Invalid high | More than 12 guests | 13 guests | Raise `ValueError`. | TC09 |

The valid range is split at five/six guests because the required pricing behavior changes there.

### 2.3 Check-in and Checkout Dates

| Partition ID | Partition | Range or condition | Representative | Expected result | Test case |
| --- | --- | --- | --- | --- | --- |
| D01 | Valid order | Check-in is before checkout | `2026-11-10` to `2026-11-13` | Date ordering is valid; calculate 3 nights. Acceptance also requires a valid duration and other valid information. | TC01 |
| D02 | Invalid, equal dates | Check-in equals checkout | `2026-11-10` to `2026-11-10` | Raise `ValueError`; the calculated duration is 0 nights. | TC02 |
| D03 | Invalid, reversed dates | Check-in is after checkout | `2026-11-10` to `2026-11-09` | Raise `ValueError`; the date difference is negative. | TC03 |

### 2.4 Availability

| Partition ID | Partition | Range or condition | Representative | Expected result | Test case |
| --- | --- | --- | --- | --- | --- |
| A01 | Available reservation | Every requested night is available | November 10, 11, and 12 are available for a November 10-13 reservation. | Accept if all other reservation information is valid. | TC01 |
| A02 | Unavailable reservation | At least one requested night is unavailable | November 11 is unavailable within a November 10-13 reservation. | Raise `UnavailableError`. | TC12 |

An unavailable checkout date belongs to A01 when all actual reserved nights are available, because checkout is excluded. TC13 explicitly checks this rule.

## 3. Test Cases

Each case specifies the dates, guest count, and availability condition. All outcomes below are **expected results** derived from the requirements.

### TC01 - Valid Standard Reservation

- **Classification:** Number of nights; check-in and checkout dates; availability.
- **Partition:** N02 - valid duration; D01 - valid order; A01 - all requested nights available.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `3` guests; all calendar dates available.
- **Expected result:** Accept the reservation for **3 nights**, November 10-12. No additional guest fee applies. Total: `(3 * $200) + $50 = $650.00`.

### TC02 - Zero Nights and Equal Dates

- **Classification:** Number of nights; check-in and checkout dates.
- **Partition:** N01 - invalid low; D02 - equal dates.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-10`; `3` guests; all calendar dates available.
- **Expected result:** Raise **`ValueError`** because check-in is not before checkout and the calculated duration is **0 nights**.

### TC03 - Check-in After Checkout

- **Classification:** Check-in and checkout dates; number of nights.
- **Partition:** D03 - reversed dates; N01 - invalid low.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-09`; `3` guests; all calendar dates available.
- **Expected result:** Raise **`ValueError`**. Check-in occurs after checkout, and the date difference is **-1 day**.

### TC04 - More Than 31 Nights

- **Classification:** Number of nights.
- **Partition:** N03 - invalid high.
- **Input:** Check-in `2026-11-10`; checkout `2026-12-12`; `3` guests; all calendar dates available.
- **Expected result:** Raise **`ValueError`** because the calculated duration is **32 nights**, exceeding the maximum of 31.

### TC05 - Fewer Than One Guest

- **Classification:** Number of guests.
- **Partition:** G01 - invalid low.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `0` guests; all calendar dates available.
- **Expected result:** Raise **`ValueError`** because a reservation must include at least one guest.

### TC06 - Five Guests Without an Additional Fee

- **Classification:** Number of guests.
- **Partition:** G02 - valid, standard rate.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `5` guests; all calendar dates available.
- **Expected result:** Accept the reservation for **3 nights**. Exactly five guests incur no additional guest fee. Total: `(3 * $200) + $50 = $650.00`.

### TC07 - Six Guests With an Additional Fee

- **Classification:** Number of guests.
- **Partition:** G03 - valid, additional fee.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `6` guests; all calendar dates available.
- **Expected result:** Accept the reservation for **3 nights**. One additional guest costs `1 * $25 * 3 = $75.00`. Total: `$600 + $75 + $50 = $725.00`.

### TC08 - Maximum Valid Guests

- **Classification:** Number of guests.
- **Partition:** G03 - valid, additional fee; supplemental upper-boundary check.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `12` guests; all calendar dates available.
- **Expected result:** Accept the reservation for **3 nights**. Seven additional guests cost `7 * $25 * 3 = $525.00`. Total: `$600 + $525 + $50 = $1,175.00`.

### TC09 - More Than 12 Guests

- **Classification:** Number of guests.
- **Partition:** G04 - invalid high.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `13` guests; all calendar dates available.
- **Expected result:** Raise **`ValueError`** because the guest count exceeds 12.

### TC10 - Minimum Valid Nights and Guests

- **Classification:** Number of nights; number of guests.
- **Partition:** N02 - valid duration; G02 - valid, standard rate; supplemental lower-boundary checks.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-11`; `1` guest; all calendar dates available.
- **Expected result:** Accept the reservation for **1 night**, November 10. Total: `(1 * $200) + $50 = $250.00`.

### TC11 - Maximum Valid Nights

- **Classification:** Number of nights.
- **Partition:** N02 - valid duration; supplemental upper-boundary check.
- **Input:** Check-in `2026-11-10`; checkout `2026-12-11`; `3` guests; all calendar dates available.
- **Expected result:** Accept the reservation for **31 nights**, from November 10 through December 10. Add the cleaning fee once. Total: `(31 * $200) + $50 = $6,250.00`.

### TC12 - An Interior Reserved Night Is Unavailable

- **Classification:** Availability.
- **Partition:** A02 - at least one requested night unavailable.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `3` guests; `2026-11-11` unavailable; all other calendar dates available.
- **Expected result:** Raise **`UnavailableError`** because November 11 is a requested night. Availability on the other nights does not satisfy the requirement that every reserved night be available.

### TC13 - Checkout Date Is Unavailable

- **Classification:** Availability; check-in and checkout dates.
- **Partition:** A01 - all requested nights available; D01 - valid order; supplemental checkout-exclusion check.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `3` guests; only `2026-11-13` unavailable; all other calendar dates available.
- **Expected result:** Accept the reservation for **3 nights**, November 10, 11, and 12. November 13 is checkout and is not reserved, so its unavailability does not block the reservation. Total: `(3 * $200) + $50 = $650.00`.

### TC14 - The Last Reserved Night Is Unavailable

- **Classification:** Availability.
- **Partition:** A02 - at least one requested night unavailable; supplemental final-reserved-night check.
- **Input:** Check-in `2026-11-10`; checkout `2026-11-13`; `3` guests; `2026-11-12` unavailable; all other calendar dates available.
- **Expected result:** Raise **`UnavailableError`** because November 12 is the last reserved night. It must be available even though November 13, the checkout date, is excluded.

## 4. Design Coverage and Traceability

| Classification | Identified partitions | Representative test cases | Partitions represented |
| --- | --- | --- | --- |
| Number of nights | N01, N02, N03 | TC02, TC01, TC04 | 3 of 3 |
| Number of guests | G01, G02, G03, G04 | TC05, TC06, TC07, TC09 | 4 of 4 |
| Check-in and checkout dates | D01, D02, D03 | TC01, TC02, TC03 | 3 of 3 |
| Availability | A01, A02 | TC01, TC12 | 2 of 2 |
| **Total** | **12 partitions** | **All represented in the designed cases** | **12 of 12** |

**Planned equivalence partition coverage:** `(12 / 12) * 100 = 100%`.

The fourteen cases include representatives for every identified partition, numeric boundary checks, and focused checks for the last reserved night and the excluded checkout date. A test case can represent partitions from more than one classification; this does not create overlap within an individual classification's partition set.

| Requirement area | Relevant test cases |
| --- | --- |
| Duration limits and date calculations (R01, R06, R07) | TC01-TC04, TC10, TC11 |
| Guest limits (R02) | TC05-TC10 |
| Base nightly price and one-time cleaning fee (R03, R04) | TC01, TC06-TC08, TC10, TC11, TC13 |
| Additional fee for every guest above five, per night (R05) | TC06-TC08 |
| Checkout exclusion (R08) | TC01, TC10, TC13, TC14 |
| Availability and `UnavailableError` (R09, R11) | TC01, TC12-TC14 |
| `ValueError` for invalid information (R10) | TC02-TC05, TC09 |

## 5. Why Equivalence Partitioning Reduces the Number of Tests

Equivalence partitioning groups inputs that are expected to receive the same treatment. For example, guest counts from one through five use the standard rate, while counts from six through twelve require an additional fee. A representative from each group lets the design check each distinct behavior without testing every possible combination of dates, guest counts, and availability calendars.

The shared valid reservation also represents several classifications at once, reducing duplicate cases. Boundary and checkout-specific cases add targeted checks where an incorrect comparison, fee threshold, or date-range endpoint could produce a defect. A passing representative would not prove that every input in its partition works correctly.

## 6. Execution Status

These are **designed test cases with expected outcomes**. No application code was supplied, so application tests have not been executed. The coverage above describes which identified partitions are represented in the design; it is not an execution result or a claim that the application is defect-free.

## Source

The supplied [LP03.md](LP03.md) provides the application requirements and assignment instructions. All expected results in this README are derived from those requirements.
