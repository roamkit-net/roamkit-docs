# Support runbook — Airalo go-live

Goal: support can resolve the full customer journey without engineering for routine cases.

## Find an order / eSIM

1. Ask user for account email (or order / eSIM id if known).
2. Admin / operator: locate `User` → `billing.Account` → `Order` / `Esim`.
3. Confirm ledger debit for the order (billing source of truth).
4. Confirm provider fulfillment id / Airalo reference on the eSIM or order record.
5. Check lifecycle status and recent `EsimLifecycleEvent` / install telemetry.

## Journey checklist (support)

| Step | What to verify | If stuck |
|------|----------------|----------|
| Catalog / purchase | Package visible; order created; balance debited | Billing / catalog sync |
| Provisioning | Provider order succeeded; ICCID / QR present | Airalo sandbox status; retry policy |
| QR delivery | QR / install payload in My eSIM / setup | Resend / regenerate path |
| Installation | Device path (iOS / Android / Desktop) | Guided setup wizard |
| Activation | Status transition per activation policy | Wait / policy banner |
| Data | First usage / connectivity confirmed by user | Roaming checklist; carrier delay |
| Top-up | Top-up order + debit + package extension | Balance; TopupProvider errors |
| Lifecycle | Status matches reality; no illegal downgrade | Escalate if status desync |

## Response target

Pilot KPI: first meaningful support response **&lt; 24 h**.

## Escalation

- Billing / ledger mismatch → engineering (do not hand-edit ledger).
- Provider / Airalo API failures → engineering + note in `api-logs/`.
- P0/P1 during pilot → block Phase 7 until fixed.

## Related

- [e2e-evidence.md](./e2e-evidence.md)
- [rollback.md](./rollback.md)
- [Incident runbook](../incident-runbook.md)
