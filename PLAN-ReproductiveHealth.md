# HavenAI: Privacy-First Reproductive Health Clinic Intelligence

## YHack 2026 — March 28-29, 2026 (24-hour hackathon)

---

## 1. Problem Statement

### Post-Dobbs Digital Surveillance of Reproductive Healthcare

Since the Supreme Court's Dobbs decision (June 2022), prosecutors in restrictive states have used **digital evidence** to pursue criminal cases against people seeking reproductive care. This is not hypothetical — it is documented and ongoing.

**Documented cases:**

| Case | Digital Evidence Used | Outcome |
|------|---------------------|---------|
| **Nebraska v. Celeste Burgess (2022)** | Prosecutors subpoenaed **Facebook Messenger** messages between a teenager and her mother discussing medication abortion. Meta complied and handed over private messages. | Mother (Jessica Burgess) convicted. Teenager pleaded guilty. |
| **Indiana v. Dr. Caitlin Bernard (2022)** | Indiana AG investigated a doctor who provided abortion to a 10-year-old rape victim. **Medical records and reporting data** were central to the investigation. | Doctor fined for minor state reporting technicality. |
| **Texas SB 8 civil cases (2021-present)** | Social media posts, GoFundMe campaigns, text messages used to identify targets under Texas's civil bounty mechanism. | Multiple suits filed against abortion funds. |
| **Mississippi, Tennessee, South Carolina (2022-2024)** | Phone **search history** ("how to get abortion pills," "misoprostol dosage"), **text messages**, and **purchase records** used to prosecute self-managed abortion. | Multiple convictions. |

**Broader surveillance infrastructure:**
- **Geofence warrants**: Law enforcement has obtained location data for devices near clinic locations (documented by EFF and Surveillance Technology Oversight Project)
- **Period tracking apps**: Flo Health faced FTC action for sharing data with Facebook/Google analytics. Post-Dobbs, Flo, Clue, Ovia, and others became prosecution vectors.
- **Pharmacy records**: Purchases of mifepristone/misoprostol create paper trails
- **Cloud EHR data**: Patient portals (MyChart, etc.) store records on cloud servers subject to legal process
- **The Markup's Pixel Hunt investigation**: Found tracking pixels on hospital and telehealth websites sending data to Meta and Google

### The Clinic Technology Problem

Reproductive health clinics face an impossible tradeoff:
- They **need** digital tools for patient intake, records, telehealth, and care coordination
- **Every cloud-based tool** creates data that can be subpoenaed — EHR vendors (Epic, athenahealth, NextGen), telehealth platforms (Doxy.me, Zoom for Healthcare), and even website analytics
- **HIPAA does not protect against legal process** — HIPAA allows disclosure pursuant to court orders, subpoenas, and administrative requests
- The April 2024 HIPAA Privacy Rule update (prohibiting disclosure for prosecution of lawful reproductive care) is vulnerable to being rescinded by future administrations

**Current clinic responses (ad hoc, insufficient):**
- Some clinics allow patients to use non-legal names
- Some encourage cash payment to avoid insurance trails
- Some have moved back to **paper records** for abortion care
- Some minimize data collection (but this degrades care quality)
- None have a systematic technological solution

### The Gap

No **purpose-built, privacy-first, on-device clinical intelligence system** exists for reproductive health clinics. Clinics are choosing between:
1. Cloud-based tools that create subpoena-able records, or
2. Paper records that degrade care quality and can't leverage AI

---

## 2. Solution

**HavenAI** is an on-device clinical intelligence system running K2 Think V2 (70B) locally on an ASUS Ascent GX10, deployed at reproductive health clinics. It:

1. **Processes patient intake entirely on-device** — medical history, screening, eligibility assessment — with zero cloud transmission
2. **Provides clinical decision support** — contraindication checking, protocol adherence, dosage verification — powered by K2's multi-step reasoning
3. **Generates aftercare plans** personalized to the patient's medical situation and home-state legal environment
4. **Runs ephemeral sessions** — configurable to zero-out all patient data after the visit completes, leaving nothing to subpoena
5. **Operates air-gapped** — no internet connection required. The device sits in the clinic, on no external network.

### The Pitch Line
> "In Nebraska, prosecutors subpoenaed Facebook messages to convict a teenager. In Texas, search history became evidence. Every cloud tool your clinic uses is one subpoena away from becoming a prosecution exhibit. HavenAI runs entirely on this device. When the session ends, the data is gone. There is nothing to subpoena, nothing to seize, nothing to hand over."

---

## 3. Why On-Device Is Life-or-Death (Not Contrived)

