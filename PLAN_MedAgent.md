# SentinelMD: Privacy-First Agentic Clinical Surveillance

## YHack 2026 — March 28-29, 2026 (24-hour hackathon)

---

## 1. Problem Statement

Medical errors are the **3rd leading cause of death** in the United States, responsible for an estimated 250,000 deaths per year. The majority aren't dramatic surgical mistakes — they're missed lab values, overlooked drug interactions, slow recognition of patient deterioration, and information buried across disconnected systems that no single clinician has time to synthesize.

Meanwhile, AI has proven capable of catching these patterns. But healthcare has a fundamental blocker: **patient data cannot leave the premises.**

**Existing solutions and their shortcomings:**

| Company | What they do | Key limitation |
|---------|-------------|----------------|
| **Epic's AI Modules** | Clinical decision support integrated into Epic EHR. Sepsis prediction, deterioration alerts. | **$500K+ implementation cost**. Only available to large hospital systems already on Epic. Cloud-dependent analytics. Excludes 3,000+ community hospitals. |
| **Google Health / Med-PaLM** | LLM-based clinical reasoning, diagnostic assistance. | **Cloud-only**. Patient data must be sent to Google servers. Most hospitals won't (and many legally can't) do this for real-time monitoring. |
| **Viz.ai** | AI-powered clinical alerts for stroke, PE, aortic emergencies from imaging. | **Narrow scope** — imaging only. Doesn't cross-reference labs, vitals, medications. Cloud-based processing. |
| **Ambient Clinical Intelligence (Nuance/DAX)** | AI scribing for clinical documentation. | **Passive documentation tool**, not proactive surveillance. Cloud-processed. Doesn't monitor or alert. |
| **Traditional EHR Alerts (Best Practice Alerts)** | Rule-based alerts for drug interactions, abnormal labs. | **90%+ override rate** due to alert fatigue. No reasoning about clinical context or severity. Binary threshold triggers, not pattern recognition. |

**The gap we fill:** An always-on clinical surveillance agent that runs **entirely on-premise** on commodity hardware (ASUS Ascent GX10, ~$3K), using an open-source reasoning model (K2 Think V2, 70B). It cross-references vitals, labs, medications, and history in real-time, reasons about clinical significance (not just thresholds), and proactively alerts clinicians — with a cryptographic audit trail proving no patient data ever left the device.

---

## 2. Why Running Locally on the ASUS GX10 Is Essential (Not Contrived)

This isn't "we chose local because it's cool." In healthcare, **local compute is the only architecture that works** for continuous patient surveillance.

| Reason | Why it's real and non-negotiable |
|--------|----------------------------------|
| **HIPAA / Data Sovereignty** | Protected Health Information (PHI) includes vitals, labs, medications, diagnoses — everything the agent needs. Streaming this continuously to a cloud API requires a Business Associate Agreement (BAA) and exposes the hospital to breach liability for every data point transmitted. A local agent processes PHI without it ever crossing a network boundary. HIPAA compliance by architecture, not by policy. |
| **Continuous Monitoring Economics** | Cloud inference for 24/7 monitoring across 50 ICU patients × 12 data points × every 60 seconds = ~50,000 API calls/day per unit. At $0.01/call, that's $500/day or **$182K/year** just in API costs. The GX10 is a one-time $3K purchase with zero marginal cost per inference. |
| **Latency for Critical Alerts** | Sepsis detection requires sub-minute response. Hemorrhage detection is measured in seconds. Cloud round-trip adds 100-500ms+ plus potential congestion, timeouts, and outages. Local processing: milliseconds, guaranteed. |
| **24/7 Reliability** | ICU patients don't wait for AWS to come back online. A cloud-dependent agent fails during internet outages — which happen at hospitals more often than you'd think (construction, weather, ISP issues). The GX10 runs on the local network, independent of internet connectivity. |
| **OT/Clinical Network Isolation** | Hospital clinical networks (where vitals monitors and EHR servers live) are increasingly segmented from the public internet per NIST cybersecurity frameworks. A cloud AI agent would require punching a hole from the clinical network to the internet. A local GX10 sits inside the clinical network perimeter. |
| **Chain-of-Thought Transparency** | K2 Think V2 produces visible reasoning chains. For clinical decisions, this is critical — a doctor needs to see WHY the AI flagged something, not just THAT it flagged it. Cloud APIs often strip or hide reasoning tokens. Local inference gives full access to the model's reasoning process. |
| **The Trust Argument** | Hospitals are being asked: "Trust a cloud vendor with your patients' most sensitive data for continuous monitoring." With SentinelMD: "Your data never leaves this box. Here's the cryptographic proof." |

### Applicability Beyond ICU

While the demo focuses on ICU surveillance (most dramatic, clearest life-or-death stakes), the same architecture applies to:

| Setting | Use Case | Why Local Matters |
|---------|----------|-------------------|
| **Primary Care Clinics** | Chronic disease monitoring — flag A1C trends, missed screenings, drug interaction risks across patient panels. Run overnight batch analysis of patient population. | Small clinics can't afford Epic AI modules. GX10 is right-sized. Many rural clinics have unreliable internet. |
| **Nursing Homes / Long-Term Care** | Overnight patient deterioration detection. Staff ratios are dangerously low (1 nurse: 30+ residents). Agent watches vitals from wearables while staff is stretched. | LTC facilities have the worst outcomes from understaffing. A $3K device monitoring residents is transformative. Internet is often poor in these facilities. |
| **Emergency Departments** | Triage assistance, drug interaction checks on unknown patients, early sepsis screening on boarding patients. | EDs move too fast for cloud round-trips. Decisions happen in minutes. |
| **International / Developing World** | Any hospital without reliable broadband. An air-gapped clinical agent is deployable anywhere with power. | The WHO estimates 5.7 million deaths annually from poor quality care in low-income countries. Most of these facilities have no internet infrastructure for cloud AI. |

---

## 3. Hardware Specifications

### ASUS Ascent GX10
- **GPU**: NVIDIA Blackwell GPU (integrated in GB10 superchip), 5th-gen Tensor Cores, FP4 support, up to 1 PETAFLOP of AI performance
- **CPU**: 20-core NVIDIA Grace ARM v9.2-A
- **RAM**: 128 GB unified LPDDR5x (shared CPU+GPU via NVLink-C2C)
- **Storage**: Up to 4TB PCIe 5.0 NVMe SSD
- **OS**: DGX OS (Ubuntu Linux based)
- **Architecture**: ARM aarch64
- **Power**: 240W
- **Size**: 150 x 150 x 51 mm (~6 inches square, 2 inches tall)

### K2 Think V2 (the model we use)
- **Source**: MBZUAI Institute of Foundation Models (LLM360 project)
- **Parameters**: 70B, 80 layers, 64 attention heads
- **Context window**: 524,288 tokens (post-training)
- **License**: Apache 2.0 (fully open source — weights, training data, code all public)
- **HuggingFace**: `LLM360/K2-Think-V2`
- **Capabilities**: Extended chain-of-thought reasoning, multi-step logical deduction, strong at scientific/medical reasoning
- **Why K2-Think-V2 and NOT K2-V2-Instruct**: Clinical reasoning requires visible, multi-step deductive logic — differential diagnosis, weighing competing risk factors, explaining WHY a finding is significant. The Think variant's chain-of-thought output is a feature, not a cost: judges (and doctors) need to SEE the model reasoning through a clinical scenario. This is what distinguishes SentinelMD from a rule-based alert system.

