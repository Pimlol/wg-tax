# Buyer-Side Tax Declaration

**Status:** Draft for working group review, revision 2
**Companion to:** PR #4, *Tax Jurisdiction Discovery, Settle-Only Provenance & SCITT Audit Receipts* (@whawk46 / Corrente Labs)
**Scope:** Adds buyer-side qualification. Does not modify settle-only accounting, SCITT registration, or EIP-3009 event derivation.

---

## Abstract

PR #4 lets a resource server advertise **its own** tax jurisdiction and Merchant-of-Record status. For several tax systems, that is not sufficient to determine the applicable treatment, because the treatment is a function of the **buyer**, not the seller.

This companion specifies:

1. A **buyer-signed tax declaration**, carried in a dedicated header, signed by the same key that signs the EIP-3009 authorization.
2. A **verification model** that never blocks the critical path and never converts an upstream outage into a negative answer.
3. A **qualification model** separating the treatment applied from the evidence it rests on, with an explicit default that leaves no unqualified exposure.
4. A **multi-axis tax result**, jurisdiction-scoped, replacing the scalar regime enumeration of revision 1.
5. Additional **receipt fields** and an **aggregation key**, all unrecoverable after the fact.

### What changed in revision 2

Revision 1 proposed a single `taxRegime` scalar with seven values. That was wrong, and it failed in the same way as the `rateBasisPoints` scalar this proposal originally objected to: it collapsed independent facts into one field.

Two real cases proved it, and both are documented in §5.1. The correction is a small set of independent axes plus normative invariants, not a longer enumeration.

The buyer declaration itself (§2), the verification model (§3), and the fallback ladder (§4) are unchanged.

---

## 0. Normative scope

This document standardises **the shape of the tax result exchanged**, so that different resolvers can produce and consume the same contract.

It does not standardise:

- computation of tax amounts;
- invoice generation or invoice wording;
- execution or filing of any periodic tax return;
- any particular tax engine, rules provider or data source.

**Rationale.** A specification carrying the substantive tax rules of 27 member states would be stale on publication and would make one implementation normative in practice. Standardising the result instead of the reasoning lets an in-house engine, an open-source library and a commercial provider interoperate, none of them being a dependency of the standard.

Retention of verified declarations and receipts is the seller's obligation under its own national rules and is out of scope here.

---

## 1. Problem statement

Under EU VAT rules for electronically supplied services, the applicable treatment depends on facts about the **buyer**: whether they are a taxable person, and in which member state they are established. The seller's own establishment determines almost nothing.

A seller advertising `jur=FR` still cannot decide between:

| Buyer | Treatment |
|---|---|
| Taxable person, another member state | Reverse charge; place of supply in the buyer's state |
| Taxable person, outside the EU | Not due in the EU |
| Non-taxable person, another member state | Buyer's national rate, above threshold or on option |
| Non-taxable person, another member state | Seller's national rate, below threshold |
| Unknown | No defensible position |

x402 is designed so that the seller learns nothing about the buyer. A wallet address is neither a taxable person nor a jurisdiction. This is a deliberate protocol property, and it is precisely what makes the seller's obligation unsatisfiable without an explicit declaration.

---

## 2. The buyer tax declaration

*(unchanged from revision 1)*

### 2.1 Structure

```json
{
  "version": "x402-tax-1",
  "jurisdiction": "DE",
  "taxableStatus": "TAXABLE_PERSON",
  "taxId": "DE123456789",
  "validUntil": 1790000000000,
  "signature": "0x…"
}
```

| Field | Type | Notes |
|---|---|---|
| `jurisdiction` | ISO 3166-1 alpha-2 | Place of establishment claimed by the buyer |
| `taxableStatus` | `TAXABLE_PERSON` \| `NON_TAXABLE` | Closed enumeration |
| `taxId` | string, optional | Required when `taxableStatus` is `TAXABLE_PERSON` and the jurisdiction issues one |
| `validUntil` | integer, ms | Buyer-set expiry; the seller MAY cache no longer than this |
| `signature` | EIP-712 | Over the other fields |

### 2.2 Signature and binding

The declaration MUST be signed with the **same key that signs the EIP-3009 payment authorization**. This binds the declaration to the payer without introducing any new identity primitive, requires no cryptography beyond what x402 already uses, and shifts the burden of the claim to the party making it.

