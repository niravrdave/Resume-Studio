# CareerTax AI

**Build a Better Resume. Plan a Smarter Salary.**

## Stack and substitutions
| Asked for | What we used | Why |
|---|---|---|
| React/Next.js + TypeScript | React 19 (CRA + craco), JavaScript, Tailwind, shadcn/ui | This is the stack the Emergent template supports |
| PostgreSQL | MongoDB (Motor) | The platform gives each app a managed MongoDB (`MONGO_URL`/`DB_NAME`). Collections are listed below; no migration step is needed because seeding is idempotent and runs at startup |
| Managed auth + Google OAuth | Custom JWT (bcrypt, httpOnly cookies, refresh tokens) + Emergent-managed Google sign-in | |
| Configurable LLM | Google Gemini through the Emergent universal key (`LLM_PROVIDER`, `LLM_MODEL`) | |
| Email provider | Resend, managed by Emergent (`EMERGENT_EMAIL_KEY`, `EMAIL_FROM_NAME`) | |
| Excel | openpyxl, with real formulas, named ranges, protected formula cells and `fullCalcOnLoad` | |

## Backend modules (`/app/backend`)
- `core.py`: DB, auth dependencies, rate limiting, audit log, Excel-style rounding
- `auth_routes.py`: register, email verification, login, logout, refresh, password reset, Google session, profile, data export, account deletion
- `resume_service.py` / `resume_routes.py`: file extraction (PDF/DOCX/TXT with validation), deterministic keyword and structure scoring, LLM prompts that resist prompt injection, enhanced resume versions, a check for claims not supported by the original, cover letter and application email
- `payroll.py`: CTC breakup (Mode A/B) and the April–March projection with overrides. Kept separate from tax
- `tax_rules.py` / `tax_engine.py`: versioned rules per FY and a deterministic Old/New regime engine (87A rebate with marginal relief, surcharge marginal relief, cess, rounding under 288A/288B, break-even, planning suggestions)
- `excel_builder.py`: 7-sheet formula workbook
- `exports.py`: DOCX/PDF builders
- `records_routes.py`: exports, email preview/send, documents, calculations, presets, dashboard
- `admin_routes.py`: tax-rule editor with versions and audit, ATS rubric versions, users, logs, settings

### Collections
users, user_sessions, email_tokens (TTL), login_attempts, resume_records (TTL for unsaved), uploads, calculations (with rules_snapshot), presets, tax_rules, tax_rule_history, rubrics, settings, usage, email_logs, audit_logs, error_logs.

## Tax-rule sources and versioning
| FY | Status | Basis |
|---|---|---|
| 2025-26 | Enacted | Income-tax Act 1961 as amended by Finance Act 2025. New regime slabs 0/4/8/12/16/20/24L; standard deduction ₹75,000; 87A up to ₹12L, max ₹60,000, with marginal relief. Old regime: standard deduction ₹50,000; 87A ₹5L / ₹12,500 |
| 2026-27 | Officially Published | Income-tax Act 2025 (in force from 1-Apr-2026); Budget 2026 / Finance Act 2026 kept the same slabs. The mapping of old section numbers to the new Act is marked *Pending Verification* |
| 2027-28, 2028-29 | Illustrative Assumption | No Finance Act exists yet. The app labels these years Provisional and lets users edit the assumptions |

Sources: https://www.incometaxindia.gov.in/w/tax-rates , memo-2026.pdf, the Income-tax Act 2025 PDF, and the e-Filing portal. Each admin save creates `FY.vN` in `tax_rule_history` and an audit entry. Saved calculations keep their `rule_version` and a snapshot of the rules.

## Testing
The Excel workbook was recalculated with the Python `formulas` engine, and its values match the backend engine. Tax spot checks: ₹12.85L gross under the new regime gives ₹10,400; ₹51L under the new regime gives ₹11,49,200 after surcharge marginal relief. Test reports are in `/app/test_reports`.

## Deployment
Use Emergent deploy. Set every variable in `backend/.env.example`. HTTPS is terminated at the ingress, and cookies are `Secure; SameSite=None`.

## Known limitations
- Capital gains and other special-rate income, perquisite valuation, and the ₹7.5L limit on employer contributions are not modelled
- Scanned or image-only PDFs are rejected because there is no OCR
- Documents are emailed only to the user's verified account email. This prevents the app being used as an open relay. To send to a recruiter, download the files and forward them
- Rate limiting is in-memory and per process
- TDS is spread evenly across months. Real employer payroll may adjust it month by month