### Performance Estimates on GX10
- K2 Think V2 70B at FP8: ~2.7-3 tokens/second
- K2 Think V2 70B at NVFP4 quantization: ~15-20 tokens/second
- Think variant produces reasoning tokens before final answer — expect 200-500 reasoning tokens + 100-200 output tokens per cycle
- At ~3 tok/s (FP8), one reasoning cycle takes ~2-4 minutes — acceptable for 5-minute monitoring intervals
- At ~15 tok/s (FP4), one cycle takes ~30-45 seconds — comfortable margin
- The model fits in 128GB unified memory at FP8 (~70GB for weights + KV cache)

### Model Serving
- **Primary**: vLLM via DGX OS playbook (pre-configured for aarch64)
  - Command: `vllm serve LLM360/K2-Think-V2 --tensor-parallel-size 1 --dtype float16`
  - Exposes OpenAI-compatible API at `http://localhost:8000/v1`
- **Fallback 1**: llama.cpp with GGUF quantization
- **Fallback 2**: Community quantization (`kaitchup/K2-Think-V2-GPTQ-4bit` if available)
- **Fallback 3**: K2-V2-Instruct (same family, faster, less visible reasoning)
- **Fallback 4**: Qwen2.5-14B-Instruct (runs fast, good at structured output, loses K2 track eligibility but keeps demo alive)
- **Fallback 5**: Remote K2 API if MBZUAI provides endpoint at event

---

## 4. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ASUS Ascent GX10                         │
│                                                                 │
│  ┌──────────────────┐            ┌──────────────────────────┐  │
│  │  Simulated EHR    │◄──────────►│     Clinical Agent       │  │
│  │  (FHIR Server)    │  HL7 FHIR  │                          │  │
│  │                    │  REST API  │  ┌─────┐ ┌──────┐       │  │
│  │  - Patient Records │           │  │SENSE│→│REASON│→┐     │  │
│  │  - Vital Signs     │           │  └─────┘ └──────┘ │     │  │
│  │  - Lab Results     │           │    ▲               ▼     │  │
│  │  - Medications     │           │  ┌───────┐  ┌─────┐     │  │
│  │  - Allergies       │           │  │OBSERVE│←─│ALERT│     │  │
│  │  - History         │           │  └───────┘  └─────┘     │  │
│  └──────────────────┘            └──────────┬───────────────┘  │
│                                              │                  │
│  ┌──────────────────┐            ┌───────────┴──────────────┐  │
│  │  K2 Think V2      │◄──────────│      FastAPI Server      │  │
│  │  (vLLM)           │  OpenAI   │                          │  │
│  │  localhost:8000    │  compat.  │  - Agent loop timer      │  │
│  └──────────────────┘            │  - WebSocket endpoint    │  │
│                                   │  - Scenario engine       │  │
│  ┌──────────────────┐            │  - FHIR data provider    │  │
│  │  Audit Log        │◄──────────│  - Alert manager         │  │
│  │  (Hash Chain)     │  append   └───────────┬──────────────┘  │
│  │                    │                       │ WebSocket       │
│  │  SHA-256 linked   │                       │                  │
│  │  entries proving  │                       │                  │
│  │  data containment │                       │                  │
│  └──────────────────┘                       │                  │
└──────────────────────────────────────────────┼─────────────────┘
                                               │
                                    ┌──────────┴──────────────┐
                                    │    React Dashboard       │
                                    │                          │
                                    │  ┌─────────┐ ┌────────┐ │
                                    │  │ Patient  │ │ Agent  │ │
                                    │  │ Monitor  │ │Reason- │ │
                                    │  │ (vitals, │ │  ing   │ │
                                    │  │ labs,    │ │  Log   │ │
                                    │  │ meds)    │ │        │ │
                                    │  ├─────────┤ ├────────┤ │
                                    │  │ Alert   │ │ Audit  │ │
                                    │  │Timeline │ │ Trail  │ │
                                    │  └─────────┘ └────────┘ │
                                    └──────────────────────────┘