The declaration does not prove that the signer is the legal person named by `taxId`. It establishes that the payer asserted it, under signature, at a given time. That is the same evidentiary position a seller occupies today when a customer supplies a VAT number.

### 2.3 Transport

```
X-Tax-Declaration: <base64url(JSON)>
```

Seller-only. It MUST NOT be placed in `resource_url`, `description`, `reason`, or any field that transits a facilitator: those reach third parties in cleartext, and a tax identifier has no reason to be there. The header name is left to the working group.

The declaration is **optional**. A request without one is served at the default treatment of §4.

---

## 3. Verification

*(unchanged from revision 1)*

### 3.1 An upstream outage MUST NOT produce a negative answer

When the competent authority does not respond, the result is `NOT_VERIFIABLE`, never `INVALID`. These have opposite consequences: one leads to a conservative default, the other to refusing a legitimate counterparty.

This is not a hypothetical. VIES is a federated query engine over national registers, so any single member state can be unavailable independently of the others. Partial unavailability is the normal case, not the exception.

### 3.2 Verification happens once and is cached

Cached against `(payer address, taxId)`, bounded by `validUntil`. Subsequent requests from the same payer within that window MUST NOT trigger a new upstream call.

This satisfies the design principle that extensions should not impose additional round-trips outside the normal client/server flow. Ten thousand requests produce one verification.

### 3.3 Re-verification is asynchronous

On expiry the entry degrades to `NOT_VERIFIABLE` and re-verification is attempted out of band. The request in flight never waits.

---

## 4. Qualification

### 4.1 Three distinct facts

Revision 1 under-specified this. The following are separate and MUST NOT be conflated:

| Fact | Source |
|---|---|
| Status declared by the buyer | The signed declaration |
| Whether the buyer acts as a taxable person for this supply | The declaration, per supply |
| Whether the identifier verified | The competent authority |

**A failed verification does not make a professional a consumer.** It affects the evidence, not the status.

This has a concrete and common cause. A French identifier is derivable from the SIREN, but it is **not active by default** for a business under a domestic franchise regime: activation must be requested from the tax office. A buyer under such a regime can therefore supply a structurally valid number in good faith that VIES will reject. Declared status accurate, verification failed.

### 4.2 Two fields, not one

- `taxRegime` — replaced in revision 2 by the multi-axis result of §5
- `qualificationBasis` — what the result rests on:

| Value | Meaning |
|---|---|
| `DECLARED_VERIFIED` | Declaration present, verified against the competent authority |
| `DECLARED_UNVERIFIED` | Declaration present, verification unavailable or expired |
| `INFERRED` | No declaration; location established from other evidence |
| `NONE` | No declaration, no evidence |

A conservative default is not a treatment. It is the ordinary domestic treatment applied on `NONE`. Keeping these apart lets a seller measure what share of revenue rests on weak qualification, which is the figure both an auditor and a finance function want.

### 4.3 Fallback ladder

| Buyer supplies | Outcome |
|---|---|
| Verified identifier, another member state | Reverse charge |
| Evidence of non-EU establishment | Not due in the EU |
| Two non-contradictory location items | Buyer's member state, seller collects |
| Nothing, or unverifiable | Full domestic rate |

The fourth rung is the safety valve. It ensures no request creates uncovered exposure, which is what lets a compliant seller keep serving fully anonymous agents rather than turning them away.

### 4.4 Pricing as an incentive

A seller who cannot qualify the buyer must provision the worst case, and that provision is real money. The `402` quote MAY differ by qualification level, the higher price applying to `NONE`.

This passes through a cost the buyer creates and converts identification from an obligation imposed by the seller into an economic choice made by the buyer.

---

## 5. The tax result

### 5.1 Why a scalar cannot work

Revision 1 proposed seven mutually exclusive values. Two real cases break it, and both come from the same corner of EU law.

**Case one.** A French supplier under a domestic franchise regime supplies an EU taxable person. Place of supply is the customer's member state and the customer is liable under Article 196 of the VAT Directive: the supplier's franchise does not prevent the reverse charge. So `supplierScheme` and `mechanism` are independent. Revision 1's `SMALL_BUSINESS` fused them and destroyed the place-of-supply information.

**Case two.** Since Directive 2020/285 took effect in 2025, the cross-border SME exemption is not restricted to non-taxable customers. So `supplierScheme` and `buyerStatus` are independent too.

Neither case can be expressed by any of the seven values. The failure is structural, not a missing entry.

### 5.2 Axes

