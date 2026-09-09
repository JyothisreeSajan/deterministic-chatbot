# UC-LH: Learning Progress / Learning Hours Not Reflecting (eHRMS / Shiksha Path / SPARROW-APAR) — API Integration Guide

> Karmayogi platform APIs consumed by the chatbot, in execution order. Intended for iGot developers integrating or extending this workflow.

---

## Execution Flow

```
ENTRY    → Which external portal is not reflecting learning progress?
                ↓
           eHRMS         → SECTION A
           Shiksha Path  → SECTION B (static guidance, no API)
           SPARROW/APAR  → SECTION C

SECTION A — eHRMS
STEP A1  → POST /api/private/user/v1/search
                ↓ Fetch eHRMS ID, External System Name, rootOrgId, channel
                ↓
           user not found / rootOrgId missing → error message, stop
           eHRMS ID present AND Ext. System Name present → redirect to eHRMS support
                                                             (collect name/email/eHRMS ID) → close
           eHRMS ID missing OR Ext. System Name missing    → MDO Admin Lookup

SECTION B — Shiksha Path
           → static message only (managed by Directorate of Training, CBDT)
           → no API calls; contact aed4.training@incometax.gov.in

SECTION C — SPARROW / APAR
STEP C1  → POST /api/private/user/v1/search
                ↓ Fetch basic profile (id, rootOrgId, channel, orgName, profileStatus, primaryEmail)
                ↓
           user NOT found → stop; apologize and ask user to try again later
           org == iGOT / Prarambh → wrong-org message → Transfer Request guide

STEP C2  → GET  /api/user/private/v1/read/{userId}
                ↓ Fetch full profile (verifiedKarmayogi, designation, group, cadre/service/batch/central deputation)
                ↓
           profile NOT verified → AIS eligibility check
                                    ↓ AIS yes + any of cadre/service/batch/central-deputation missing → "update profile" message, stop
                                    ↓ AIS no, or AIS yes with all 4 present            → MDO Admin Lookup
           profile verified        → show details, ask user to confirm → Step C3

STEP C3  → POST /api/supportportal/admin/user/v2/assignedcourses/{userId}
                ↓ Fetch assigned CAP courses (courseCategory: "Comprehensive Assessment Program")
                ↓
           empty / not assigned → MDO Admin Lookup
           assigned              → Step C4

STEP C4  → GET  /api/supportportal/cbplan/v2/admin/user/list/{userId}
                ↓ Fetch CBP Plans (each flagged isApar true/false)
                ↓
           cbp_total_count == 0 → MDO Admin Lookup
           cbp_total_count  > 0 → ask user: CAP or APAR? → Step C5

STEP C5  → POST /api/course/private/v4/user/enrollment/list/{userId}
                ↓ Fetch completed courses + build APAR nested Plan→Course picker
                ↓ User selects the specific CAP or APAR course they're asking about
                ↓
           selected course NOT completed → CAP/APAR-assessment-not-passed message, stop
           selected course completed     → ask: has it been more than 24 hours?
                                              ↓
                                        NO  → "may take up to 24 hours" message, stop
                                        YES → ask: does SPARROW email match iGOT email?
                                                ↓
                                          YES → share SPARROW support email, stop
                                          NO  → share video guide (action button), stop

MDO      → POST /api/private/user/v1/search             (MDO Admin lookup — shared by Sections A & C)
                ↓
           SPARROW + MDO found     → show "no APAR/CAP assigned" + MDO contact details
           SPARROW + MDO not found → YP/SPOC lookup fallback
           eHRMS   + MDO found     → show eHRMS ID / Ext. System Name update guidance + MDO contact
           eHRMS   + MDO not found → raise Zoho ticket (no self-service path)
           API error, section == eHRMS   → offer to raise Zoho ticket
           API error, section == SPARROW → generic retry message (no ticket)

YP       → data_lookup: yp_lookup (internal service, not a Karmayogi REST API)
                ↓
           found     → share YP/SPOC contact details, stop
           not found → generic "couldn't find KB contact, try again later" message, stop
                       (SPARROW path only — unlike other flows, no ticket is offered here)
```

> **Note:** Section A (eHRMS) and Section C (SPARROW/APAR) share the same MDO Admin Lookup node (`lookup_mdo_admin`), but the routing after it — and whether a support ticket is ultimately offered — differs by section. Section B (Shiksha Path) makes no API calls at all.

