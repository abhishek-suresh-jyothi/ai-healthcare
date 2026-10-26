# Arogya Karnataka

A privacy-conscious healthcare operations and health-research prototype for Karnataka. The primary research interface is honest about what is—and is not—available: the district directory is seeded, but official indicator data is not; each disease has its own registry entry, but no labelled cohort, evaluated model, clinical score, or validation metric is supplied.

## Runtime and scope

This workspace is implemented for the supplied platform: **Next.js App Router + TypeScript + PostgreSQL + Drizzle ORM**. The requested Python/ML stack is provided as an optional, separate research workflow under `research/`. No Flask/MySQL service is substituted into this runtime: a managed PostgreSQL database and Next.js are the available application environment. `requirements.txt` covers the offline Python research tooling only.

The application is suitable as a **research prototype and synthetic demonstration**, not as a clinical product. Do not enter real patient data or use it for diagnosis, triage, prescriptions, or treatment decisions.

## Implemented modules

- **Role-scoped workspace:** administrator, doctor, nurse, lab technician, pharmacist, receptionist, researcher, and patient demo roles. Sign-in uses Firebase Authentication when configured, otherwise a salted scrypt password hash. Both paths issue an eight-hour, signed `HttpOnly`, `SameSite=Lax` cookie; PostgreSQL remains the source of user roles and hospital scope. Production requires a strong `SESSION_SECRET`.
- **Patient records:** hospital-scoped patient registration; patient-owned record filtering; chart views for medical history, allergies, recorded vitals, consultation notes, lab reports, prescriptions, and referrals. Clinical write routes validate input and log auditable events.
- **Appointments:** schedule/list endpoints with hospital and patient-account scoping.
- **Maternal care:** pregnancy registration, antenatal BP/haemoglobin observations, administrator/doctor-entered review notes, delivery record with one-delivery-per-pregnancy protection, and postnatal mother/newborn follow-ups. No automated maternal risk score is generated.
- **Hospital operations:** transaction-backed bed allocation and discharge; lab report workflow; pharmacy batch stock; billing and full-payment status.
- **Disease research:** independent registry entries for diabetes, heart disease, hypertension, kidney disease, liver disease, anaemia, respiratory disease, and infectious disease. Feature-only research intake is de-identified from patient charts and deliberately returns **no prediction** until a separate, documented, evaluated, deployed model is reviewed.
- **Karnataka districts:** 31 district directory entries, an accessible *schematic* district explorer, data-completeness views, source/period/definition-aware aggregate indicator import, and an admin/researcher-only de-identified CSV export. The import is admin-only and rows remain `pending_review`.
- **Privacy/audit separation:** patient-level clinical data and district aggregate indicators use separate tables. District APIs/exports never query or expose patient records. Feature data is not written to the patient chart.
- **Research toolkit:** independent Python cohort-validation, preprocessing, per-condition model comparison, holdout evaluation, and preparation tests. It produces no artifact until a dataset is supplied.

## Requirements

- Node.js 20.9+ (Node 22 recommended)
- PostgreSQL 14+ with an application database
- Python 3.10+ only if using the optional research tooling

## Local setup

```bash
npm install
cp .env.example .env
```

Edit `.env` for your local PostgreSQL database. Apply the Drizzle schema and provision the sign-in and catalogue data:

```bash
npx drizzle-kit push
npx tsx src/db/seed.ts                       # system, catalogue, and sign-in accounts only
npx tsx src/db/seed.ts --with-clinical-data  # also add synthetic clinical records
```

### Optional Firebase Authentication

To use Firebase for email/password authentication, create a Firebase project, enable **Authentication → Sign-in method → Email/Password**, and create a Firebase Auth account for each active PostgreSQL user who should be able to sign in. Verify each Firebase email address. Set the project's Web API key as `NEXT_PUBLIC_FIREBASE_API_KEY` in `.env` before starting the dev server or building the deployment. When this value is set, `/api/auth/login` checks the credentials with Firebase and requires a verified email; without it, the existing PostgreSQL password authentication remains active.

Firebase accounts are matched to active PostgreSQL users by email. PostgreSQL continues to provide the account's role, hospital scope, and application data; Firebase does not store the application's clinical records. The Firebase Web API key is public configuration, not a server secret.

By default the seeder creates **no clinical records**. Set `DEMO_CLINICAL_DATA=true` in `.env` (or pass `--with-clinical-data`) to also add synthetic patients, appointments, laboratory reports, bills, pharmacy stock batches, and maternal records.

Start the application:

```bash
npm run dev
```