| Field | Values |
|---|---|
| `placeOfSupplyJurisdiction` | ISO 3166-1 alpha-2, the **actual** jurisdiction (`DE`, `FR`, `US`) |
| `placeOfSupplyRule` | `B2B_GENERAL`, `B2C_ELECTRONIC`, `SPECIAL` |
| `taxTreatment` | `TAXABLE`, `EXEMPT`, `NOT_DUE_IN_EU` |
| `liableParty` | `SUPPLIER`, `CUSTOMER`, `NONE` |
| `mechanism` | `SUPPLIER_COLLECTION`, `REVERSE_CHARGE`, `NONE` |
| `supplierScheme` | `STANDARD`, `SME_DOMESTIC`, `SME_CROSS_BORDER` |
| `exemptionLegalBasis` | string, `null` unless `taxTreatment` is `EXEMPT` |

Two naming choices are deliberate.

**Actual jurisdiction codes, not relative ones.** Revision 1 used `SUPPLIER_JURISDICTION` and `CUSTOMER_JURISDICTION`. That records which rule produced the answer, not where the supply is taxable. Downstream consumers need the second.

**`NOT_DUE_IN_EU`, not `OUT_OF_SCOPE`.** A non-EU customer means EU VAT is not due. It does not mean no indirect tax exists: a sales tax, GST or foreign VAT may apply in the customer's jurisdiction. The `eu-vat` vocabulary scope makes this explicit rather than implying a global conclusion.

### 5.3 Invariants

Normative and mechanically checkable. A result violating any of these MUST be rejected.

```
IF   mechanism = REVERSE_CHARGE
THEN liableParty = CUSTOMER

IF   mechanism = REVERSE_CHARGE
THEN buyerStatus = TAXABLE_PERSON

IF   taxTreatment = EXEMPT
THEN exemptionLegalBasis IS NOT NULL

IF   taxTreatment = NOT_DUE_IN_EU
THEN liableParty = NONE AND mechanism = NONE
```

Invariants are what keeps the closed-vocabulary discipline once the scalar is gone. Without them, a multi-axis result can express combinations that do not exist in law.

### 5.4 Worked examples

**EU taxable person, standard supplier**

```yaml
placeOfSupplyJurisdiction: DE
placeOfSupplyRule: B2B_GENERAL
taxTreatment: TAXABLE
liableParty: CUSTOMER
mechanism: REVERSE_CHARGE
supplierScheme: STANDARD
qualificationBasis: DECLARED_VERIFIED
```

**EU taxable person, supplier under a domestic franchise** — case one of §5.1

```yaml
placeOfSupplyJurisdiction: DE
placeOfSupplyRule: B2B_GENERAL
taxTreatment: TAXABLE
liableParty: CUSTOMER
mechanism: REVERSE_CHARGE
supplierScheme: SME_DOMESTIC
qualificationBasis: DECLARED_VERIFIED
```

The supplier issues no VAT. The customer self-assesses and may deduct that same self-assessed amount subject to its own right of deduction. No VAT is invoiced by the supplier, so none is deductible from the invoice; the deduction arises from the reverse charge itself.

**Supplier under an active cross-border SME exemption in the place of supply**

```yaml
placeOfSupplyJurisdiction: DE
placeOfSupplyRule: B2B_GENERAL
taxTreatment: EXEMPT
liableParty: NONE
mechanism: NONE
supplierScheme: SME_CROSS_BORDER
exemptionLegalBasis: "directive-2020-285"
qualificationBasis: DECLARED_VERIFIED
```

No VAT arises, so there is nothing for the customer to self-assess and nothing to deduct.

**Undeclared buyer**

```yaml
placeOfSupplyJurisdiction: FR
placeOfSupplyRule: B2C_ELECTRONIC
taxTreatment: TAXABLE
liableParty: SUPPLIER
mechanism: SUPPLIER_COLLECTION
supplierScheme: STANDARD
qualificationBasis: NONE
```

---

## 6. Receipt fields

### 6.1 Settlement asset

```json
"settlementAsset": "eip155:8453/erc20:0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
```

The same nominal amount settled in USDC and in EURC produces different taxable bases in EUR. A receipt recording only an amount and a currency name is ambiguous once multiple assets are in use.

### 6.2 Exchange rate, frozen and dated

```json
"paymentCurrency": "USDC",
"taxCurrency": "EUR",
"exchangeRate": "0.9187",
"exchangeRateSource": "ecb-daily",
"exchangeRateTimestampMs": 1787806800000
```