---

## Step A1 — eHRMS User Feed Fetch

> Confirms the user exists and retrieves the eHRMS ID / External System Name pair needed to route the user correctly.

**Endpoint:** `POST /api/private/user/v1/search`

```bash
curl -X POST \
  "https://portal.uat.karmayogibharat.net/api/private/user/v1/search" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "filters": {
        "userId": "{{user_id_hash}}"
      },
      "limit": 1
    }
  }'
```

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `response.count` | `collected.ehrms_user_count` | Verify user exists |
| `response.content[0].rootOrgId` | `collected.root_org_id` | Passed to MDO Admin lookup |
| `response.content[0].channel` | `collected.org_channel` | Display; MDO/YP lookup context |
| `response.content[0].profileDetails.additionalProperties.externalSystemId` | `collected.ehrms_id` | eHRMS ID presence check |
| `response.content[0].profileDetails.additionalProperties.externalSystem` | `collected.ext_system_name` | External System Name presence check |

> **Field note:** `externalSystem` is the assumed key for External System Name under `additionalProperties` — verify against the live API response before deploying.

### Decision After Step A1

| Condition | Outcome |
|---|---|
| `ehrms_user_count == 0` or `root_org_id` is null/empty | Show "couldn't retrieve your profile details" message; stop |
| `ehrms_id` present AND `ext_system_name` present | Both fields available; redirect user to eHRMS support (collect name/email/eHRMS ID), stop |
| `ehrms_id` present AND `ext_system_name` missing | Proceed to MDO Admin Lookup (Ext. System Name update path) |
| `ehrms_id` missing (default) | Proceed to MDO Admin Lookup (eHRMS ID update path) |

---

## Section B — Shiksha Path

> No API calls. Shiksha Path is managed by the Directorate of Training, CBDT — not by Karmayogi Bharat/DoPT — so the flow only shows a static referral message with the support contact `aed4.training@incometax.gov.in`.

---

## Step C1 — SPARROW/APAR User Profile Fetch

> Confirms the user exists and collects org context needed for all downstream SPARROW/APAR decisions.

**Endpoint:** `POST /api/private/user/v1/search`

```bash
curl -X POST \
  "https://portal.uat.karmayogibharat.net/api/private/user/v1/search" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "filters": {
        "userId": "{{user_id_hash}}"
      },
      "limit": 1
    }
  }'
```

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `response.count` | `collected.sp_user_count` | Verify user exists |
| `response.content[0].id` | `collected.fetched_user_id` | Primary key for all subsequent SPARROW API calls |
| `response.content[0].rootOrgId` | `collected.root_org_id` | Passed to MDO Admin lookup |
| `response.content[0].channel` | `collected.org_channel` | Org identity check; YP lookup key |
| `response.content[0].organisations[0].orgName` | `collected.org_name` | Org identity check; display |
| `response.content[0].profileDetails.profileStatus` | `collected.profile_status` | Verified / unverified branch |
| `response.content[0].profileDetails.personalDetails.primaryEmail` | `collected.user_primary_email` | Reference / display |

### Decision After Step C1

| Condition | Outcome |
|---|---|
| `sp_user_count == 0` or `fetched_user_id == null` | Show "couldn't find your profile" message; stop |
| `org_channel` or `org_name` == "igot", or `org_name` contains "prarambh" | Wrong org; show message → guide user to raise a Transfer Request |
| Default | Proceed to Step C2 |

---

## Step C2 — SPARROW/APAR Full Profile Fetch

> Retrieves verification status, designation/group, and All India Services (AIS) fields needed to determine why APAR/CAP may not be assigned.

**Endpoint:** `GET /api/user/private/v1/read/{userId}`

```bash
curl -X GET \
  "https://portal.uat.karmayogibharat.net/api/user/private/v1/read/{{fetched_user_id}}" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}"
```

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `response.channel` | `collected.org_channel` | Refresh org identity |
| `response.organisations[0].orgName` | `collected.org_name` | Refresh org name |
| `response.profileDetails.verifiedKarmayogi` | `collected.profile_verified` | Verified status (boolean) |
| `response.profileDetails.professionalDetails[0].designation` | `collected.user_designation` | Display; profile confirm screen |
| `response.profileDetails.professionalDetails[0].group` | `collected.user_group` | Display; profile confirm screen |
| `response.profileDetails.cadreDetails` | `collected.cadre_details` | AIS eligibility check |
| `response.profileDetails.cadreDetails.cadreName` | `collected.cadre_name` | Display |
| `response.profileDetails.cadreDetails.civilServiceName` | `collected.service_details` | AIS eligibility check |
| `response.profileDetails.cadreDetails.cadreBatch` | `collected.batch_details` | AIS eligibility check |
| `response.profileDetails.cadreDetails.isOnCentralDeputation` | `collected.central_deputation` | AIS eligibility check |