Visit `http://localhost:3000`. `/api/health` checks the database connection. On the managed project platform, use the platform preview runtime rather than running a persistent `next start` process yourself.

### Demonstration accounts

The Drizzle seed script creates the following accounts at `@arogya.local`:

| Account | Email | Role |
|---|---|---|
| Administrator | `admin.demo@arogya.local` | admin |
| Doctor | `doctor.demo@arogya.local` | doctor |
| Nurse | `nurse.demo@arogya.local` | nurse |
| Laboratory | `lab.demo@arogya.local` | lab technician |
| Pharmacy | `pharmacy.demo@arogya.local` | pharmacist |
| Reception | `reception.demo@arogya.local` | receptionist |
| Research | `research.demo@arogya.local` | researcher |
| Patient | `patient.demo@arogya.local` | patient (linked to that account's own synthetic patient record) |

All eight accounts share `DEMO_PASSWORD` from `.env`. When it is unset (or shorter than 4 characters), the **development-only** default `ArogyaDemo#2026` is used instead and a warning is logged.

**Sign-in troubleshooting.** Demo data is provisioned automatically: the first sign-in attempt (and `/api/health`) runs an idempotent bootstrap that creates the accounts and *re-synchronises every demo password to the currently configured `DEMO_PASSWORD`*. So after changing `DEMO_PASSWORD` in `.env`, simply sign in again with the new value — no re-seeding or restart is needed. If sign-in still fails, check the exact value in `.env` (anything shorter than 4 characters is ignored in favour of the default) and use the “Use demo access” link on the sign-in screen, which fills the credential this deployment actually published. The sign-in screen also shows the shared demo account email and password whenever `DEMO_PASSWORD` is configured.

Two deployment notes:

- `SESSION_SECRET` should be a random value of at least 32 characters. If it is missing, sign-in still works instead of failing with a server error, but the signing key is then derived from `DATABASE_URL`, which is not a real secret. Set `SESSION_SECRET` before any shared use.
- The session cookie is marked `Secure` only when the request genuinely arrives over HTTPS, so sign-in works on an HTTPS preview and on a local HTTP server alike.

Running `npx tsx src/db/seed.ts` explicitly is optional; it performs the same idempotent seeding once from the command line. Use a unique secret and password in development; **never seed these accounts into production**.

## Keeping the clinical database empty

`DEMO_CLINICAL_DATA` controls whether synthetic clinical records exist at all. When it is `false` (the default), the automatic bootstrap provisions only what is needed to sign in and use the application — the hospital, RBAC roles and permissions, departments, demo accounts, the 31-entry district directory, the untrained model registry, the bed register, and the drug and laboratory test catalogues. It creates **no** patients, appointments, consultations, vital signs, medical history, allergies, laboratory reports, prescriptions, admissions, discharges, referrals, pregnancy/antenatal/delivery/postnatal records, bills, payments, stock batches, or research intakes.

This separation is deliberate: reference and catalogue data is not patient data. An emptied database therefore stays empty of clinical records, while the application remains usable and sign-in keeps working.

To empty every table (all rows removed, identity sequences reset):

```bash
psql "$DATABASE_URL" -c "DO \$\$ DECLARE stmt text; BEGIN
  SELECT string_agg(format('%I.%I', schemaname, tablename), ', ') INTO stmt
  FROM pg_tables WHERE schemaname = 'public';
  EXECUTE 'TRUNCATE TABLE ' || stmt || ' RESTART IDENTITY CASCADE';
END \$\$;"
```

Keep `DEMO_CLINICAL_DATA=false` if you want it to stay empty; the system and sign-in data will be quietly re-provisioned on the next sign-in or health check, but no clinical records will reappear. To restore the synthetic demonstration records, set `DEMO_CLINICAL_DATA=true` and re-run `npx tsx src/db/seed.ts --with-clinical-data`.

To start again from scratch, drop and recreate the schema and re-push:

```bash
psql "$DATABASE_URL" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
npx drizzle-kit push
npx tsx src/db/seed.ts
```

## Database and API

`src/db/schema.ts` declares normalized PostgreSQL entities for hospitals, departments, roles, permissions, users, doctors, staff, patients, history, allergies, appointments, consultations, vitals, diseases, dataset provenance, model versions and research intakes, tests and reports, prescriptions and medications, inventory, beds, admissions/discharges, pregnancy/antenatal/delivery/postnatal records, referrals, district indicators, billing/payments, and audit logs. Foreign keys and indexes encode ownership and lookup paths. See [`ERD.md`](ERD.md).

REST endpoints are under `src/app/api/`:

- `/api/auth/{login,logout,me}`
- `/api/dashboard`, `/api/patients`, `/api/patients/:id`, `/api/appointments`, `/api/maternal`
- `/api/operations` for inpatient, laboratory, pharmacy, billing, and payment actions
- `/api/research` for the condition registry and feature-only intake
- `/api/districts`, `/api/districts/import`, `/api/reports/districts`
- `/api/health`

Application authorization is enforced in route handlers and queries. The project includes an RBAC catalogue, but does **not** claim PostgreSQL row-level security is configured. A production multi-hospital deployment must additionally review database-level row policies, organisation isolation, CSRF/origin policy, rate limiting, MFA, key rotation, central log retention, encrypted backup/restore drills, and privacy/clinical governance. It has no medical-file upload endpoint; protected document storage must be separately designed rather than exposing a public file path.

For a local encrypted backup workflow, configure a protected backup destination and retention policy. For example, `pg_dump --format=custom --no-owner "$DATABASE_URL" > backup.dump` creates a database dump; protect it with approved encryption, access controls, rotation, and restore tests before storing it. Do not commit backup files.

## Research integrity and official district sources

The seeded district table contains **names only**. The district-indicator table is empty until an administrator imports a documented, aggregate source. NFHS-5 is a representative household survey (2019–21); it is not a universal disease registry, hospital dataset, or an individual-patient cohort. It may support only the indicator definitions and geographic coverage of the selected factsheet. The research view says “no values loaded” rather than showing invented figures. The map is labelled schematic, and its points are not official boundaries or geocoded measurements.

The import API (`POST /api/districts/import`) accepts up to 1,000 aggregate JSON records. Each must include an existing Karnataka district code, indicator code/name/value/unit, start/end dates, source name/URL, and a population/numerator/denominator definition and limitations. Denominators are optional only when the source legitimately has none. Imports are saved with `qualityStatus = pending_review`; the import route never marks data verified. No patient identifiers, household microdata, or records belong in this interface. Downloaded reports contain only those district-level indicator records.

Official NFHS publication entry point: [IIPS/RCHIIPS National Family Health Survey](https://rchiips.org/NFHS/index.shtml/). Verify the exact factsheet, version, indicator definition, period, and permitted use before import. NFHS-4/NFHS-5 time comparisons require harmonised definitions and coverage; a time-series chart is intentionally not populated without reviewed multi-period values.

## Machine-learning research workflow

Install the optional environment and run its tests:

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\\Scripts\\Activate.ps1
python -m pip install -r requirements.txt
pytest -q research/tests
```

The commands in [`research/README.md`](research/README.md) document two steps:

1. `research/prepare_dataset.py` validates a documented, de-identified condition-specific CSV, explicit binary labels, feature names, missingness, and class balance; it writes a manifest and quality report.
2. `research/train_model.py` runs *one condition per invocation*, compares logistic regression and random forest (optionally XGBoost), selects on validation average precision, and reports a held-out test set with sensitivity, specificity, precision, recall, F1, ROC-AUC, PR-AUC, Brier score/reliability bins, and descriptive permutation importance.

These scripts are scaffolding; you must provide permitted data, population/label definitions, and clinical/data-governance review. No real datasets, trained artifacts, or accuracy claims ship in this repository. A random split may be inappropriate when patients/sites/time periods overlap. Holdout scores and a 0.5 threshold are not independent clinical validation or a clinical action threshold. SHAP is listed as an optional dependency but no explanations are displayed until a specific model and explanation method are assessed.

The web API currently records only explicitly non-identifying feature snapshots for a supported condition and returns `{ prediction: null, status: "model_unavailable" }`. It cannot read Python artifacts and does not pretend a trained model is available.

## Validation and tests

```bash
npx next typegen
npm exec tsc -- --noEmit --pretty false
npm run build
pytest -q research/tests
```

The managed application platform checks `/api/health` when building and starting the preview. Python tests are separate from the web application build.

## Limitations before real deployment

- No clinical model, labelled disease dataset, diagnostic thresholds, published metrics, SHAP explanation, or clinical validation is included.
- No NFHS-5 indicators are preloaded; only district names are seeded.
- No file/document uploads, patient-facing registration, password-reset workflow, email/SMS integration, external payment gateway, insurance claims, or automated database-backup scheduler is implemented.
- The runtime database is PostgreSQL, not MySQL; charts are rendered with native accessible UI/SVG, not Chart.js or Leaflet. The displayed map is a schematic directory, not geospatial boundaries.
- Real-patient use requires a separately security-reviewed deployment, approved data, clinical governance, applicable legal/privacy controls, validated model-specific infrastructure, and qualified health professionals.