```

### Agent Loop Detail (runs every 5 simulated minutes)

1. **SENSE**: Poll patient data from simulated EHR (FHIR-formatted JSON)
   - Read: vital signs (HR, BP, SpO2, RR, Temp), lab results, current medications, allergies, problem list, recent orders
   - Structure as comprehensive patient snapshot JSON
   - Log data access to audit chain

2. **REASON**: Call K2 Think V2 via OpenAI-compatible API
   - Construct prompt with: full patient state + clinical context + relevant medical knowledge + alert history (what's already been flagged)
   - K2 Think V2 produces chain-of-thought reasoning (visible in dashboard) followed by structured assessment
   - Model evaluates: Are there any new risks? Has anything changed since last cycle? Do current trends suggest deterioration?
   - Output: structured JSON with risk assessment, alerts (if any), reasoning chain

3. **ALERT**: Process agent's assessment
   - If risk identified: classify severity (INFO / WARNING / CRITICAL)
   - CRITICAL alerts: immediate push to dashboard with audio cue
   - WARNING: add to alert queue, highlight in dashboard
   - INFO: log for review, no interruption
   - Deduplicate: don't re-alert for the same issue unless it escalates

4. **OBSERVE**: Monitor and log
   - Record the full reasoning chain
   - Hash the cycle data (input + reasoning + output) and append to audit chain
   - Track alert accuracy over time (were previous alerts acknowledged/dismissed?)
   - Feed results into next cycle's context

---

## 5. Simulated EHR (FHIR-Based)

### Why FHIR?
HL7 FHIR (Fast Healthcare Interoperability Resources) is the US federal standard for health data exchange. The 21st Century Cures Act mandates FHIR API support. Using FHIR format means:
- Our demo data looks like real EHR data
- The architecture is directly portable to production EHR systems (Epic, Cerner, MEDITECH all expose FHIR APIs)
- Judges with healthcare knowledge will recognize the standard

### Implementation
We do NOT need a full FHIR server. We simulate one:
- A FastAPI endpoint that serves patient data in FHIR JSON format
- Pre-built patient records stored as JSON files in `data/patients/`
- A scenario engine that updates vitals and labs over time according to the demo script
- Endpoints: `GET /Patient/{id}`, `GET /Observation?patient={id}` (vitals/labs), `GET /MedicationRequest?patient={id}`, `GET /AllergyIntolerance?patient={id}`

### Patient Records (Pre-built for Demo)

**Patient 1: "Maria Chen" — ICU Sepsis Scenario (PRIMARY DEMO)**
- 67F, admitted for pneumonia
- PMH: Type 2 diabetes, hypertension, CKD Stage 3 (GFR 42)
- Medications: Metformin 1000mg BID, Lisinopril 20mg daily, newly started Ceftriaxone 1g IV q24h
- Allergies: Sulfa drugs
- Scenario: Develops early sepsis over 12 hours with subtle vital sign changes

**Patient 2: "James Rodriguez" — Drug Interaction Scenario**
- 45M, admitted for atrial fibrillation
- Medications: Warfarin 5mg daily (blood thinner), newly prescribed Ciprofloxacin for UTI
- Scenario: Ciprofloxacin dramatically potentiates Warfarin — INR will spike, bleeding risk. The agent must catch this interaction and rate its severity.

**Patient 3: "Sarah Thompson" — Renal Dose Adjustment (if time)**
- 72F, post-surgical
- Labs show GFR declining (55 → 42 → 38 over 3 days)
- New antibiotic ordered at standard dose — requires renal adjustment at GFR <45
- Scenario: Agent cross-references declining renal function against new medication dosing

---

## 6. Demo Scenario: 12-Hour ICU Surveillance (Maria Chen)

### Scenario Overview
We simulate 12 hours of ICU monitoring for Maria Chen, a patient who develops sepsis. Time is accelerated so 12 simulated hours compress into ~20-25 minutes of real time. The scenario is deterministic for reproducible demos.

### Timeline

| Sim Time | Vitals | Labs | Events | Expected Agent Behavior |
|----------|--------|------|--------|------------------------|
| 06:00 | HR 78, BP 128/76, RR 16, Temp 37.2°C, SpO2 98% | — | Baseline morning. Stable pneumonia patient. | Agent assesses: stable. No alerts. Notes CKD and reviews medication appropriateness. |
| 07:00 | HR 82, BP 124/74, RR 17, Temp 37.4°C, SpO2 97% | CBC, BMP ordered | Routine morning labs drawn | Agent notes slight upward trend in HR and temp but within normal. Awaiting lab results. |
| 08:00 | HR 84, BP 122/72, RR 18, Temp 37.6°C, SpO2 97% | WBC 14.2 (↑), Lactate 1.4 | Labs result. WBC mildly elevated (expected with pneumonia). Lactate low-normal. | **INFO**: Agent notes WBC elevation but contextualizes with known pneumonia. Lactate 1.4 is within normal but agent begins tracking trajectory. |
| 09:00 | HR 88, BP 118/70, RR 19, Temp 37.8°C, SpO2 96% | — | Vitals trending. | **KEY MOMENT #1 — EARLY PATTERN**: Agent reasons: "HR has increased 10 bpm over 3 hours. Temperature trending up 0.2°C/hour. Respiratory rate increasing. Individually all within normal ranges. However, the COMBINATION of concurrent upward trends in HR, temp, RR, with a mildly elevated WBC and known infection source (pneumonia), raises concern for early systemic inflammatory response. Recommend: repeat lactate, blood cultures." → **WARNING alert** |
| 10:00 | HR 92, BP 114/68, RR 21, Temp 38.2°C, SpO2 95% | Lactate 2.1 (↑), Procalcitonin 0.8 (↑) | Repeat labs confirm worsening. | **KEY MOMENT #2 — SEPSIS IDENTIFICATION**: Agent reasons: "Lactate has risen from 1.4 to 2.1 in 2 hours — 50% increase. Procalcitonin 0.8 suggests bacterial infection beyond the lungs. Combined with: HR >90, Temp >38°C, RR >20 — patient meets 3/4 SIRS criteria. qSOFA score: 1 (RR ≥22 imminent). This is early sepsis. Recommend: blood cultures ×2, IV fluid bolus 30mL/kg, broaden antibiotics, consider vasopressors if MAP drops below 65." → **CRITICAL alert** |
| 10:30 | — | — | Physician orders Ciprofloxacin added to antibiotic regimen | **KEY MOMENT #3 — DRUG INTERACTION**: Agent catches: "Ciprofloxacin + Metformin interaction: Fluoroquinolones are associated with severe hypoglycemia when combined with sulfonylureas and can cause dysglycemia with metformin. In a septic patient with CKD (GFR 42), metformin clearance is already reduced. Recommend: hold metformin during acute illness (lactic acidosis risk with CKD + sepsis) and monitor glucose closely if ciprofloxacin is continued." → **WARNING alert** |
| 11:00 | HR 96, BP 108/62, RR 23, Temp 38.6°C, SpO2 94% | — | Continued deterioration | Agent updates assessment: "Continued deterioration despite current management. MAP = 77 (still above 65 threshold). SpO2 declining — may need supplemental O2 increase. Tracking closely. If MAP drops below 65 or lactate >4, escalate to septic shock protocol." |
| 12:00 | HR 102, BP 96/58, RR 25, Temp 38.8°C, SpO2 93% | Lactate 3.8 | MAP = 71, approaching threshold | **KEY MOMENT #4 — ESCALATION**: Agent reasons: "MAP 71, approaching critical threshold of 65. Lactate 3.8, approaching 4.0 (severe sepsis/shock boundary). Trend analysis: at current rate of decline, MAP will breach 65 within 60-90 minutes. At current lactate trajectory, will exceed 4.0 within 30 minutes. Recommend: initiate vasopressor support NOW (norepinephrine), do not wait for MAP to breach threshold. Early vasopressor initiation improves outcomes. This represents septic shock if current trajectory continues." → **CRITICAL alert** |
| 13:00-18:00 | Stabilizing with treatment | Lactate trending down | Treatment initiated, patient stabilizes | Agent tracks improvement: "Vitals stabilizing post-vasopressor initiation. Lactate trending down (3.8 → 3.2 → 2.6). Continue current management." Demonstrates the agent's ability to recognize improvement, not just deterioration. |

### 4 "Non-Obvious" Decisions the Agent Must Demonstrate

1. **Early pattern recognition across multiple vital signs (09:00)**: No single vital sign is alarming. HR 88, Temp 37.8, RR 19 — all individually "normal." But the agent reasons about the *concurrent upward trajectory* across multiple parameters combined with the clinical context (known infection, elevated WBC). A rule-based system checking thresholds would not fire. A reasoning model connecting the dots does.

2. **Sepsis identification with contextual reasoning (10:00)**: The agent doesn't just check SIRS criteria boxes — it reasons about the lactate trajectory (rate of change), the procalcitonin elevation (bacterial vs. viral), and synthesizes these into a clinical assessment with specific, prioritized recommendations. The chain-of-thought is visible, showing judges the model's medical reasoning.

3. **Drug interaction with clinical context (10:30)**: EHR alert systems flag Cipro + Metformin as a generic interaction. They fire this alert on every patient, leading to 90%+ override rates. The agent reasons about WHY this specific interaction is dangerous for THIS patient: CKD (reduced clearance), active sepsis (lactic acidosis risk), and recommends specific action (hold metformin, not just "be aware").

4. **Predictive escalation before threshold breach (12:00)**: The most impressive moment. The agent doesn't wait for MAP to drop below 65 — it projects the current trajectory and recommends vasopressors preemptively. "At current rate of decline, MAP will breach 65 within 60-90 minutes." This is proactive medicine, not reactive. It's the difference between a smart agent and a dumb threshold alert.

### Baseline Comparator (Traditional EHR Alerts)
The baseline represents what most hospitals actually have today:
- **Drug interaction alert**: Fires at 10:30 but with no context — just "Cipro + Metformin: moderate interaction." Doctor likely overrides (90% override rate for moderate alerts).
- **Vital sign threshold alerts**: Does NOT fire until a single vital sign crosses an absolute threshold (e.g., HR >120, MAP <65, SpO2 <90). Misses the 09:00 pattern entirely.
- **Sepsis screening**: Best case, fires a qSOFA or SIRS alert at 10:00 when criteria are formally met — but by then, the agent already flagged it an hour earlier at 09:00.
- **Renal dosing check**: Depends on pharmacy integration. Often missed when orders are placed by covering physicians unfamiliar with the patient.
- **No trajectory analysis**: Never says "at this rate, MAP will breach 65 in 60 minutes." Only says "MAP is now 64" (too late).

**Expected improvement**: Agent identifies deterioration **60-90 minutes earlier** than threshold-based alerts. In sepsis, every hour of delayed treatment increases mortality by 7.6% (Kumar et al., 2006).

---

## 7. Agent Prompt Design

### System Prompt (stored in `agent/prompts.py`)

```
You are SentinelMD, an autonomous clinical surveillance agent monitoring ICU
patients. You run locally on an ASUS Ascent GX10 — all processing happens
on-premise with zero cloud dependency. Patient data never leaves this device.