### Decision After Step C2

| Condition | Outcome |
|---|---|
| `profile_status == "VERIFIED"` or `profile_verified == true` | Show profile details for user confirmation → Step C3 |
| Default (not verified) | Ask AIS (IAS/IPS/IFS) eligibility question |

> **AIS branch:** If the user says "yes" to AIS and any of `cadre_details` / `service_details` / `batch_details` / `central_deputation` is missing, show a message asking the user to complete those profile fields (APAR should reflect in ~2–3 hours after updating) and stop. If AIS is "no", or AIS is "yes" with all four fields present, proceed to MDO Admin Lookup for profile-verification guidance.

---

## Step C3 — Assigned CAP Courses Fetch

> Checks whether any Comprehensive Assessment Program (CAP) courses are assigned to the user at all, before checking the specific APAR plan.

**Endpoint:** `POST /api/supportportal/admin/user/v2/assignedcourses/{userId}`

```bash
curl -X POST \
  "https://portal.uat.karmayogibharat.net/api/supportportal/admin/user/v2/assignedcourses/{{fetched_user_id}}" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "Content-Type: application/json" \
  -H "x-authenticated-user-token: " \
  -d '{
    "courseCategory": "Comprehensive Assessment Program"
  }'
```

> **TODO:** `x-authenticated-user-token` is currently sent empty in the flow definition — verify the correct value against the live API before release.

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `content` | `collected.assigned_courses` | CAP course picker options; assignment presence check |

### Decision After Step C3

| Condition | Outcome |
|---|---|
| `assigned_courses` is non-empty | Proceed to Step C4 (CBP Plan fetch) |
| `assigned_courses` is empty / null | Proceed to MDO Admin Lookup (Step 4.3 — "no APAR/CAP assigned") |
| API error | Show technical-issue message; stop — **not** treated as "not assigned" |

---

## Step C4 — CBP Plan Fetch (APAR Assignment Check)

> Confirms whether the user actually has an APAR-flagged Career Broadening Plan (CBP), distinct from the CAP-course-assignment check in Step C3.

**Endpoint:** `GET /api/supportportal/cbplan/v2/admin/user/list/{userId}`

```bash
curl -X GET \
  "https://portal.uat.karmayogibharat.net/api/supportportal/cbplan/v2/admin/user/list/{{fetched_user_id}}" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "x-authenticated-user-orgid: igot"
```

> The `x-authenticated-user-orgid: igot` header is **static** and required regardless of the user's actual org.

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `content` | `collected.all_courses` | CBP plan content, each item flagged `isApar` true/false |
| `count` | `collected.cbp_total_count` | Determines whether an APAR/CAP plan exists |

### Decision After Step C4

| Condition | Outcome |
|---|---|
| `cbp_total_count > 0` | Ask user which category they're facing an issue with: CAP or APAR → Step C5 |
| `cbp_total_count == 0` or API error | Proceed to MDO Admin Lookup (no APAR/CAP assigned) |

---

## Step C5 — Enrollment Fetch & Course Completion Check

> Fetches completed courses to verify the specific CAP/APAR course the user selected is actually done, and builds the nested APAR Plan→Course picker (same `nested_apar_courses` transform used by the APAR-not-visible flow).

**Endpoint:** `POST /api/course/private/v4/user/enrollment/list/{userId}`

```bash
curl -X POST \
  "https://portal.uat.karmayogibharat.net/api/course/private/v4/user/enrollment/list/{{fetched_user_id}}" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "retiredCoursesEnabled": true
    }
  }'
```

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `courses` | `collected.sp_completed_courses` | Match against the user-selected course to check completion |
| `courses` (transform: `nested_apar_courses`, ctx: `collected.all_courses`) | `collected.sp_apar_picker_list` | APAR course picker options (grouped by Plan) |

> On API error, the flow still proceeds to the category/course picker (`sp_branch_category_picker`) rather than dead-ending — completion is then evaluated against whatever data is available.