| Reason | Why it's non-negotiable |
|--------|------------------------|
| **Prosecutors use digital evidence** | Nebraska v. Burgess proved Meta will comply with subpoenas for private messages. Cloud EHR vendors will too. On-device with ephemeral sessions means no data exists to subpoena. |
| **HIPAA doesn't protect** | HIPAA allows disclosure under court orders and subpoenas. The 2024 rule update is politically vulnerable. On-device processing doesn't rely on any legal protection — the data simply doesn't exist externally. |
| **Geofence warrants** | Law enforcement obtains location data for devices near clinics. An air-gapped GX10 generates zero location-trackable signals. |
| **Third-party tracking** | The Markup found tracking pixels on hospital websites sending data to Meta/Google. An on-device system with no internet connection has zero third-party tracking by definition. |
| **Cross-state subpoenas** | 15+ states have shield laws blocking out-of-state reproductive care subpoenas. But shield laws only protect data **within the state**. If clinic data is on AWS servers in Virginia, shield law coverage is uncertain. On-device in the clinic = data is physically in the shield-law state. |
| **Patient trust** | Patients from restrictive states are terrified of digital trails. Many avoid care entirely. "Nothing is stored, nothing leaves this room" is the only message that rebuilds trust. |
| **Staff protection** | Clinic staff face prosecution risks in some jurisdictions. Minimizing the data that exists protects staff as well as patients. |

---

## 4. Hardware and Model Specifications

### ASUS Ascent GX10
- **GPU**: NVIDIA Blackwell GPU (GB10 superchip), 5th-gen Tensor Cores, 1 PETAFLOP AI
- **CPU**: 20-core NVIDIA Grace ARM v9.2-A
- **RAM**: 128 GB unified LPDDR5x
- **Storage**: Up to 4TB PCIe 5.0 NVMe SSD
- **OS**: DGX OS (Ubuntu Linux)
- **Architecture**: ARM aarch64
- **Power**: 240W
- **Size**: 150 x 150 x 51 mm (~6 inches square)

### K2-V2-Instruct (70B)
- **Source**: MBZUAI Institute of Foundation Models (LLM360)
- **Parameters**: 70B, Apache 2.0 license
- **Context window**: 524,288 tokens
- **HuggingFace**: `LLM360/K2-V2-Instruct`
- **Why this model**: Multi-step clinical reasoning (contraindication cascades, protocol adherence, personalized aftercare) requires genuine reasoning capability, not keyword matching.
- **Why Instruct (not Think)**: Faster structured output for clinical workflows.

### Performance on GX10
- NVFP4: ~15-20 tokens/second
- Clinical intake analysis (~1000 tokens output): ~50-60 seconds
- Aftercare plan generation (~1500 tokens): ~75-90 seconds
- Fast enough for real-time clinical use

### Model Serving
- **Primary**: vLLM via DGX OS playbook (`vllm serve LLM360/K2-V2-Instruct --dtype float16`)
- **Fallback 1**: llama.cpp with GGUF quantization
- **Fallback 2**: Qwen2.5-14B-Instruct
- **Fallback 3**: Remote K2 API if sponsor provides (compromise on privacy pitch)

---

