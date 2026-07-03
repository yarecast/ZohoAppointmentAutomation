# Multi-Branch Medical Appointment Automation System

A CRM and integration workflow built on Zoho CRM, Zoho Creator, and Zoho Flow. Simulates the appointment intake and patient lifecycle process for a three-branch clinic chain (Mexico City, Guadalajara, Monterrey).

Portfolio project. All patient data is synthetic.

## Problem

Multi-branch clinics typically run intake through a form or call log that is disconnected from the CRM the staff actually works from. Every request needs to be re-typed into the CRM by hand, which delays follow-up and creates duplicate patient records when the same person books at more than one branch.

## What this does

A patient appointment request submitted through a Zoho Creator form is automatically synced into Zoho CRM as a de-duplicated Contact and a pipeline-tracked Deal, with no manual data entry.

- New patient → new Contact created, status `Lead`
- Returning patient (matched by email) → existing Contact reused, no duplicate
- Every request → a Deal created and placed on the Medical Appointment Pipeline, starting at `Appointment Booked`

## Architecture

```
Zoho Creator (form)
      |
      v  Record Created trigger
Zoho Flow
      |
      |-- Fetch Contact by Email (Zoho CRM)
      |-- If Entry Id is null:
      |       TRUE  -> Create Contact (Patient_Status = Lead)
      |       FALSE -> reuse existing Contact
      |-- Create Deal, linked to Contact, Stage = Appointment Booked
      v
Zoho CRM (Contacts + Deals + Medical Appointment Pipeline)
```

Each layer has one job: Creator captures data, Flow owns the integration and deduplication logic, CRM is the system of record for both the patient relationship and the appointment lifecycle.

## Zoho CRM

**Contacts — custom fields**

| Field | Type | Purpose |
|---|---|---|
| `Patient_ID` | Text | Unique patient identifier across systems |
| `Home_Branch` | Picklist: `Branch_A_CDMX`, `Branch_B_GDL`, `Branch_C_MTY` | Patient's primary clinic |
| `Patient_Status` | Picklist: `Lead`, `Active`, `Inactive` | Relationship lifecycle, independent of any single visit |
| `Insurance_Provider` | Text | Payer captured at intake |

**Deals — custom fields**

| Field | Type | Purpose |
|---|---|---|
| `Appointment_Date` | Date/Time | Scheduled visit |
| `Specialty` | Text/Picklist | Requested specialty |
| `Branch` | Picklist | Branch for the appointment |
| `Creator_Record_ID` | Text | Back-reference to the originating Creator entry |
| `Appointment_Status` | Picklist | Operational status (Scheduled, Completed, etc.) |

**Medical Appointment Pipeline**

| Stage | Probability | Meaning |
|---|---|---|
| Appointment Booked | 20% | Request received, not yet confirmed |
| Appointment Confirmed | 40% | Patient confirmed attendance |
| Visit Completed | 80% | Patient was seen |
| Active Patient | 100% (Won) | Patient retained for ongoing care |
| Cancelled/No-Show | 0% (Lost) | Appointment did not result in a visit |

## Zoho Creator

App: `ClinicAppointmentSystem` — form: `Appointment_Request`

Fields: patient name, email, phone, date of birth, appointment date, specialty, branch, reason for visit, insurance provider, referring doctor, plus `Patient_ID` and `CRM_Contact_ID` for cross-system traceability and `Created_By_Branch` to track which front desk logged the request.

An earlier version of the dedup/sync logic was built directly in Creator using Deluge (`OnAdd_PatientDeduplication_CRMSync`, included in `/creator` for reference). It was abandoned after repeated instability in Creator's newer Deluge workflow editor, and the logic was rebuilt in Zoho Flow instead, which handles the branching and cross-app calls more reliably.

## Zoho Flow

Flow: `Creator_To_CRM_Appointment_Sync` — trigger: record created on `Appointment_Request`

1. Fetch Contact in CRM by email
2. If/else on whether an Entry Id was found
3. True branch: create Contact (`Patient_Status = Lead`, `Lead_Source = Zoho Creator - Appointment`)
4. Create Deal linked to the Contact, `Stage = Appointment Booked`

Zoho Flow does not currently support exporting a flow's definition as a file, so the build is documented through screenshots of the builder (trigger, fetch, if/else, and both create steps) in `/screenshots`.

## Known limitations

The core sync is live and verified end to end. Three values are still static rather than dynamic, all tracing back to one root cause: the `Appointment_Date` field in Creator is currently typed as `Time` instead of `Date-Time`, which is blocking two downstream mappings.

- `Appointment_Date` (Creator) needs to change from `Time` to `Date-Time`
- Deal `Closing Date` is currently hardcoded to `2026-12-31`, pending the field type fix above so it can map from `Appointment_Date`
- Deal `Creator_Record_ID` is currently hardcoded to `TEST-001`, pending a fix to pass the actual triggering entry ID from the Flow trigger payload

## Repository structure

```
/docs      full technical documentation (business summary, specs, user manual)
/creator   Deluge scripts and form field export
/screenshots  CRM pipeline, Creator form, Flow builder (trigger, fetch, if/else, create steps)
```

## Stack

Zoho CRM, Zoho Creator, Zoho Flow