Fixed at the chargeable event, never recomputed. A receipt whose amounts move between two exports cannot be reconciled and cannot be defended under audit.

`exchangeRateSource` MUST come from a closed vocabulary. An auditor cannot act on a source each implementer names differently. For EU VAT the reference is the rate at the time of chargeability; other jurisdictions are for their own vocabulary to define.

### 6.3 `null` is not zero

Where no rate is determined or applied by the supplier, the rate field MUST be `null`. `0` is reserved for an actual legal zero rate.

Reverse charge, exemption, an SME regime and non-EU supplies are not a legal 0%. Writing `0` leads a downstream consumer to sum lines of different natures and produce a false total.

### 6.4 Three timestamps

| Field | Meaning |
|---|---|
| `paymentTimestamp` | Settlement confirmation, chain-timestamped |
| `supplyTimestamp` | When the service was actually supplied |
| `taxPoint` | Chargeable event under the applicable jurisdiction |

In x402 these almost always coincide to the second. They are nonetheless three different notions, and a jurisdiction may separate them. Merging them into one field makes the receipt unusable in that case.

### 6.5 Result fields

`placeOfSupplyJurisdiction`, `taxTreatment`, `liableParty`, `mechanism`, `supplierScheme` and `qualificationBasis` belong in the receipt as well as in the quote. The receipt is what an audit examines, and the result is what an audit questions.

---

## 7. Invoice aggregation

*(unchanged from revision 1)*

SCITT receipts provide an immutable micro-audit trail. They do not discharge an invoicing obligation.

The aggregation key MUST be `(declaration identity, period, result)`. It MUST NOT be the payer address alone: a buyer can change tax identity mid-period, and the correct output is then two invoices split at the point of change, not one under whichever identity happened to be current at generation time.

Corrections MUST be issued as new entries referencing the original, never as modifications.

---

## 8. Open questions

### For the working group

1. **Canonical profiles.** Should recurrent combinations carry stable identifiers (for example `EU_B2B_REVERSE_CHARGE`), and if so are they normative values, normative aliases for field combinations, or conformance test vectors only?
2. **Minimal core.** Which fields are genuinely required for x402 interoperability, and which belong in optional extensions?
3. **Ruleset governance.** If results reference a versioned jurisdiction ruleset, who publishes those identifiers and under what authority? Interoperability cannot rest on identifiers no one maintains, and it must not rest on a proprietary one either.
4. **Evidence level.** What proof should accompany a result produced by a third-party resolver?
5. **Consumer location evidence.** EU rules require non-contradictory items of evidence to locate a non-taxable buyer. A wallet supplies none natively. Either `INFERRED` is not served and those cases fall to the default, or the specification must define evidence the protocol does not carry. I lean toward the former.
6. **Legal weight of a wallet-signed declaration.** A signature binds a key to a statement, not a legal person to a statement. Whether a tax authority accepts it as evidence of the seller's good faith is untested and needs practitioner input rather than protocol design.
7. **Non-EU seller, EU buyer.** Out of scope of this draft. Flagged because it will be raised.
8. **Merchant of Record.** Where an MoR is interposed, its establishment governs rather than the underlying provider's. Which party owns the result is unresolved.

### Pending professional validation

The exact invoice wording for a supplier under a domestic franchise regime supplying an EU taxable person is not settled. Both a franchise mention and a reverse-charge mention appear to be due, and available sources are not firm on whether both are required. This is a downstream documentary concern, outside the normative core of §0.

---

## 9. Non-goals

- No change to settle-only accounting, SCITT registration, or EIP-3009 event derivation. This companion adds to PR #4; it does not modify its existing pillars.
- No member-state-specific rates, thresholds, filing formats or invoice wording. The mechanism is specified; national particulars are not.
- No reference implementation in this document.
- **Sourcing and legal status.** The treatments described in §5 rest on primary sources: Article 196 of the VAT Directive for the reverse charge, Article 286 ter of the French tax code for the identification requirement of a supplier under a domestic franchise regime, Directive 2020/285 for the cross-border SME exemption, and BOFiP §140 for the scope of the French recapitulative statement for services. This draft is nonetheless written from operating experience with EU VAT filing and from documentary research, not from a tax advisory practice. It is not tax advice, and the axes and invariants of §5 should be reviewed by a qualified practitioner before adoption.