### Course Picker

- **CAP** category → picker built from `collected.assigned_courses` (already fetched in Step C3; no extra API call).
- **APAR** category → picker built from `collected.sp_apar_picker_list` (nested Plan→Course, from this step).

### Decision After Selecting a Course

| Condition | Outcome |
|---|---|
| Selected `courseId` found in `sp_completed_courses` with `status == 2` / `"completed"` | Course completed → ask "has it been more than 24 hours?" |
| Otherwise | Show "course/assessment not successfully completed yet" message; stop |

### Decision — More Than 24 Hours?

| Condition | Outcome |
|---|---|
| No (within 24 hours) | Show "may take up to 24 hours to reflect" message; stop |
| Yes (more than 24 hours) | Ask "does the SPARROW-registered email match your iGOT email?" |

### Decision — SPARROW Email Match

| Condition | Outcome |
|---|---|
| Yes, same email | Show **SPARROW Support Email** (`support-sparrow@gov.in`) — ask user to contact them directly with iGOT email + course details; stop |
| No, different email | Show the **video guide** action button — see below; stop |

#### Video Guide — Email Mismatch Node (`sp_email_no_match_info`)

When the SPARROW-registered email does not match the user's iGOT email, the flow does **not** call any API — it renders a static `action_button` on the message node pointing to an instructional video:

```yaml
- id: sp_email_no_match_info
  type: message
  prompt:
    text: |
      Since the email IDs are different, the iGOT training data may not be automatically linked to your SPARROW account.

      Please refer to the video guide below for instructions on how to fetch iGOT training data into SPARROW APAR:
  action_button:
    label: "📹 How to Fetch iGOT Training Data into SPARROW APAR"
    url: "https://youtu.be/gSMSuFib2n8"
  next: satisfied
```

- `action_button.label` / `action_button.url` render as a clickable button below the message text (not inline markdown), on whichever channel the flow is enabled for.
- The current `url` is a **YouTube** link, not a SharePoint link — swap this field if the video is moved to SharePoint or another host.
- No `collected.*` fields are read or written by this node; it is a terminal, purely-informational step (`next: satisfied`).

---

## MDO Admin Lookup (Shared — Sections A & C)

> Identifies the MDO Admin for the user's organisation. Imported from the shared `_mdo_admin_lookup` fragment and reused by both the eHRMS and SPARROW/APAR sections; the routing after it differs by section.

**Endpoint:** `POST /api/private/user/v1/search`

```bash
curl -X POST \
  "https://portal.uat.karmayogibharat.net/api/private/user/v1/search" \
  -H "Authorization: Bearer {{KARMAYOGI_API_KEY}}" \
  -H "Content-Type: application/json" \
  -d '{
    "request": {
      "filters": {
        "rootOrgId": "{{root_org_id}}",
        "organisations.roles": ["MDO_ADMIN"],
        "status": 1
      },
      "limit": 1
    }
  }'
```

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `response.count` | `collected.mdo_admin_count` | Determine if MDO Admin exists |
| `response.content[0].profileDetails.personalDetails.firstname` | `collected.mdo_admin_name` | Display name |
| `response.content[0].profileDetails.personalDetails.primaryEmail` | `collected.mdo_admin_email` | Contact email |

### Decision After MDO Lookup

| Condition | Outcome |
|---|---|
| `external_system == SPARROW` and `mdo_admin_count > 0` | Show "no APAR/CAP assigned" + MDO contact details; stop |
| `external_system == SPARROW` and `mdo_admin_count == 0` | Fall back to YP/SPOC lookup |
| `external_system == EHRMS` and `mdo_admin_count > 0` and `ehrms_id` missing | Show eHRMS ID update guidance + MDO contact; stop |
| `external_system == EHRMS` and `mdo_admin_count > 0` (Ext. System Name missing) | Show External System Name update guidance + MDO contact; stop |
| `mdo_admin_count == 0` (default — eHRMS, no MDO found) | Raise Zoho support ticket (see Ticket Creation) |
| API failure, `external_system == EHRMS` | Offer to raise a Zoho support ticket |
| API failure, `external_system == SPARROW` | Generic "technical issue, try again later" message; **no ticket offered** |

---

## YP/SPOC Fallback Lookup (SPARROW only)

> When no MDO Admin is found for a SPARROW/APAR case, the flow looks up the YP/SPOC contact via an internal data service. Imported from the shared `_yp_lookup` fragment.