## 5. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ASUS Ascent GX10                         │
│                      (clinic back office)                        │
│                                                                 │
│  ┌──────────────┐                ┌──────────────────────────┐  │
│  │  K2-V2-Inst  │◄─────────────│      FastAPI Server       │  │
│  │  (vLLM)      │  OpenAI API  │                           │  │
│  │  localhost:   │              │  - Intake processing      │  │
│  │  8000        │              │  - Clinical decision spt  │  │
│  └──────────────┘              │  - Aftercare generation   │  │
│                                 │  - Session management     │  │
│  ┌──────────────┐              │  - WebSocket to dashboard │  │
│  │  Ephemeral    │              └──────────┬────────────────┘  │
│  │  Session Store│                         │                   │
│  │  (RAM only)   │                         │ WebSocket         │
│  │  - Auto-purge │                         │                   │
│  │  - No disk    │              ┌──────────┴────────────────┐  │
│  │    writes     │              │    React Dashboard        │  │
│  └──────────────┘              │    (clinician-facing)     │  │
│                                 │                           │  │
│  ┌──────────────┐              │  - Patient intake flow    │  │
│  │  Medical      │              │  - Clinical alerts        │  │
│  │  Knowledge    │              │  - Aftercare plan view    │  │
│  │  Base (local) │              │  - Session controls       │  │
│  │  - Drug       │              └───────────────────────────┘  │
│  │    interactions│                                             │
│  │  - Protocols  │                                             │
│  │  - State laws │                                             │
│  └──────────────┘                                              │
│                                                                 │
│         ⛔ AIR-GAPPED — ZERO NETWORK — EPHEMERAL ⛔            │
└─────────────────────────────────────────────────────────────────┘
```

### Clinical Workflow

1. **SESSION START**: Clinician initiates new patient session. Session ID generated. All data held in RAM only.

2. **INTAKE**: Clinician enters patient information through guided intake flow:
   - Medical history (relevant conditions, allergies, current medications)
   - Gestational age assessment (LMP date, ultrasound if available)
   - Screening questions (contraindications, risk factors)
   - Home state (for aftercare legal context)

3. **CLINICAL ANALYSIS**: K2 processes intake data and provides:
   - **Contraindication check**: Flags medication interactions, medical conditions that affect care plan (e.g., ectopic pregnancy risk factors, IUD in place, anticoagulant use)
   - **Protocol adherence**: Verifies care plan matches clinical guidelines (ACOG, WHO, NAF protocols)
   - **Risk stratification**: Identifies patients needing additional monitoring or referral

4. **AFTERCARE GENERATION**: K2 generates personalized aftercare plan considering:
   - Medical specifics (medication, dosage, timing, expected symptoms)
   - Warning signs requiring emergency care
   - Home-state legal context (what's safe to search, what pharmacy to use, telehealth follow-up options)
   - Emotional support resources

5. **SESSION END**: Clinician closes session. **All patient data purged from RAM.** Nothing written to disk. Nothing to subpoena.

---

## 6. Test Data: Synthetic Patient Scenarios

### Data Generation
We use **Synthea** (MIT license, by MITRE) to generate realistic synthetic patient records, customized with reproductive health modules. All demo data is clearly labeled as synthetic.

### Demo Scenarios (3 patients demonstrating different capabilities)

#### Scenario 1: "Maria Santos" — Medication Abortion, Contraindication Detection
```
Patient: Maria Santos, 24F
Home state: Texas (restrictive)
Currently at: Clinic in New Mexico (access state, shield law)
LMP: 7 weeks ago
Medical history: Asthma (well-controlled), currently on corticosteroids
Current medications: Prednisone 10mg daily, albuterol inhaler PRN
Allergies: Sulfa drugs
Request: Medication abortion
```

**What the agent should catch:**
- Corticosteroid use requires monitoring — potential interaction with mifepristone (both affect cortisol pathways). Not a contraindication but requires clinician awareness.
- Gestational age (7 weeks) is within medication abortion guidelines
- Home state is Texas — aftercare plan must address: avoid searching for follow-up information on personal devices, use clinic-provided burner number for follow-up, know which ER symptoms require emergency care (important: ER physicians treat incomplete miscarriage the same regardless of cause — patient should present symptoms, not cause)

**Expected agent output:**
```json
{
  "clinical_assessment": {
    "eligible": true,
    "gestational_age_weeks": 7,
    "within_guidelines": true,
    "contraindications": [],
    "alerts": [
      {
        "severity": "MODERATE",
        "title": "Corticosteroid Interaction — Monitor",
        "detail": "Patient on Prednisone 10mg daily. Mifepristone is a glucocorticoid receptor antagonist and may reduce corticosteroid efficacy. Consider: (1) temporary prednisone dose adjustment, (2) additional monitoring for adrenal insufficiency symptoms, (3) consult with prescribing physician if feasible without compromising patient privacy.",
        "action_required": "Clinician review before proceeding"
      }
    ],
    "risk_level": "STANDARD_WITH_MONITORING"
  },
  "aftercare_plan": {
    "medical": "...",
    "privacy_guidance": {
      "home_state": "Texas",
      "risk_level": "HIGH",
      "guidance": [
        "Do not search for abortion-related terms on personal devices",
        "Use the clinic-provided follow-up number for questions",
        "If you need emergency care: present symptoms (heavy bleeding, fever >100.4°F, severe pain), not the cause — ER treats incomplete miscarriage identically regardless of etiology",
        "Do not discuss details over unencrypted messaging (SMS, Facebook Messenger)",
        "Consider using Signal for any necessary communication with the clinic"
      ]
    }
  }
}
```

#### Scenario 2: "Jennifer Park" — Complex Medical History, Multi-Step Reasoning
```
Patient: Jennifer Park, 31F
Home state: Ohio (restrictive, 6-week ban)
Currently at: Clinic in Illinois (access state, shield law)
LMP: 9 weeks ago
Medical history: Type 1 diabetes, prior C-section (2023), IUD (Mirena) —
  IUD strings not visible on self-exam
Current medications: Insulin (Humalog/Lantus), Metformin 500mg
Allergies: None known
Request: Medication abortion
```

**What the agent should catch (multi-step reasoning):**
- **IUD in place with non-visible strings**: This is a **critical finding**. IUD must be confirmed removed before medication abortion. Non-visible strings may indicate embedded/migrated IUD. Requires ultrasound confirmation of IUD location. If IUD is embedded, surgical removal may be needed first.
- **Type 1 diabetes**: Mifepristone can affect blood glucose levels. Patient needs glucose monitoring plan during medication abortion process.
- **Prior C-section**: Slightly elevated risk of uterine complications. Not a contraindication but noted for monitoring.
- **9 weeks gestational age**: Within guidelines but approaching upper limit for some protocols. Confirm accurate dating.

This scenario demonstrates genuine multi-step clinical reasoning — the IUD finding changes the entire care pathway.

#### Scenario 3: "Ashley Williams" — Straightforward Case, Aftercare Focus
```
Patient: Ashley Williams, 28F
Home state: Georgia (6-week ban)
Currently at: Clinic in Virginia (access state)
LMP: 6 weeks ago
Medical history: None significant
Current medications: Prenatal vitamins (discontinued), oral contraceptive (discontinued 3 months ago)
Allergies: Penicillin
Request: Medication abortion
```

**What the agent should produce:**
- Straightforward clinical assessment — no contraindications, no alerts
- Detailed aftercare plan with Georgia-specific privacy guidance
- Follow-up care options that minimize digital footprint
- Emotional support resources

This scenario demonstrates the aftercare generation capability and state-specific privacy guidance.

---

## 7. Agent Prompt Design

### System Prompt

```
You are HavenAI, an on-device clinical intelligence system for reproductive
health clinics. You run locally on an ASUS Ascent GX10 with zero cloud
connectivity. All patient data is ephemeral — held in RAM only and purged
when the session ends.