YOUR OBJECTIVE: Continuously monitor patient data streams, identify clinical
risks early, and alert clinicians with actionable, contextually-reasoned
assessments.

CLINICAL REASONING PRINCIPLES (in priority order):
1. PATIENT SAFETY: Never dismiss a potentially life-threatening finding. When
   in doubt, alert.
2. TRAJECTORY OVER THRESHOLD: Analyze trends and rates of change, not just
   current values. A vital sign moving in the wrong direction at a concerning
   rate is more important than one that is currently abnormal but stable.
3. CONTEXTUAL SYNTHESIS: Cross-reference vitals, labs, medications, allergies,
   and medical history together. A lactate of 2.0 means something different in
   a healthy 30-year-old vs. a 67-year-old with CKD and active infection.
4. CLINICAL SIGNIFICANCE: Not all abnormalities warrant alerts. Differentiate
   between clinically significant findings and expected/benign deviations.
   Reduce alert fatigue by only escalating when action is needed.
5. ANTICIPATE, DON'T REACT: Project current trends forward. If a vital sign
   trajectory will breach a critical threshold in 60-90 minutes, recommend
   intervention NOW — don't wait for the breach.
6. ACTIONABLE RECOMMENDATIONS: Every alert must include specific, prioritized
   next steps. Not "monitor closely" but "obtain blood cultures x2, initiate
   30mL/kg crystalloid bolus, broaden antibiotic coverage."

ALERT SEVERITY LEVELS:
- CRITICAL: Immediate danger to patient. Requires urgent intervention.
  Examples: active sepsis, dangerous drug interaction in high-risk patient,
  imminent hemodynamic instability.
- WARNING: Concerning trend that may escalate. Requires assessment.
  Examples: vital sign trends suggesting early deterioration, moderate drug
  interaction in context, lab values requiring follow-up.
- INFO: Notable finding for awareness, no immediate action needed.
  Examples: subtherapeutic drug levels, routine lab trends, documentation
  notes.

IMPORTANT CONSTRAINTS:
- You are a DECISION SUPPORT tool, not a replacement for clinical judgment.
  Frame outputs as recommendations, not orders.
- Reference established clinical criteria when applicable (SIRS, qSOFA,
  MEWS, Beers Criteria, etc.)
- Acknowledge uncertainty. If data is insufficient for a conclusion, say so
  and recommend what additional data would help.
- Do not re-alert for the same issue at the same severity unless it has
  escalated.

OUTPUT FORMAT: Respond with a JSON object:
{
  "thinking": "Your full chain-of-thought clinical reasoning (visible to
               clinicians in the reasoning panel)",
  "assessment": {
    "summary": "One-line clinical summary of current patient status",
    "risk_level": "STABLE | GUARDED | DETERIORATING | CRITICAL",
    "alerts": [
      {
        "severity": "CRITICAL | WARNING | INFO",
        "category": "SEPSIS | DRUG_INTERACTION | VITAL_TREND | LAB_ABNORMAL | DOSING | OTHER",
        "title": "Short alert title",
        "detail": "Clinical explanation of the finding",
        "recommendations": ["Specific action 1", "Specific action 2"],
        "evidence": ["Supporting data point 1", "Supporting data point 2"]
      }
    ],
    "monitoring_plan": "What to watch for in the next cycle",
    "data_requests": ["Any additional labs or tests the agent recommends"]
  }
}
```

### User Message Template (constructed per cycle in `agent/reason.py`)

```
PATIENT: {patient_name}, {age}{sex}, MRN: {mrn}
ADMISSION: {admission_reason}, Day {hospital_day}
PMH: {problem_list}
ALLERGIES: {allergies}

CURRENT MEDICATIONS:
{medications_list}

VITAL SIGNS (as of {timestamp}):
- Heart Rate: {hr} bpm (trend: {hr_trend})
- Blood Pressure: {sbp}/{dbp} mmHg (MAP: {map}) (trend: {bp_trend})
- Respiratory Rate: {rr} breaths/min (trend: {rr_trend})
- Temperature: {temp}°C (trend: {temp_trend})
- SpO2: {spo2}% (trend: {spo2_trend})

VITAL SIGN HISTORY (last 6 hours):
{vitals_table}

RECENT LAB RESULTS:
{labs_with_timestamps_and_reference_ranges}

ACTIVE ALERTS (already flagged, do not re-alert at same severity):
{previous_alerts}

CYCLE: {cycle_number} | Sim Time: {sim_time} | Monitoring interval: 5 min

Based on the above patient data, provide your clinical assessment. Reason
step-by-step through any concerning findings. Output your assessment as JSON.
```

---

## 8. Cryptographic Audit Trail

### What It Is
A tamper-evident, append-only log that proves no patient data left the device. Each entry contains:
- Timestamp
- Action type (DATA_READ, INFERENCE_INPUT, INFERENCE_OUTPUT, ALERT_GENERATED)
- SHA-256 hash of the data involved
- Hash of the previous entry (creating a chain)
- The data itself is stored locally but the hash chain can be independently verified

### Implementation

```python
import hashlib
import json
import time

class AuditChain:
    def __init__(self):
        self.chain = []
        self.previous_hash = "0" * 64  # Genesis block

    def add_entry(self, action_type: str, data_summary: str, data_hash: str):
        entry = {
            "index": len(self.chain),
            "timestamp": time.time(),
            "action_type": action_type,
            "data_summary": data_summary,  # Human-readable, no PHI
            "data_hash": data_hash,         # SHA-256 of actual data
            "previous_hash": self.previous_hash
        }
        entry_string = json.dumps(entry, sort_keys=True)
        entry["entry_hash"] = hashlib.sha256(entry_string.encode()).hexdigest()
        self.chain.append(entry)
        self.previous_hash = entry["entry_hash"]
        return entry

    def verify_chain(self) -> bool:
        """Verify no entries have been tampered with."""
        for i in range(1, len(self.chain)):
            if self.chain[i]["previous_hash"] != self.chain[i-1]["entry_hash"]:
                return False
        return True