**Service:** `yp_lookup` (internal data lookup — not a Karmayogi REST API)

**Lookup key:** `collected.org_channel`

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `name` | `collected.yp_name` | Display name |
| `email` | `collected.yp_email` | Contact email |
| `mobile` | `collected.yp_mobile` | Contact mobile |
| `cc_email` | `collected.yp_cc_email` | CC email for correspondence |

### Decision After YP/SPOC Lookup

| Condition | Outcome |
|---|---|
| YP/SPOC found | Display contact details; stop |
| YP/SPOC not found | Show "couldn't find the KB point of contact, please try again after some time" message; stop — **no ticket is offered** on this path (unlike the APAR-not-visible flow's YP/SPOC fallback) |

---

## Ticket Creation (eHRMS — No MDO Admin Found)

> Reached only from the eHRMS section, when the MDO Admin lookup returns zero results (or fails). SPARROW's "no contact found" paths (YP/SPOC not found, MDO API error) do **not** raise a ticket in this flow.

**Endpoint:** `POST /tickets` (integration: `zoho_desk_api`, via the shared `_zoho_ticket` fragment)

- `mdo_not_found_fallback` (`ticket_confirm`) — shows a summary ("Learning Hours Not Reflecting — No MDO Admin Mapped to Organization", including the identified organization/channel) and asks the user to confirm.
- `raise_ticket_no_mdo` (`transfer_llm`, `auto_raise: silent`) — an LLM drafts the ticket subject ("Learning Hours Not Reflecting — No MDO Admin Found") and description (organization/channel, external system, eHRMS ID / External System Name status), `priority_override: P3`.
- `confirm_ticket` (`api_call`) — creates the ticket via `POST /tickets`.
- Custom fields set via this flow's `_zoho_ticket` import parameters: `cf_category: "Recognition & Engagement"`, `cf_sub_category: "Learning Hours"`, `cf_flow_id: LEARNING_HOURS`.

### Response Fields Used

| Field | Mapped To | Used For |
|---|---|---|
| `$.ticketNumber` | `collected.ticket_id` | Displayed to the user as the ticket ID |

### Decision After Ticket Creation

| Condition | Outcome |
|---|---|
| User confirms ticket creation | Raise Zoho ticket; show ticket ID |
| User declines | End conversation; no ticket raised |
| Ticket API fails | Show failure message; end |

---

## API Dependency Table

| Step | Endpoint | Method | Auth Required | Special Headers | Purpose | Key Fields |
|---|---|---|---|---|---|---|
| A1 | `/api/private/user/v1/search` | POST | Yes | — | eHRMS: fetch eHRMS ID + External System Name + org context | `ehrms_id`, `ext_system_name`, `rootOrgId`, `channel` |
| C1 | `/api/private/user/v1/search` | POST | Yes | — | SPARROW: fetch basic user profile + org context | `id`, `rootOrgId`, `channel`, `profileStatus` |
| C2 | `/api/user/private/v1/read/{userId}` | GET | Yes | — | SPARROW: fetch verification status, designation/group, cadre/AIS fields | `verifiedKarmayogi`, `designation`, `group`, `cadreDetails` |
| C3 | `/api/supportportal/admin/user/v2/assignedcourses/{userId}` | POST | Yes | `x-authenticated-user-token` (TODO: verify value) | Fetch assigned CAP courses | `content[].identifier`, `content[].name` |
| C4 | `/api/supportportal/cbplan/v2/admin/user/list/{userId}` | GET | Yes | `x-authenticated-user-orgid: igot` | Fetch CBP Plan (APAR assignment check) | `count`, `content[].isApar` |
| C5 | `/api/course/private/v4/user/enrollment/list/{userId}` | POST | Yes | — | Fetch completed courses; verify selected course completion | `courses[].courseId`, `courses[].status` |
| MDO | `/api/private/user/v1/search` | POST | Yes | — | MDO Admin lookup (shared: eHRMS & SPARROW) | `mdo_admin_name`, `mdo_admin_email` |
| YP | `yp_lookup` (internal data service) | — | — | — | YP/SPOC contact lookup (SPARROW only, no ticket fallback) | `yp_name`, `yp_email`, `yp_mobile` |
| Ticket | `/tickets` (zoho_desk_api) | POST | Yes | — | Raise support ticket (eHRMS: no MDO Admin found) | `ticketNumber` |

---
 