YOUR ROLE:
- Assist clinicians with patient intake analysis and clinical decision support
- Identify contraindications, drug interactions, and risk factors
- Verify care plans against clinical guidelines (ACOG, WHO, NAF protocols)
- Generate personalized aftercare plans considering medical AND privacy needs
- Flag situations requiring additional clinician review

YOU ARE NOT:
- A replacement for clinical judgment. You assist, the clinician decides.
- Providing direct patient care. A licensed clinician is always present.
- Storing any patient data. Sessions are ephemeral.

CLINICAL GUIDELINES:
- Medication abortion (mifepristone + misoprostol): FDA-approved up to 70 days
  (10 weeks) gestational age
- Mifepristone contraindications: confirmed/suspected ectopic pregnancy,
  IUD in place, chronic adrenal failure, concurrent long-term corticosteroid
  therapy, allergy to mifepristone/misoprostol, inherited porphyria,
  anticoagulant therapy
- Warning signs requiring emergency care: heavy bleeding (soaking >2 pads/hr
  for 2+ hours), fever >100.4°F lasting >24 hours, severe abdominal pain not
  responsive to ibuprofen, foul-smelling discharge
- Follow-up: confirm completion within 7-14 days

PRIVACY-AWARE AFTERCARE:
When generating aftercare plans, consider the patient's home state legal
environment. For patients from restrictive states:
- Advise against searching abortion-related terms on personal devices
- Recommend encrypted communication (Signal) for follow-up
- ER guidance: present symptoms, not cause (ER treats incomplete miscarriage
  identically regardless of etiology)
- Avoid discussing details on social media or unencrypted messaging

STATE LEGAL CONTEXT (simplified, embedded in knowledge base):
[Include current status of major state restrictions — which states ban,
at what gestational age, exceptions, shield law states]

OUTPUT FORMAT: Respond with a JSON object:
{
  "clinical_assessment": {
    "eligible": true/false,
    "gestational_age_weeks": <number>,
    "within_guidelines": true/false,
    "contraindications": [{"item": "...", "severity": "CRITICAL/MODERATE/LOW"}],
    "alerts": [{"severity": "...", "title": "...", "detail": "...", "action_required": "..."}],
    "risk_level": "STANDARD | STANDARD_WITH_MONITORING | ELEVATED | REFER_OUT"
  },
  "aftercare_plan": {
    "medical": {
      "medications": "...",
      "expected_symptoms": "...",
      "warning_signs": "...",
      "follow_up": "..."
    },
    "privacy_guidance": {
      "home_state": "...",
      "risk_level": "HIGH | MODERATE | LOW",
      "guidance": ["..."]
    },
    "emotional_support": "..."
  }
}

IMPORTANT: Output ONLY valid JSON. No markdown, no code fences.
```

### User Message Template

```
NEW PATIENT SESSION
==================

Demographics: {age}{sex}
Home state: {home_state}
Current clinic location: {clinic_state}

Medical History:
{medical_history}

Current Medications:
{medications}

Allergies:
{allergies}

Reproductive History:
- LMP: {lmp_date} ({gestational_weeks} weeks ago)
- Prior pregnancies: {gravida_para}
- Contraception history: {contraception}
- Relevant surgical history: {surgical}

Current Request: {request_type}
Additional Notes: {notes}

