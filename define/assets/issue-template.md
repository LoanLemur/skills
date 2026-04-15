## Problem

{2-4 bullets. One takeaway each. What's broken, missing, or painful.
Name actors and the gap. No prose paragraphs.}

## Goal

{One bullet. What the system can do after this ships that it can't today.}

## Design

### How it works

{One bold actor header per actor. Under each, 3-6 bullets: action →
outcome. Short sentences. Name states, transitions, notifications.}

Example shape:

**MERCHANT**
- Clicks "Request payment" on customer profile
- Fills modal: amount, memo, optional due date
- Submits → SMS sent, request appears in Sent

**CUSTOMER**
- Taps SMS link, enters card, confirms
- Merchant notified, request marked Paid

### Key decisions

{One bullet per decision. Format: decision, then rationale, then
rejected alternatives on an indented line. Blank line between decisions.}

Example shape:

- Stripe Checkout, not custom. PCI scope small, 3DS free.
  Rejected Elements (still PCI), custom (compliance cost).

### Constraints

{One bullet each. External API limits, business rules, prior failures,
dependencies. Omit section if none.}

### Edge cases

{One bullet each. Format: `trigger → outcome`. Omit section if none.}

Example shape:

- Zero amount → form blocks submit, "Amount must be > 0". No record.
- SMS delivery fails → request stays Pending, merchant sees delivery error.

## Codebase Context

{2-4 bullets. Adjacent patterns, constraints, conventions found in
Phase 2. Not file paths. Omit section if Phase 2 found nothing notable.}

## Behavior Inventory

{Numbered list. Each item names a specific trigger, state change, and
outcome. Deferred items marked with cross-reference.}

1. {Behavior one}
2. {Behavior two}
3. {Behavior three}

## Outstanding Questions

{One bullet each. Omit section if none.}