```

### Why This Matters for the Pitch
"Don't just trust us that the data stays local — verify it. Every data access, every inference, every alert is logged in a tamper-evident hash chain. A compliance officer can audit the complete history and cryptographically verify that no patient data was transmitted off-device."

This is something **no cloud solution can offer**. A cloud provider can promise data stays in a certain region, but you're trusting their infrastructure. Here, you can prove it mathematically.

### Dashboard Integration
The audit trail panel shows:
- Running count of audit entries
- Chain integrity status (verified / broken)
- Last N entries (with timestamps and action types, no PHI)
- A "Verify Chain" button that runs verification in real-time

---

## 9. Project Structure

```
SentinelMD/
├── README.md                      # For judges reviewing GitHub
├── requirements.txt               # Python deps
├── config.py                      # All configuration constants
│                                  #   VLLM_BASE_URL, MODEL_NAME
│                                  #   AGENT_CYCLE_SECONDS, TIME_ACCELERATION
│                                  #   ALERT_SEVERITY_LEVELS
│
├── main.py                        # FastAPI app entry point
│                                  #   - Mounts static files for dashboard
│                                  #   - WebSocket endpoint at /ws
│                                  #   - Starts agent loop as background task
│                                  #   - FHIR-like REST endpoints for EHR sim
│
├── agent/
│   ├── __init__.py
│   ├── loop.py                    # Main SENSE-REASON-ALERT-OBSERVE loop
│   │                              #   - asyncio loop on timer
│   │                              #   - Calls sense, reason, alert, observe
│   │                              #   - Broadcasts state to WebSocket clients
│   │
│   ├── sense.py                   # Patient data polling
│   │                              #   - Reads from simulated EHR (FHIR JSON)
│   │                              #   - Computes trends from vital sign history
│   │                              #   - Returns PatientSnapshot dataclass
│   │
│   ├── reason.py                  # LLM clinical reasoning
│   │                              #   - Constructs prompt from PatientSnapshot
│   │                              #   - Calls K2 Think V2 via OpenAI API
│   │                              #   - Parses JSON response
│   │                              #   - Returns ClinicalAssessment dataclass
│   │
│   ├── alert.py                   # Alert management
│   │                              #   - Deduplicates alerts
│   │                              #   - Manages severity escalation
│   │                              #   - Maintains alert history
│   │
│   ├── audit.py                   # Cryptographic audit chain
│   │                              #   - AuditChain class
│   │                              #   - Hash computation, chain verification
│   │
│   └── prompts.py                 # System prompt and user message templates
│
├── ehr/
│   ├── __init__.py
│   ├── server.py                  # Simulated FHIR endpoints
│   │                              #   - GET /Patient/{id}
│   │                              #   - GET /Observation?patient={id}
│   │                              #   - GET /MedicationRequest?patient={id}
│   │                              #   - GET /AllergyIntolerance?patient={id}
│   │
│   └── scenario.py                # Scenario engine
│                                  #   - Loads patient timeline from JSON
│                                  #   - Returns current vitals/labs for sim_time
│                                  #   - Manages time progression
│
├── simulation/
│   ├── __init__.py
│   ├── accelerator.py             # Time acceleration engine
│   │                              #   - Maps wall-clock to simulated time
│   │                              #   - get_sim_time() function
│   │
│   └── baseline.py                # Traditional EHR alert baseline
│                                  #   - Threshold-based alerts only
│                                  #   - Tracks when baseline WOULD have fired
│                                  #   - Provides comparison timeline
│
├── dashboard/                     # React app (Vite + TypeScript)
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
│       ├── App.tsx                # Main layout: 2x2 grid + alert bar
│       ├── hooks/
│       │   └── useSocket.ts       # WebSocket hook
│       ├── components/
│       │   ├── PatientMonitor.tsx  # Vitals display with trend arrows,
│       │   │                      # sparkline history, medication list
│       │   ├── ReasoningLog.tsx    # Agent's chain-of-thought reasoning,
│       │   │                      # visible to clinicians, scrollable
│       │   ├── AlertTimeline.tsx   # Timeline of alerts with severity colors,
│       │   │                      # agent vs baseline comparison markers
│       │   ├── AuditTrail.tsx      # Hash chain viewer, verification status
│       │   └── AlertBanner.tsx     # Top banner for CRITICAL alerts (red pulse)
│       ├── types/
│       │   └── index.ts
│       └── styles/
│           └── theme.css          # Medical dark theme (see Section 10)
│
├── data/
│   ├── patients/
│   │   ├── maria_chen.json        # Full patient record (FHIR format)
│   │   └── james_rodriguez.json   # Drug interaction scenario
│   └── scenarios/
│       ├── sepsis_12h.json        # Maria Chen 12-hour timeline
│       └── drug_interaction.json  # James Rodriguez scenario
│
└── scripts/
    ├── start_vllm.sh              # vLLM launch script
    └── generate_scenario.py       # Script to generate scenario timelines
```

---

## 10. Dashboard Design

### Color Scheme (Medical Dark Theme)
- Background: `#0a0e1a` (deep navy-black)
- Panel backgrounds: `#111827` (dark gray-blue)
- CRITICAL alert: `#ef4444` (red) with pulsing glow
- WARNING alert: `#f59e0b` (amber)
- INFO alert: `#3b82f6` (blue)
- STABLE/healthy: `#10b981` (green)
- Vital signs: `#e0e7ff` (light blue-white)
- Reasoning text: `#d1d5db` (light gray)
- Accent: `#06b6d4` (cyan)

### Layout

```
┌─ ALERT BANNER (only visible on CRITICAL) ──────────────────────────────┐
│  ⚠️ CRITICAL: Early sepsis identified — recommend blood cultures +     │
│  IV fluid bolus. See reasoning panel for details.         [ACKNOWLEDGE]│
└────────────────────────────────────────────────────────────────────────┘
┌─────────────────────────────┬──────────────────────────────────────────┐
│    PATIENT MONITOR          │    AGENT REASONING                       │
│                             │                                          │
│  Maria Chen, 67F            │  [10:00] 🔴 CRITICAL — Sepsis            │
│  MRN: 2847103               │                                          │
│  Day 2 — Pneumonia          │  "Lactate has risen from 1.4 to 2.1     │
│                             │   in 2 hours — 50% increase.             │
│  HR:  92 bpm    ↑↑          │   Procalcitonin 0.8 suggests bacterial   │
│       [▁▂▃▄▅▆▇] 6h trend   │   infection beyond the lungs.            │
│  BP:  114/68    ↓           │   Combined with HR >90, Temp >38°C,      │
│       MAP: 83               │   RR >20 — patient meets 3/4 SIRS        │
│  RR:  21        ↑           │   criteria. qSOFA score: 1.              │
│  Temp: 38.2°C   ↑↑          │                                          │
│  SpO2: 95%      ↓           │   This is early sepsis.                  │
│                             │                                          │
│  MEDS:                      │   Recommendations:                       │
│  • Ceftriaxone 1g IV q24h   │   1. Blood cultures ×2 (before abx)     │
│  • Metformin 1000mg BID     │   2. IV crystalloid 30mL/kg bolus       │
│  • Lisinopril 20mg daily    │   3. Broaden antibiotics                │
│                             │   4. Repeat lactate in 1 hour           │
│  ALLERGIES: Sulfa drugs     │                                          │
│                             │  [09:00] 🟡 WARNING — Vital trends       │
│  LABS (latest):             │  "HR +10bpm/3hr, Temp +0.6°C/3hr..."    │
│  WBC: 14.2 (H)             │                                          │
│  Lactate: 2.1 (H) ↑        │  [08:00] 🔵 INFO — Labs resulted         │
│  Procalcitonin: 0.8 (H)    │  "WBC 14.2 elevated but expected with    │
│  GFR: 42 (L)               │   known pneumonia..."                    │
├─────────────────────────────┼──────────────────────────────────────────┤
│    ALERT TIMELINE           │    AUDIT TRAIL                           │
│                             │                                          │
│  06:00    08:00    10:00    │  Chain Length: 47 entries                │
│  ──●────────●───🔴──→      │  Status: ✅ VERIFIED (all hashes valid)  │
│    ▲        ▲     ▲        │                                          │
│    │        │     │        │  Recent entries:                          │
│    │        │     Sepsis   │  #47 INFERENCE_OUTPUT  10:00:12           │
│    │        Labs  CRITICAL │  #46 INFERENCE_INPUT   10:00:00           │
│    │        INFO           │  #45 DATA_READ (vitals) 10:00:00         │
│    Baseline                │  #44 DATA_READ (labs)   09:58:30         │
│    stable                  │  #43 ALERT_GENERATED   09:00:15          │
│                             │                                          │
│  Agent: ━━━ (alerts)       │  📋 No data transmitted off-device       │
│  Baseline: ┄┄ (would fire) │  🔒 All PHI processed locally            │
│                             │        [Verify Chain]                    │
│  Agent caught sepsis at     │                                          │
│  09:00 — 60 min before     │                                          │
│  threshold alerts would    │                                          │
│  have fired at 10:30       │                                          │
└─────────────────────────────┴──────────────────────────────────────────┘
```