Analyze this patient's clinical situation. Identify all contraindications,
interactions, and risk factors. Generate a clinical assessment and
personalized aftercare plan. Output as JSON.
```

---

## 8. Project Structure

```
HavenAI/
├── README.md
├── requirements.txt
├── config.py                      # vLLM URL, model name, session timeout
│
├── main.py                        # FastAPI entry point
│                                  #   - Intake submission endpoint
│                                  #   - Analysis endpoint
│                                  #   - Session management (create/destroy)
│                                  #   - WebSocket for real-time updates
│                                  #   - Serves React dashboard
│
├── agent/
│   ├── __init__.py
│   ├── clinical.py                # Clinical analysis orchestrator
│   │                              #   - Processes intake → calls K2 → returns assessment
│   │                              #   - Validates output against safety bounds
│   │                              #   - Generates aftercare plan
│   │
│   ├── llm.py                     # K2 API client wrapper
│   │                              #   - OpenAI-compatible API calls
│   │                              #   - JSON parsing with retry
│   │
│   └── prompts.py                 # System prompt + user message templates
│
├── knowledge/
│   ├── drug_interactions.json     # Mifepristone/misoprostol interaction database
│   ├── protocols.json             # ACOG/WHO/NAF clinical protocols
│   ├── state_laws.json            # State-by-state legal status (restrictive/access/shield)
│   └── aftercare_templates.json   # Base aftercare plan templates by scenario
│
├── session/
│   ├── __init__.py
│   └── ephemeral.py               # RAM-only session management
│                                  #   - In-memory dict of active sessions
│                                  #   - Auto-timeout (configurable, default 60 min)
│                                  #   - Explicit destroy endpoint
│                                  #   - ZERO disk writes for patient data
│                                  #   - Session data: intake, assessment, aftercare only
│
├── demo/
│   ├── patients/                  # Synthetic patient scenarios
│   │   ├── maria_santos.json
│   │   ├── jennifer_park.json
│   │   └── ashley_williams.json
│   └── generate_patients.py       # Script to generate synthetic data
│
├── dashboard/                     # React app (Vite + TypeScript)
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.tsx                # Layout + session management
│       ├── hooks/useSocket.ts     # WebSocket for real-time analysis
│       ├── pages/
│       │   ├── Intake.tsx         # Guided intake form (step-by-step)
│       │   └── Assessment.tsx     # Clinical results + aftercare view
│       ├── components/
│       │   ├── IntakeForm.tsx     # Multi-step intake wizard
│       │   ├── ClinicalAlerts.tsx # Contraindication/alert cards (color-coded)
│       │   ├── AftercarePlan.tsx  # Printable aftercare document
│       │   ├── PrivacyGuide.tsx   # State-specific privacy guidance
│       │   ├── SessionTimer.tsx   # Countdown to auto-purge, manual purge button
│       │   └── PrivacyBadge.tsx   # "🔒 Ephemeral Session — No Data Stored"
│       └── styles/
│           └── theme.css
│
└── scripts/
    └── start_vllm.sh
```

---

## 9. Dashboard Design

### Technology
- React + TypeScript, Vite
- Multi-step form wizard for intake
- WebSocket for analysis progress
- Print-optimized aftercare view

### Color Scheme
- Background: `#0f0a14` (deep purple-black — warmer than pure dark, feels safe)
- Panel backgrounds: `#1a1428`
- Critical alert: `#ff4444` (red)
- Moderate alert: `#ffaa00` (amber)
- Safe/clear: `#00cc77` (green)
- Privacy elements: `#9966ff` (purple — distinct from clinical colors)
- Text: `#e8e0f0`

### Layout — Intake Flow

```
┌──────────────────────────────────────────────────────────────────┐
│  🔒 HavenAI — Ephemeral Session — No Data Stored                │
│  Session: #a7f3 | Auto-purge in: 52:30 | [🗑 End Session Now]   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PATIENT INTAKE                          Step 2 of 4             │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━         │
│  ● Demographics  ● Medical History  ○ Medications  ○ Request     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Medical History                                         │   │
│  │                                                          │   │
│  │  Conditions (select all that apply):                     │   │
│  │  □ Diabetes     □ Hypertension    □ Asthma              │   │
│  │  □ Blood clotting disorder        □ Ectopic pregnancy   │   │
│  │  □ Adrenal condition              □ Porphyria           │   │
│  │  □ Other: _____________                                  │   │
│  │                                                          │   │
│  │  IUD currently in place?  ○ Yes  ○ No  ○ Unsure         │   │
│  │  If yes, strings visible? ○ Yes  ○ No  ○ N/A            │   │
│  │                                                          │   │
│  │  Prior pregnancies: G__ P__                              │   │
│  │  Prior C-section?  ○ Yes  ○ No                          │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│                                    [← Back]  [Next →]            │
└──────────────────────────────────────────────────────────────────┘
```

### Layout — Assessment Results