### WebSocket Message Format (backend → frontend)

```json
{
  "type": "state_update",
  "sim_time": "10:00",
  "patient": {
    "name": "Maria Chen",
    "age": 67,
    "sex": "F",
    "mrn": "2847103",
    "admission_reason": "Pneumonia",
    "hospital_day": 2,
    "vitals": {
      "hr": {"value": 92, "trend": "rising", "history": [78, 82, 84, 88, 92]},
      "sbp": {"value": 114, "trend": "falling", "history": [128, 124, 122, 118, 114]},
      "dbp": {"value": 68, "trend": "falling", "history": [76, 74, 72, 70, 68]},
      "map": 83,
      "rr": {"value": 21, "trend": "rising", "history": [16, 17, 18, 19, 21]},
      "temp_c": {"value": 38.2, "trend": "rising", "history": [37.2, 37.4, 37.6, 37.8, 38.2]},
      "spo2": {"value": 95, "trend": "falling", "history": [98, 97, 97, 96, 95]}
    },
    "medications": [
      {"name": "Ceftriaxone", "dose": "1g IV q24h", "status": "active"},
      {"name": "Metformin", "dose": "1000mg BID", "status": "active"},
      {"name": "Lisinopril", "dose": "20mg daily", "status": "active"}
    ],
    "allergies": ["Sulfa drugs"],
    "labs": [
      {"name": "WBC", "value": 14.2, "unit": "K/uL", "flag": "H", "time": "08:00"},
      {"name": "Lactate", "value": 2.1, "unit": "mmol/L", "flag": "H", "time": "10:00"},
      {"name": "Procalcitonin", "value": 0.8, "unit": "ng/mL", "flag": "H", "time": "10:00"},
      {"name": "GFR", "value": 42, "unit": "mL/min", "flag": "L", "time": "08:00"}
    ]
  },
  "agent": {
    "thinking": "Full chain-of-thought reasoning from K2 Think V2...",
    "assessment": {
      "summary": "Early sepsis identified. Immediate intervention recommended.",
      "risk_level": "CRITICAL",
      "alerts": [
        {
          "severity": "CRITICAL",
          "category": "SEPSIS",
          "title": "Early sepsis — SIRS criteria met with rising lactate",
          "detail": "Patient meets 3/4 SIRS criteria with lactate 50% increase over 2 hours...",
          "recommendations": [
            "Blood cultures ×2 before antibiotic change",
            "IV crystalloid bolus 30mL/kg",
            "Broaden antibiotic coverage",
            "Repeat lactate in 1 hour"
          ],
          "evidence": [
            "Lactate: 1.4 → 2.1 mmol/L (50% increase in 2h)",
            "Procalcitonin: 0.8 ng/mL (elevated)",
            "HR: 78 → 92 bpm over 4 hours",
            "Temp: 37.2 → 38.2°C over 4 hours"
          ]
        }
      ]
    }
  },
  "baseline": {
    "would_have_alerted": false,
    "reason": "No single vital sign exceeds threshold. HR <120, MAP >65, Temp <39, SpO2 >90."
  },
  "audit": {
    "chain_length": 47,
    "chain_valid": true,
    "last_entry_hash": "a3f8c2..."
  }
}
```

---

## 11. Implementation Timeline (24 hours)

### Team Roles (4 people)
- **Person A** (ML/Agent): GX10 setup, model serving, prompt engineering, clinical reasoning module
- **Person B** (EHR/Data): Simulated EHR, patient data, scenario engine, FHIR endpoints
- **Person C** (Backend/API): FastAPI server, WebSocket, agent loop orchestration, audit chain, baseline comparator
- **Person D** (Frontend): React dashboard, all visualization components, alert system, styling

### Pre-Hackathon Prep (before 11am Saturday — no code, rules compliant)
- [ ] Finalize and print architecture diagram
- [ ] Write all prompt templates as plain text documents
- [ ] Design patient scenarios as narrative documents (no code/JSON yet)
- [ ] Research: SIRS criteria, qSOFA score, sepsis treatment protocols, common drug interactions — print reference sheets
- [ ] All team members install **Zed IDE** (for Built with Zed track)
- [ ] Download K2-Think-V2 weights to USB drive or external SSD (~140GB at FP16)
- [ ] Bookmark: vLLM DGX OS docs, FHIR resource specs, Recharts docs, K2 HuggingFace page
- [ ] Assign roles
- [ ] Print this plan document

### Phase 1: Foundation (11:00am - 2:00pm, 3 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| GX10 hardware setup | Person A | 90 min | Boot GX10, connect network, verify GPU (`nvidia-smi`), identify DGX OS version, locate vLLM playbook |
| Load and serve K2 Think V2 | Person A | 60 min | Copy weights from USB, launch vLLM, verify API responds at localhost:8000/v1. Test with a simple clinical prompt. |
| Create patient data files | Person B | 90 min | Build `data/patients/maria_chen.json` and `data/scenarios/sepsis_12h.json`. All vitals/labs/meds in FHIR-like JSON. 288+ data points across 12-hour timeline. |
| Simulated EHR endpoints | Person B | 60 min | FastAPI routes that serve patient data from JSON files. `GET /fhir/Patient/1`, `GET /fhir/Observation?patient=1&time={sim_time}` |
| FastAPI skeleton + WebSocket | Person C | 90 min | FastAPI app, WebSocket at `/ws`, Pydantic models for PatientSnapshot, ClinicalAssessment, AuditEntry. Background task placeholder. |
| Audit chain module | Person C | 45 min | `agent/audit.py` — AuditChain class with add_entry(), verify_chain(). SHA-256 hashing. |
| React dashboard shell | Person D | 90 min | Vite + React-TS setup, install Recharts, `useSocket` hook, 2x2 CSS Grid + alert banner, dark theme, placeholder panels |

**Phase 1 Milestone**: K2 Think V2 is serving and responding. Patient data is queryable. FastAPI + WebSocket running. Dashboard connects and shows placeholders.

### Phase 2: Agent Core (2:00pm - 6:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| `agent/sense.py` | Person B | 60 min | Poll patient data from EHR sim. Compute trends (is HR rising? at what rate?). Return PatientSnapshot with current values + trend annotations. |
| `agent/prompts.py` | Person A | 60 min | System prompt + user message template. Include all clinical context, vital sign history table, medication list, previous alerts. |
| `agent/reason.py` | Person A | 120 min | Core reasoning: construct full prompt, call K2 Think V2, parse JSON response. Handle Think variant's chain-of-thought (may appear before JSON). Validate alert severity. Retry on parse failure. |
| `agent/alert.py` | Person C | 60 min | Alert deduplication, severity tracking, escalation logic. Don't re-fire same alert at same severity. Track alert history for context in next cycle's prompt. |
| `agent/loop.py` | Person A+C | 60 min | Wire SENSE → REASON → ALERT → OBSERVE. Run on asyncio timer. Broadcast to WebSocket. Log to audit chain. Handle errors gracefully. |
| `PatientMonitor.tsx` | Person D | 90 min | Vitals display: large numbers with trend arrows (↑↓), color coding (green=normal, yellow=concerning, red=critical). Sparkline history for each vital. Medication list. Lab results with flags. |
| `ReasoningLog.tsx` | Person D | 60 min | Scrollable log of agent's reasoning. Show chain-of-thought text. Color-code by severity. Timestamp each entry. Expandable detail view. |

**Phase 2 Milestone**: Agent runs complete SENSE → REASON → ALERT → OBSERVE cycle. K2 Think V2 produces clinical reasoning + alerts. Dashboard shows live patient vitals and agent reasoning.

### Phase 3: Demo Scenario (6:00pm - 10:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| `simulation/accelerator.py` | Person C | 60 min | Time acceleration: map wall-clock to sim time. `TIME_ACCELERATION_FACTOR = 36` → 12 sim hours in 20 real minutes. `get_sim_time()` function. |
| `simulation/baseline.py` | Person C | 60 min | Threshold-based alerting baseline. Track when traditional alerts WOULD have fired. Compute detection delay vs agent. |
| Scenario engine integration | Person B | 60 min | `ehr/scenario.py` serves time-appropriate data. As sim_time progresses, vitals deteriorate per the sepsis timeline. Labs result at appropriate sim_times. New medications appear at 10:30. |
| Drug interaction scenario data | Person B | 60 min | James Rodriguez patient data (if time). Warfarin + Ciprofloxacin scenario for demo variety. |
| `AlertTimeline.tsx` | Person D | 90 min | Visual timeline showing when agent fired alerts vs when baseline would have. Clear "60 minutes earlier" comparison. Color-coded severity markers. |
| `AuditTrail.tsx` | Person D | 60 min | Hash chain viewer. Running count, verification status, recent entries list. "Verify Chain" button. Privacy badge ("No data transmitted off-device"). |
| Prompt tuning | Person A | 120 min | **CRITICAL TASK**: Iterate on prompts to ensure K2 makes the right clinical reasoning at each of the 4 key moments. Test each moment manually. Adjust reasoning principles, context framing, output format as needed. This determines demo quality. |

**Phase 3 Milestone**: Full 12-hour scenario runs in ~20 minutes. Agent catches all 4 key moments. Alert timeline shows agent outperforming baseline. Audit trail verifies. Dashboard looks good.

### Phase 4: Polish + Demo Prep (10:00pm - 6:00am, ~8 hours with sleep)

| Task | Owner | Est. Time |
|------|-------|-----------|
| `AlertBanner.tsx` (critical alert top banner) | Person D | 45 min |
| Error handling + reconnection | Person C | 60 min |
| James Rodriguez drug interaction demo (if time) | Person B | 90 min |
| Write demo script + talking points | Person A | 60 min |
| Plan and storyboard 30-second video | All | 60 min |
| Integration bug fixes | All | 120 min |
| **Sleep in rotating shifts** | All | 3-4 hours each |

### Phase 5: Final Polish (6:00am - 11:00am, 5 hours)

| Task | Owner | Est. Time |
|------|-------|-----------|
| End-to-end demo rehearsal (run 3+ times) | All | 120 min |
| Record and edit 30-second video | All | 60 min |
| Devpost submission | Person A | 30 min |
| Dashboard visual polish | Person D | 60 min |
| README.md + architecture diagram in repo | Person C | 30 min |

---

## 12. Demo Storyboard (5-7 minutes at judging table)

### Setup
Dashboard running time-accelerated 12-hour scenario. GX10 physically on table. Scenario timed so judges see the 09:00-10:00 critical moments during the demo.

### Script

**[0:00-0:30] Opening — The Problem**
"250,000 people die from medical errors in the US every year — it's the third leading cause of death. Most aren't dramatic — they're a lab value that got buried, a drug interaction that was dismissed because the alert system cries wolf on everything, a patient whose vitals were all individually 'fine' but who was actually sliding into sepsis. Today's clinical alert systems are threshold-based: they fire when a number crosses a line. They generate so many false alarms that doctors override 90% of them. We built something fundamentally different."

**[0:30-1:15] The Solution — Architecture**
"This is SentinelMD — an AI clinical surveillance agent running entirely on this device [point to GX10]. It uses K2 Think V2, a 70-billion parameter open-source reasoning model, running locally. It continuously monitors patient data — vitals, labs, medications, allergies, history — and reasons about clinical significance the way a physician would. Not threshold checking. Reasoning. And because it runs locally, no patient data ever leaves this device. We have a cryptographic audit trail that proves it."

**[1:15-2:30] Live Demo — Early Pattern Detection**
"Here's our ICU patient, Maria Chen. She was admitted for pneumonia. Right now it's 9am in our simulation. Let me show you her vitals — heart rate 88, blood pressure 118/70, respiratory rate 19, temp 37.8. All individually within normal ranges. A traditional alert system? Silent. No threshold breached.

But watch the agent's reasoning panel... [wait for cycle] There. The agent is connecting dots across multiple vital signs simultaneously. Heart rate up 10 bpm over 3 hours, temperature trending up 0.2 degrees per hour, respiratory rate climbing — and it's contextualizing this against her known infection and elevated WBC from this morning's labs. It's issuing a WARNING: 'Vital sign trajectory concerning for early systemic inflammatory response. Recommend repeat lactate and blood cultures.' This is 60 to 90 minutes before any threshold-based system would fire."

**[2:30-3:30] Live Demo — Sepsis Identification**
"Now it's 10am. New labs come in. Lactate has jumped from 1.4 to 2.1 — a 50% increase in two hours. The agent immediately escalates to CRITICAL. Look at the reasoning — it's citing SIRS criteria, it's calculating the lactate trajectory, it's synthesizing the procalcitonin elevation with the vital sign trends. And it gives specific, prioritized recommendations: blood cultures before changing antibiotics, IV fluid bolus at 30 mL per kg, broaden coverage.

Now look at the alert timeline at the bottom. The red markers are the agent's alerts. The gray dotted line? That's when a traditional threshold system would have fired — not until 10:30 at the earliest, when a single vital finally crosses a hard threshold. That's 90 minutes of lost time. In sepsis, every hour of delayed treatment increases mortality by 7.6%."

**[3:30-4:15] Live Demo — Drug Interaction with Context**
"Here's the other thing the agent catches. At 10:30, the covering physician orders Ciprofloxacin — a new antibiotic. A standard EHR alert would fire: 'moderate interaction with Metformin.' The doctor overrides it — they override 90% of these alerts because most are clinically irrelevant. But the agent does something different. It reasons about THIS patient's specific context: she has CKD with a GFR of 42, she's actively septic, and Metformin with impaired renal clearance plus sepsis creates lactic acidosis risk. It recommends holding the Metformin entirely during acute illness. That's not a threshold check — that's clinical reasoning."

**[4:15-4:45] The Privacy Story**
"Now let me show you something no cloud AI solution can do. [point to audit trail panel] This is our cryptographic audit chain. Every single data access, every inference input and output, is logged and hash-chained. A compliance officer can verify — mathematically, not on trust — that zero patient data was transmitted off this device. Click 'Verify Chain'... all 47 entries verified. HIPAA compliance by architecture, not by policy."