```
┌──────────────────────────────────────────────────────────────────┐
│  🔒 HavenAI — Ephemeral Session — No Data Stored                │
│  Session: #a7f3 | Auto-purge in: 48:15 | [🗑 End Session Now]   │
├────────────────────────┬─────────────────────────────────────────┤
│                        │                                         │
│  CLINICAL ASSESSMENT   │  AFTERCARE PLAN                         │
│                        │                                         │
│  Eligible: ✅ Yes      │  📋 Medical Aftercare                   │
│  Gestational Age: 7wk  │  - Mifepristone 200mg oral (clinic)    │
│  Risk Level: STANDARD  │  - Misoprostol 800mcg buccal (24-48h)  │
│  WITH MONITORING       │  - Ibuprofen 600mg q6h PRN for pain    │
│                        │                                         │
│  ⚠️ MODERATE ALERT     │  Expected: Bleeding + cramping for      │
│  Corticosteroid        │  1-2 weeks. Clots up to lemon-size     │
│  Interaction           │  are normal.                            │
│  Prednisone 10mg may   │                                         │
│  have reduced efficacy │  🚨 Seek emergency care if:             │
│  with mifepristone.    │  - Soaking >2 pads/hr for 2+ hrs      │
│  Monitor for adrenal   │  - Fever >100.4°F for >24 hrs          │
│  insufficiency.        │  - Severe pain not responding to        │
│  → Clinician review    │    ibuprofen                            │
│    before proceeding   │                                         │
│                        │  🔒 PRIVACY GUIDANCE (Texas)            │
│  No contraindications  │  - Do NOT search abortion terms on      │
│  found.                │    personal devices                     │
│                        │  - Use Signal for clinic follow-up      │
│  No drug interactions  │  - At ER: describe symptoms, not cause  │
│  (critical).           │  - Avoid social media discussion        │
│                        │                                         │
│                        │  [🖨 Print Aftercare]  [📄 Export]      │
├────────────────────────┴─────────────────────────────────────────┤
│  [🗑 End Session & Purge All Data]                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Implementation Timeline (24 hours)

### Team Roles (3-4 people)
- **Person A** (ML/Agent): GX10 setup, model serving, prompt engineering, clinical analysis module
- **Person B** (Backend): FastAPI server, session management, knowledge base, demo data
- **Person C** (Frontend): React dashboard, intake form wizard, assessment view, aftercare display
- **Person D** (Integration/Polish): End-to-end wiring, demo prep, video, print styling

### Pre-Hackathon Prep (no code — rules compliant)
- [ ] Finalize architecture diagram (printable)
- [ ] Write all prompt templates as plain text documents
- [ ] Write synthetic patient scenarios as text descriptions
- [ ] Research and document: mifepristone contraindications, drug interactions, ACOG protocols, state-by-state legal status — as reference notes
- [ ] All team members install **Zed IDE** (Built with Zed track)
- [ ] Download K2-V2-Instruct weights to USB drive
- [ ] Bookmark: vLLM DGX OS playbook, Synthea docs, Recharts docs
- [ ] Assign roles
- [ ] Print this plan

### Phase 1: Foundation (11:00am - 2:00pm, 3 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| GX10 setup + vLLM | Person A | 150 min | Boot, GPU verify, serve K2-V2-Instruct, verify API |
| FastAPI skeleton | Person B | 90 min | Session management endpoints (create/destroy), intake submission, analysis trigger, WebSocket |
| Ephemeral session store | Person B | 45 min | In-memory dict, auto-timeout, explicit destroy, ZERO disk writes for patient data |
| Knowledge base files | Person B | 45 min | `drug_interactions.json`, `protocols.json`, `state_laws.json` — static reference data |
| React dashboard shell | Person C | 90 min | Vite + React-TS, routing (Intake → Assessment), WebSocket hook, layout |
| Intake form wizard | Person C | 60 min | 4-step form: Demographics → Medical History → Medications → Request |

**Milestone**: vLLM serving. Can submit intake via form. Session creates/destroys in RAM.

### Phase 2: Clinical Intelligence (2:00pm - 6:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| `agent/prompts.py` | Person A | 60 min | System prompt with clinical guidelines, user message template |
| `agent/llm.py` | Person A | 45 min | vLLM client, JSON parsing, retry logic |
| `agent/clinical.py` | Person A | 120 min | Intake → K2 analysis → validate output → return assessment + aftercare. Include safety bounds checking (never recommend if contraindicated). |
| `ClinicalAlerts.tsx` | Person C | 60 min | Color-coded alert cards (red/amber/green), expandable detail |
| `AftercarePlan.tsx` | Person C | 60 min | Medical aftercare + warning signs + privacy guidance, print-optimized |
| `PrivacyGuide.tsx` | Person C | 45 min | State-specific privacy guidance panel with purple styling |
| Wire intake → analysis → display | Person B | 60 min | End-to-end: form submit → session store → K2 call → WebSocket → dashboard |

**Milestone**: Submit intake for Maria Santos → K2 detects corticosteroid interaction → aftercare plan generated with Texas privacy guidance.

### Phase 3: Demo Scenarios (6:00pm - 10:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| Create all 3 synthetic patients | Person B | 60 min | JSON files for Maria Santos, Jennifer Park, Ashley Williams |
| Prompt tuning — Scenario 1 | Person A | 60 min | Ensure K2 catches corticosteroid interaction for Maria Santos |
| Prompt tuning — Scenario 2 | Person A | 90 min | Ensure K2 catches IUD critical finding + diabetes alert for Jennifer Park (hardest scenario) |
| Prompt tuning — Scenario 3 | Person A | 30 min | Verify clean assessment + Georgia privacy guidance for Ashley Williams |
| `SessionTimer.tsx` | Person C | 45 min | Countdown timer to auto-purge, manual "End Session & Purge" button with confirmation |
| Print stylesheet for aftercare | Person D | 45 min | Aftercare plan prints cleanly on paper (clinics hand these to patients) |
| Demo script + talking points | Person D | 30 min | Exact demo flow for judges |

**Milestone**: All 3 scenarios produce correct clinical assessments. Aftercare plans are printable. Session purge works.

### Phase 4: Polish + Demo Prep (10:00pm - 11:00am, with sleep)

| Task | Owner | Est. Time |
|------|-------|-----------|
| Error handling | Person B+D | 90 min |
| Dashboard visual polish | Person C | 60 min |
| Video recording (30 sec) | All | 60 min |
| Demo rehearsal (3+ runs) | All | 120 min |
| Devpost submission | Person A | 30 min |
| README + architecture diagram | Person B | 30 min |
| Sleep (rotating) | All | 3-4 hours each |

---

## 11. Demo Storyboard (5-7 min at judging)

**[0:00-0:45] The Problem**
"In 2022, prosecutors in Nebraska subpoenaed Facebook messages to convict a teenager for obtaining abortion medication. Her mother is now in prison. In Texas, search history for 'misoprostol dosage' has been used as evidence. In Mississippi, text messages. Every digital tool a reproductive health clinic uses — their EHR, their telehealth platform, their patient portal — creates records that prosecutors can subpoena. HIPAA does not protect against court orders. And the clinics know this. Some have gone back to paper records. Others collect minimal data, which degrades care quality. There is no technological solution that lets clinics use AI without creating the digital trail that puts their patients at risk."

**[0:45-1:30] The Solution**
"This is HavenAI. It runs entirely on this device [point to GX10] — a 70-billion parameter reasoning model providing clinical decision support with zero cloud connectivity. Patient data is held in RAM only. When the session ends [press End Session button], it's gone. Not encrypted, not archived — purged. There is nothing to subpoena, nothing to seize, nothing to hand over. The device sits in the clinic, on no network, air-gapped by design."

**[1:30-3:30] Live Demo — Patient Scenario**
"Let me show you a patient scenario. Maria Santos, 24, traveling from Texas — which has a near-total ban — to a clinic in New Mexico, which has a shield law.

[Walk through intake form — demographics, medical history showing asthma + prednisone, medications, request]

The clinician submits the intake. HavenAI analyzes...

[Show assessment appearing]

Look — it flagged a moderate alert. Maria is on Prednisone, a corticosteroid. Mifepristone is a glucocorticoid receptor antagonist — it can reduce corticosteroid efficacy. This isn't a contraindication, but it requires monitoring. A busy clinician processing 30 patients a day might miss this interaction. HavenAI catches it.

Now look at the aftercare plan. There's the standard medical aftercare — medications, expected symptoms, warning signs. But there's also a privacy section, generated specifically because Maria's home state is Texas: don't search for abortion terms on personal devices, use Signal for follow-up, if you need emergency care present your symptoms not the cause.

[Show print view] This prints cleanly — the clinic hands this to the patient on paper. No digital copy leaves the building."

**[3:30-4:15] Scenario 2 — Critical Finding**
"Here's a harder case. Jennifer Park, 31, IUD in place but strings not visible on self-exam.

[Quick intake, show assessment]

The agent flagged this as CRITICAL. An IUD must be confirmed removed before medication abortion. Non-visible strings may indicate the IUD has migrated. This changes the entire care pathway — she needs an ultrasound first. This is the kind of multi-step clinical reasoning that justifies a 70B model. A simple lookup table wouldn't catch this."

**[4:15-4:45] Session Purge**
"Now watch this. [Press End Session & Purge All Data. Confirmation dialog. Confirm.]

The session is gone. The patient data was in RAM only — never written to disk. If someone walked in with a subpoena right now, there is nothing on this device. That's the point."

**[4:45-5:15] Close**
"100 million women of reproductive age live under the shadow of Dobbs. Clinics need AI tools but can't risk the digital trail. HavenAI gives them clinical intelligence with zero data persistence. One device. No cloud. No trail. Privacy by architecture."

### 30-Second Video Script
- [0-5s] Text on black: "Prosecutors used Facebook messages to convict a teenager. (Nebraska, 2022)"
- [5-8s] Text: "Search history. Location data. Cloud records. All subpoena-able."
- [8-13s] Show GX10 device + dashboard intake form
- [13-18s] Clinical assessment appearing: contraindication alert flagged, aftercare plan generated
- [18-23s] Privacy guidance panel: "Texas — Do not search abortion terms on personal devices"
- [23-27s] Session purge animation: data disappearing, "Session Destroyed — Zero Data Remains"
- [27-30s] "HavenAI — Privacy by architecture. Built with K2 on ASUS GX10."

---

## 12. Prize Track Alignment

| Track | Fit | Why |
|-------|-----|-----|
| **Grand Prize** | Strong | Technically impressive, addresses defining societal issue of the decade |
| **Societal Impact (ASUS)** | Excellent | Reproductive healthcare access, 100M+ women affected, runs on ASUS hardware |
| **Best Use of K2 Think V2** | Excellent | Multi-step clinical reasoning (drug interactions, contraindication cascades, protocol adherence) is core. Not a chatbot. |
| **Built with Zed** | Free | Use Zed as IDE. |

---

## 13. Cut List (if behind schedule)

| Priority | Feature | Time saved | Impact |
|----------|---------|------------|--------|
| 1st | Print stylesheet for aftercare | 45 min | Low — judges see on screen |
| 2nd | Scenario 3 (Ashley Williams — straightforward) | 30 min | Low — two scenarios sufficient |
| 3rd | Session auto-timeout (keep manual purge only) | 30 min | Low |
| 4th | State law knowledge base (hardcode Texas/Georgia in prompt) | 45 min | Low for demo |
| 5th | Drug interaction database (embed in prompt instead) | 45 min | Low for demo |
| **NEVER** | Contraindication detection (Scenario 1) | — | Core clinical value |
| **NEVER** | Critical finding detection (Scenario 2 — IUD) | — | Proves multi-step reasoning |
| **NEVER** | Privacy guidance generation | — | Core differentiator |
| **NEVER** | Session purge functionality | — | The whole privacy story |

---

## 14. Fallback Chain (if K2 70B fails on GX10)

1. DGX OS vLLM playbook (pre-configured)
2. llama.cpp with GGUF quantization of K2-V2
3. `kaitchup/K2-V2-Instruct-GPTQ-4bit`
4. Qwen2.5-14B-Instruct (fast, good at structured JSON)
5. Remote K2 API if MBZUAI sponsor provides (compromise on air-gapped pitch)

---

## 15. Key Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| vLLM won't run on GX10 | Medium | Critical | Fallback chain. Test in first 30 min. |
| K2 gives clinically inaccurate output | Medium | High | Prompt includes explicit clinical guidelines. All output validated against safety bounds in code. "Clinician reviews" is part of the workflow — AI assists, doesn't decide. |
| K2 misses contraindication | Medium | High | Embed key contraindications directly in prompt. Test each scenario 5+ times during prompt tuning. |
| Sensitive topic makes judges uncomfortable | Low | Medium | Frame as healthcare technology + privacy engineering. Focus on technical architecture, not politics. The Nebraska case is factual, not partisan. |
| JSON parsing failures | High | Medium | Robust parsing, strip code fences, retry 2x, validate schema. |
| GX10 networking issues | Low | None | This product needs ZERO network. Everything runs on localhost. |

---

## 16. Dependencies

### Python Backend
```
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
websockets>=12.0
openai>=1.0.0
pydantic>=2.0
```

### React Dashboard
```bash
npx create-vite dashboard --template react-ts
npm install recharts
```

### vLLM on GX10
```bash
vllm serve LLM360/K2-V2-Instruct --dtype float16
```

---

## 17. Ethical and Legal Notes

- HavenAI is a **clinical decision support tool**, not a replacement for clinical judgment. A licensed clinician is always present and makes all care decisions.
- All demo data is **synthetic**. No real patient data is used.
- The product's architecture (air-gapped, ephemeral sessions) is specifically designed to protect patient privacy in the post-Dobbs legal environment.
- The product operates within existing legal frameworks — it does not facilitate illegal activity. It provides clinical support in states where reproductive healthcare is legal.
- Clinical guidelines embedded in the system are sourced from ACOG, WHO, and NAF published protocols.
- The privacy guidance provided to patients is informational, not legal advice.

---

## 18. Funding and Market Notes

### Target Market
- **Planned Parenthood**: ~600 health centers (post-Dobbs, concentrated in access states)
- **Independent clinics**: Hundreds of NAF member clinics
- **Telehealth providers**: Hey Jane, Carafem, Aid Access
- **Sexual health clinics**: Broader market beyond abortion care

### Revenue Model
- Sell GX10 + software as a turnkey appliance to clinics ($3K hardware + software license)
- Grant-funded deployments for under-resourced clinics

### Key Funders
- **Susan Thompson Buffett Foundation**: Largest private funder of reproductive health (~$500M+/year)
- **David and Lucile Packard Foundation**: $30-50M annually
- **William and Flora Hewlett Foundation**: Major reproductive rights funder
- **MacKenzie Scott**: $275M+ to reproductive rights orgs in 2022-2023
- **Digital Defense Fund**: Directly provides tech support to abortion clinics
- **Mozilla Foundation**: Funded privacy audits of reproductive health apps
- **Total available funding**: $2-3 billion annually in reproductive health philanthropy; technology is a growing category