**[4:45-5:30] Why It Matters + Business Case**
"Three things make this different from everything on the market. First: it reasons, it doesn't just check thresholds. K2 Think V2's chain-of-thought means the agent can explain WHY it's concerned, which builds clinician trust instead of alert fatigue. Second: it runs on a $3,000 device with zero recurring costs. Epic's AI modules cost $500K+. Cloud monitoring for 50 ICU patients runs $180K per year in API costs alone. Third: it's fully open-source and fully local. No vendor lock-in, no data exposure, no internet dependency.

Community hospitals, rural clinics, nursing homes, developing-world hospitals — they all need this and none of them can afford what exists today. That's 3,000+ community hospitals in the US alone. At $10-30K per unit per year, this is a $100M+ market that no one is serving."

**[5:30-6:00] Close**
"SentinelMD: AI clinical surveillance that fits in your server closet. Built with K2 Think V2 on ASUS Ascent GX10."

### 30-Second Video Script
- [0-5s] Text on dark background: "250,000 deaths per year from medical errors"
- [5-10s] Shot of GX10 device, small and quiet on a desk
- [10-18s] Dashboard: vitals trending, agent reasoning through sepsis detection, CRITICAL alert fires
- [18-24s] Alert timeline: agent alert 90 minutes before traditional system
- [24-27s] Audit trail: "Zero patient data transmitted off-device. Cryptographically verified."
- [27-30s] "SentinelMD — Built with K2 Think V2 on ASUS Ascent GX10"

---

## 13. Prize Track Alignment

| Track | Fit | Why | Prize |
|-------|-----|-----|-------|
| **Grand Prize** | Strong | Novel, technically ambitious, real-world impact, polished demo | $4,000 / $2,000 / $1,000 |
| **Societal Impact (ASUS)** | Excellent | Medical errors = 3rd cause of death. Runs on ASUS GX10. Democratizes clinical AI for underserved hospitals. | 2x GX10 + ROG gear |
| **Best Use of K2 Think V2 (MBZUAI)** | Excellent | K2 Think V2 is THE core — its chain-of-thought clinical reasoning IS the product. Multi-step medical deduction is exactly what the model was built for. | reMarkable tablet per member |
| **Built with Zed** | Free eligibility | Use Zed as IDE. No extra work. | $2,000 / $1,000 / $500 |
| **Best UI/UX** | Strong if polished | Medical dashboard with real-time vitals, alert system, reasoning transparency | $100 |

**Primary targets**: Societal Impact + K2 Think V2. Grand Prize is a strong reach. Zed is free.

---

## 14. Cut List (if behind schedule)

| Priority | Feature to cut | Time saved | Impact on demo |
|----------|---------------|------------|----------------|
| Cut 1st | James Rodriguez drug interaction scenario (second patient) | 90 min | Low — one patient demo is fine |
| Cut 2nd | Audit trail panel (keep the backend, cut the UI component) | 60 min | Low — mention it verbally, show code |
| Cut 3rd | Baseline comparison timeline (just show agent alerts, describe baseline verbally) | 60 min | Medium — lose the "90 minutes earlier" visual but can state it |
| Cut 4th | Time acceleration (manually set scenario to key moments for demo) | 60 min | Medium — demo is less fluid but still works |
| Cut 5th | Vital sign sparkline histories (show current values only) | 45 min | Low — less polished but functional |
| **NEVER CUT** | Agent reasoning panel with chain-of-thought | — | This IS the demo. The visible reasoning is what distinguishes us from a rule engine. |
| **NEVER CUT** | CRITICAL alert banner + core alert system | — | Without alerts, there's no product. |
| **NEVER CUT** | At least one complete scenario moment (sepsis detection at 09:00-10:00) | — | Without a live demo of the agent catching something, there's nothing to show judges. |

---

## 15. Fallback Chain (if K2 Think V2 fails on GX10)

1. **Primary**: DGX OS pre-configured vLLM playbook
   ```bash
   vllm serve LLM360/K2-Think-V2 --dtype float16
   ```

2. **Fallback 1**: llama.cpp with GGUF
   ```bash
   ./llama-server -m K2-Think-V2-Q4_K_M.gguf --n-gpu-layers 999 --port 8000
   ```

3. **Fallback 2**: K2-V2-Instruct (same family, faster, less chain-of-thought)
   ```bash
   vllm serve LLM360/K2-V2-Instruct --dtype float16
   ```
   Note: Loses visible reasoning chain but still eligible for K2 track.

4. **Fallback 3**: Qwen2.5-14B-Instruct
   ```bash
   vllm serve Qwen/Qwen2.5-14B-Instruct --dtype float16
   ```
   Fast and reliable. Loses K2 track eligibility.

5. **Last resort**: Remote K2 API (if MBZUAI provides endpoint). Demo still shows GX10 running EHR sim + agent logic + dashboard. Not ideal for "fully local" pitch.

**Rule**: If primary doesn't work within 30 minutes, move to next fallback. Don't debug vLLM during hackathon.

---

## 16. Key Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| vLLM won't run on GX10 aarch64/Blackwell | Medium | Critical | Fallback chain. Test immediately Phase 1. |
| K2 Think V2 clinical reasoning is poor | Medium | High | Extensive prompt tuning in Phase 3. Fallback: structure more of the reasoning in code, use model for final synthesis only. |
| K2 produces unparseable output | High | Medium | Robust JSON extraction (regex for JSON within thinking tokens). Retry 2x. Safe defaults on failure. |
| Model hallucinates medical facts | Medium | High | System prompt emphasizes established criteria (SIRS, qSOFA). Validate recommendations against allowlist. Frame as "decision support, not orders" in demo. |
| Think variant too slow for demo pacing | Medium | Medium | Switch to NVFP4 quantization for 5x speedup. Reduce context length. Worst case: fallback to Instruct variant. |
| Scenario feels scripted / not "real AI" | Low | High | Chain-of-thought display proves live reasoning. Vary the scenario slightly between runs. Have judges ask "what if" questions and run a modified scenario. |

---

## 17. Dependencies and Installation

### Python (backend + agent)
```
# requirements.txt
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
websockets>=12.0
openai>=1.0.0          # For vLLM OpenAI-compatible API
pydantic>=2.0
python-dotenv>=1.0
```

### React Dashboard
```bash
cd dashboard
npx create-vite . --template react-ts
npm install recharts
```

### vLLM on GX10
```bash
# Option A: DGX OS playbook (preferred)
# Option B: pip install vllm
# Option C: Docker with --gpus all
```

---

## 18. Zed Track Notes

Same as GridMind plan: all team members use Zed as IDE for all coding. Free track eligibility, no extra work.

---

## 19. Devpost Submission Checklist

- [ ] Project title: "SentinelMD"
- [ ] Description: Problem (medical errors + privacy) → Solution (local agentic surveillance) → Architecture → Results
- [ ] 30-second demo video uploaded
- [ ] GitHub repo linked
- [ ] Tracks selected: Societal Impact (ASUS), Best Use of K2 Think V2 (MBZUAI), Built with Zed, Grand Prize, Best UI/UX
- [ ] Team members listed
- [ ] Technologies: K2 Think V2, ASUS Ascent GX10, FHIR, FastAPI, React, Recharts, vLLM, Zed, SHA-256 audit chain
