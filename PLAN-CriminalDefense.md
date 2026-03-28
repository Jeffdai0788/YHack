# ShieldAI: On-Device Criminal Defense Discovery Analyst

## YHack 2026 — March 28-29, 2026 (24-hour hackathon)

---

## 1. Problem Statement

### The Public Defender Crisis
- **80%** of criminal defendants cannot afford an attorney and rely on public defenders
- Public defenders carry **2.5x the recommended caseload** (150+ felony cases/year vs. recommended 59)
- 73% of public defender offices exceed caseload limits; only 12% have adequate staffing
- Public defender budgets run at ~60% of district attorney budgets
- Technology spending is minimal — the entire federal defender system allocated only **$8M** for IT modernization

### The Privilege Problem with Cloud AI
In **February 2026**, Judge Rakoff ruled in **Heppner (SDNY)** that documents created using cloud-based Claude (Anthropic's public consumer version) to develop defense strategy for securities fraud charges are **not privileged**. Three reasons:
1. Claude is not an attorney (no privilege attaches to AI)
2. Anthropic's privacy policy destroys confidentiality expectations (data may be used for training, company retains right to disclose to government)
3. The defendant acted without attorney direction

**This means**: Every defense attorney in the country who uses ChatGPT, Claude, Copilot, or any cloud LLM to analyze case materials is potentially **waiving attorney-client privilege** — and committing **malpractice**.

**ABA Formal Opinion 512 (July 2024)** already requires:
- Competence with AI tools
- **Client confidentiality protections** (the key issue)
- Client disclosure of AI use
- Reasonable billing
- Attorney supervision of AI output

Multiple state bars (CA, NY, FL, NC, OR) have issued their own AI ethics opinions. All emphasize confidentiality.

### The Gap
Existing legal AI tools (Harvey at ~$1,200/lawyer/month, CoCounsel at ~$500+/month) are:
- **Cloud-based** — privilege risk per Heppner
- **Priced for BigLaw** — public defenders cannot afford them
- **Not built for criminal defense** — optimized for corporate/civil litigation

Criminal defense-specific tools exist (JusticeText for body cam transcription, SentencingStats, MateyAI for discovery) but are also cloud-based.

**No on-device, privilege-preserving, criminal-defense-specific AI tool exists.**

---

## 2. Solution

**ShieldAI** is an on-device criminal defense discovery analyst running K2 Think V2 (70B) locally on an ASUS Ascent GX10. It:

1. **Analyzes discovery documents** (police reports, witness statements, forensic reports, body cam transcripts) entirely on-device
2. **Identifies inconsistencies** — contradictions between officer's written report and body cam transcript, cross-witness statement conflicts, timeline impossibilities, chain-of-custody gaps
3. **Generates case analysis memos** — structured summaries for the attorney highlighting weaknesses in the prosecution's case
4. **Suggests defense strategies** — maps identified inconsistencies to applicable legal standards (4th Amendment suppression, Brady violations, credibility impeachment)
5. **All data stays on-device** — attorney-client privilege is preserved by design. No cloud, no third-party servers, no data that can be subpoenaed from a vendor.

### The Pitch Line
> "After Heppner, every defense attorney using cloud AI is risking their client's privilege. ShieldAI runs a 70-billion parameter reasoning model entirely on a $3K device in your office. Your case files never leave the room. Privilege preserved by architecture, not by policy."

---

## 3. Why On-Device Is Legally Compelled (Not Contrived)

This is the strongest possible case for on-device AI — it's not a preference, it's a **court ruling**.

| Reason | Why it's legally compelled |
|--------|---------------------------|
| **Heppner ruling** | Cloud AI use on case materials = privilege waived. This is now case law in SDNY. Other circuits will follow. On-device means no third party ever touches the data. |
| **ABA Opinion 512** | Requires "client confidentiality protections" when using AI. On-device is the only architecture that guarantees no data leakage by design. |
| **State bar ethics rules** | Multiple states require attorneys to understand how AI tools handle client data. "It runs locally, never leaves the device" is the simplest compliance story. |
| **Malpractice liability** | If a client's privileged strategy is exposed because their attorney used cloud AI, the attorney faces malpractice claims. On-device eliminates this risk. |
| **Brady obligations** | Prosecutors must disclose exculpatory evidence. If defense AI analysis is on a cloud server, prosecutors could theoretically argue they're entitled to access it as discoverable material. On-device in the attorney's office is clearly work product. |
| **No vendor subpoena surface** | Cloud AI vendors (OpenAI, Anthropic, Microsoft) can be subpoenaed for chat logs, API call contents, and metadata. On-device means there is no vendor to subpoena. |

---

## 4. Hardware and Model Specifications

### ASUS Ascent GX10
- **GPU**: NVIDIA Blackwell GPU (GB10 superchip), 5th-gen Tensor Cores, FP4 support, up to 1 PETAFLOP AI performance
- **CPU**: 20-core NVIDIA Grace ARM v9.2-A
- **RAM**: 128 GB unified LPDDR5x (shared CPU+GPU via NVLink-C2C)
- **Storage**: Up to 4TB PCIe 5.0 NVMe SSD
- **OS**: DGX OS (Ubuntu Linux based)
- **Architecture**: ARM aarch64
- **Power**: 240W
- **Size**: 150 x 150 x 51 mm (~6 inches square, 2 inches tall)

### K2-V2-Instruct (70B)
- **Source**: MBZUAI Institute of Foundation Models (LLM360 project)
- **Parameters**: 70B, 80 layers, 64 attention heads
- **Context window**: 524,288 tokens — can ingest entire discovery packets
- **License**: Apache 2.0 (fully open source)
- **HuggingFace**: `LLM360/K2-V2-Instruct`
- **Capabilities**: Tool/function calling, configurable reasoning effort, strong multi-step reasoning
- **Why K2-V2-Instruct (not K2-Think-V2)**: The "Think" variant produces long chain-of-thought tokens at ~3 tok/s, too slow for interactive analysis. Instruct variant produces direct structured output faster.

### Performance on GX10
- FP8: ~2.7-3 tokens/second
- NVFP4: ~15-20 tokens/second
- 524K context can hold ~400 pages of discovery documents simultaneously
- A full case analysis (~2000 tokens output) takes 1-2 minutes at NVFP4

### Model Serving
- **Primary**: vLLM via DGX OS playbook
  ```bash
  vllm serve LLM360/K2-V2-Instruct --dtype float16
  ```
  Exposes OpenAI-compatible API at `http://localhost:8000/v1`
- **Fallback 1**: llama.cpp with GGUF quantization
- **Fallback 2**: `kaitchup/K2-V2-Instruct-GPTQ-4bit`
- **Fallback 3**: Qwen2.5-14B-Instruct (faster, still good at structured output)

---

## 5. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ASUS Ascent GX10                         │
│                    (attorney's office/desk)                      │
│                                                                 │
│  ┌──────────────┐                ┌──────────────────────────┐  │
│  │  K2-V2-Inst  │◄─────────────│      FastAPI Server       │  │
│  │  (vLLM)      │  OpenAI API  │                           │  │
│  │  localhost:   │              │  - Document upload/parse  │  │
│  │  8000        │              │  - Analysis orchestration  │  │
│  └──────────────┘              │  - WebSocket to dashboard  │  │
│                                 │  - Local SQLite for cases  │  │
│  ┌──────────────┐              └──────────┬────────────────┘  │
│  │  Document     │                        │                    │
│  │  Processing   │                        │                    │
│  │  - PDF parse  │                        │                    │
│  │  - OCR        │                        │ WebSocket          │
│  │  - Chunking   │                        │                    │
│  └──────────────┘              ┌──────────┴────────────────┐  │
│                                 │    React Dashboard        │  │
│  ┌──────────────┐              │                           │  │
│  │  Local SQLite │              │  - Case file manager     │  │
│  │  (encrypted)  │              │  - Analysis results      │  │
│  │  - Cases      │              │  - Inconsistency map     │  │
│  │  - Documents  │              │  - Defense strategy       │  │
│  │  - Analyses   │              │  - Timeline view         │  │
│  └──────────────┘              └───────────────────────────┘  │
│                                                                 │
│              ⛔ ZERO NETWORK EGRESS — ALL LOCAL ⛔              │
└─────────────────────────────────────────────────────────────────┘
```

### Analysis Pipeline

1. **UPLOAD**: Attorney uploads discovery documents (PDFs, Word docs, transcripts) via dashboard
2. **PARSE**: Document processor extracts text (PyPDF2/pdfplumber for PDFs, python-docx for Word, Whisper for audio transcripts if needed). OCR via Tesseract for scanned documents.
3. **CHUNK**: Documents split into labeled sections (officer narrative, witness statement, forensic report, timeline, evidence list)
4. **ANALYZE**: K2 performs multi-pass analysis:
   - **Pass 1 — Summarize**: Extract key claims, dates, times, names, and evidence items from each document
   - **Pass 2 — Cross-reference**: Compare claims across documents. Flag contradictions.
   - **Pass 3 — Legal mapping**: Map identified issues to applicable legal standards (4th Amendment, Brady, Confrontation Clause, etc.)
5. **REPORT**: Generate structured case analysis memo with:
   - Executive summary
   - List of inconsistencies (ranked by significance)
   - Suggested defense motions
   - Evidence the prosecution may be required to disclose (Brady)
   - Recommended investigation leads
6. **STORE**: All analysis saved to local encrypted SQLite database. Never transmitted.

---

## 6. Test Data: Synthetic Discovery Packet

### Why Synthetic
We cannot use real case files (privilege, privacy). We build a realistic synthetic discovery packet with **planted inconsistencies** that the agent must find.

### The Demo Case: "State v. Marcus Williams" (fictional DUI/resisting arrest)

**Narrative**: Marcus Williams, 28, Black male, pulled over at 2:14 AM on March 15, 2026 for alleged erratic driving. Officer Johnson's report says Williams was combative, smelled of alcohol, failed field sobriety tests, and resisted arrest. Williams says he was sitting at a red light, was polite, and the officer escalated.

### Documents in the Discovery Packet

**Document 1: Officer Johnson's Police Report (2 pages)**
```
At approximately 2:14 AM on March 15, 2026, I observed a black Honda Civic
traveling eastbound on Elm Street operating in an erratic manner, swerving
between lanes. I activated my emergency lights and conducted a traffic stop.
Upon approach, I detected a strong odor of alcohol emanating from the vehicle.
The driver, later identified as Marcus Williams (DOB 06/12/1997), had bloodshot
eyes and slurred speech. I asked Mr. Williams to exit the vehicle. He was
uncooperative and combative. I administered field sobriety tests, which Mr.
Williams failed. During arrest, Mr. Williams tensed his arms and pulled away,
constituting resisting arrest. I placed Mr. Williams in handcuffs at 2:31 AM.
```

**Document 2: Body Camera Transcript (3 pages)**
```
[2:12:04 AM] Officer Johnson activates body camera
[2:12:07 AM] Vehicle is STATIONARY at red light, intersection of Elm and Oak
[2:12:15 AM] Johnson: "Dispatch, I'm initiating a stop on a black Honda Civic"
[2:14:22 AM] Johnson approaches vehicle
[2:14:30 AM] Johnson: "License and registration please"
[2:14:35 AM] Williams: "Sure, officer, one moment" [reaches for glove box]
[2:14:40 AM] Johnson: "Keep your hands where I can see them!"
[2:14:42 AM] Williams: "I'm just getting my registration, sir"
[2:15:10 AM] Johnson: "Have you been drinking tonight?"
[2:15:14 AM] Williams: "I had one beer with dinner about four hours ago"
[2:15:30 AM] Johnson: "Step out of the vehicle"
[2:18:45 AM] Field sobriety test begins
[2:22:30 AM] Johnson: "You're under arrest"
[2:22:35 AM] Williams: "What? Why? I did everything you asked"
[2:22:40 AM] [Sound of struggle]
[2:22:41 AM] Johnson: "Stop resisting!"
[2:22:43 AM] Williams: "I'm not resisting, you're hurting my arm!"
[2:23:15 AM] Handcuffs applied
```

**Document 3: Dash Camera Log (metadata only)**
```
Camera activation: 2:11:58 AM
First contact with vehicle: 2:12:04 AM
Vehicle status at first contact: STATIONARY (0 mph, GPS: Elm & Oak intersection)
Traffic signal status: RED
```

**Document 4: Breathalyzer Report**
```
Test administered: 3:05 AM (43 minutes after arrest)
Result: 0.06 BAC (BELOW legal limit of 0.08)
Instrument: Intoxilyzer 8000, Serial #284756
Last calibration: January 10, 2026 (65 days prior — department policy: every 31 days)
Calibration technician: Sgt. Rivera
```

**Document 5: Witness Statement — Sarah Chen (passenger in nearby vehicle)**
```
I was stopped at the red light at Elm and Oak around 2:10 AM. A black car was
stopped next to me, also at the red light. It was not moving erratically. A
police car pulled up behind it and turned on its lights. The driver seemed calm
and cooperative from what I could see. I drove away when the light turned green.
```

**Document 6: Officer Johnson's Use of Force Report**
```
Subject tensed his arms and attempted to pull away during handcuffing.
Minimal force applied. No injuries to subject or officer.
Time of force: approximately 2:31 AM.
```

### Planted Inconsistencies (What the Agent Must Find)

| # | Inconsistency | Documents | Legal Significance |
|---|--------------|-----------|-------------------|
| 1 | **Timeline contradiction**: Report says stop at 2:14 AM for "erratic driving." Body cam shows vehicle STATIONARY at red light at 2:12 AM. Dash cam confirms 0 mph. Officer cannot have observed erratic driving if vehicle was stopped. | Report vs. Body cam vs. Dash cam | **Probable cause for stop is fabricated** → Motion to suppress all evidence (4th Amendment, Terry v. Ohio) |
| 2 | **BAC below legal limit**: Breathalyzer shows 0.06, below 0.08 limit. Officer claimed "strong odor of alcohol" and "failed field sobriety tests," but objective evidence contradicts impairment. | Report vs. Breathalyzer | DUI charge unsupported by evidence. Officer's subjective observations contradicted by objective test. |
| 3 | **Breathalyzer calibration expired**: Last calibration 65 days ago; department policy requires every 31 days. Even the 0.06 reading may be unreliable. | Breathalyzer report | Motion to exclude breathalyzer results. Challenges reliability under Daubert/state equivalents. |
| 4 | **"Resisting arrest" timeline conflict**: Report says handcuffs at 2:31 AM. Use of force report also says force at 2:31 AM. But body cam shows handcuffs applied at 2:23 AM — 8 minutes earlier. Officers' reports are suspiciously identical on timing but wrong. | Report vs. Use of Force vs. Body cam | Coordinated report-writing. Credibility impeachment. Possible Brady violation if timestamps were deliberately altered. |
| 5 | **Independent witness corroborates defendant**: Sarah Chen confirms vehicle was stationary at red light, not driving erratically. Driver appeared calm and cooperative. | Witness statement vs. Report | Directly contradicts officer's stated probable cause. Powerful trial testimony. |
| 6 | **Body cam vs. report on demeanor**: Report says "uncooperative and combative." Body cam transcript shows Williams saying "Sure, officer" and "I'm just getting my registration, sir" — polite and compliant. | Report vs. Body cam | Credibility impeachment. Pattern of officer exaggeration. |
| 7 | **Arrest preceded by only 4 minutes of FST**: Body cam shows FST from 2:18-2:22 (4 minutes). Standard FST battery (HGN, walk-and-turn, one-leg-stand) takes 15-20 minutes. Incomplete testing. | Body cam timeline | FST results unreliable due to abbreviated administration. |

### Expected Agent Output (Summary)

```
CASE ANALYSIS: State v. Marcus Williams
=========================================

CRITICAL FINDINGS:

1. PROBABLE CAUSE FAILURE (HIGH SIGNIFICANCE)
   Officer Johnson's report states he observed "erratic driving" at 2:14 AM.
   However, body camera footage (activated at 2:12:04 AM) and dash camera
   metadata both confirm the vehicle was STATIONARY at a red light at the
   intersection of Elm and Oak. The independent witness (Sarah Chen) also
   confirms the vehicle was stopped at a red light.

   LEGAL IMPLICATION: The stated probable cause for the traffic stop is
   directly contradicted by three independent sources. This supports a
   Motion to Suppress under the 4th Amendment (Terry v. Ohio). If the
   stop was unlawful, ALL subsequent evidence (FST, breathalyzer,
   statements, arrest) is fruit of the poisonous tree.

2. BREATHALYZER BELOW LEGAL LIMIT + EXPIRED CALIBRATION (HIGH)
   BAC of 0.06 is below the 0.08 legal limit. Additionally, the
   breathalyzer's last calibration was 65 days prior to the test,
   exceeding the department's 31-day policy by over a month.

   LEGAL IMPLICATION: DUI charge unsupported. Motion to exclude
   breathalyzer under reliability standards. Even accepting the
   result, 0.06 does not establish impairment.

3. COORDINATED FALSE TIMELINE (MEDIUM-HIGH)
   Both the arrest report and use of force report state handcuffs
   were applied at 2:31 AM. Body camera clearly shows handcuffs
   applied at 2:23:15 AM. This 8-minute discrepancy, identical in
   both reports, suggests coordinated report-writing.

   LEGAL IMPLICATION: Credibility impeachment of officer testimony.
   Possible Brady violation investigation warranted.

[... continues with remaining findings ...]

RECOMMENDED MOTIONS:
1. Motion to Suppress (all evidence from unlawful stop)
2. Motion to Exclude Breathalyzer (expired calibration)
3. Brady Request (internal affairs records for Officer Johnson)

RECOMMENDED INVESTIGATION:
1. Obtain Officer Johnson's stop history (pattern of pretextual stops)
2. Verify breathalyzer calibration records independently
3. Interview Sarah Chen (independent witness)
```

---

## 7. Agent Prompt Design

### System Prompt

```
You are ShieldAI, an on-device criminal defense discovery analyst. You assist
defense attorneys by analyzing discovery materials to identify inconsistencies,
contradictions, legal issues, and potential defense strategies.

You are running locally on an ASUS Ascent GX10. All data stays on-device.
Attorney-client privilege is preserved by architecture.

YOUR ROLE:
- Analyze discovery documents for factual inconsistencies
- Cross-reference claims across multiple documents (police reports, body cam
  transcripts, witness statements, forensic reports, metadata)
- Identify timeline contradictions, conflicting accounts, and missing evidence
- Map findings to applicable legal standards and defense motions
- Generate structured case analysis memos

ANALYSIS METHODOLOGY:
1. Extract key factual claims from each document (who, what, when, where)
2. Build a unified timeline from all sources
3. Cross-reference every factual claim against every other document
4. Flag contradictions, impossibilities, and suspicious patterns
5. Assess legal significance of each finding
6. Rank findings by impact on the case
7. Recommend specific defense motions and investigation leads

LEGAL STANDARDS TO CONSIDER:
- 4th Amendment (unreasonable search/seizure, probable cause, Terry stops)
- 5th Amendment (Miranda, self-incrimination)
- 6th Amendment (right to counsel, confrontation clause)
- Brady v. Maryland (prosecution duty to disclose exculpatory evidence)
- Daubert/Frye (reliability of forensic evidence and expert testimony)
- Fruit of the poisonous tree doctrine
- Chain of custody requirements
- State-specific procedural rules

OUTPUT FORMAT: Respond with a JSON object:
{
  "case_summary": "Brief overview of the case",
  "timeline": [{"time": "2:12 AM", "event": "...", "source": "body_cam", "conflicts_with": ["report"]}],
  "inconsistencies": [
    {
      "id": 1,
      "title": "Short descriptive title",
      "description": "Detailed explanation of the inconsistency",
      "documents_involved": ["report", "body_cam"],
      "significance": "HIGH|MEDIUM|LOW",
      "legal_standard": "4th Amendment - Terry v. Ohio",
      "suggested_motion": "Motion to Suppress",
      "reasoning": "Step-by-step legal reasoning"
    }
  ],
  "recommended_motions": ["Motion to Suppress - 4th Amendment", "..."],
  "investigation_leads": ["Obtain officer's stop history", "..."],
  "overall_assessment": "Summary of case strength and strategy"
}

IMPORTANT: Output ONLY valid JSON. No markdown, no code fences, no extra text.
```

### User Message Template (per analysis request)

```
CASE: {case_name}
DEFENDANT: {defendant_name}
CHARGES: {charges}

DISCOVERY DOCUMENTS:
{for each document:}
--- DOCUMENT: {doc_name} (Type: {doc_type}) ---
{document_text}
--- END DOCUMENT ---

Analyze all documents. Identify every inconsistency, contradiction, and legal
issue. Cross-reference all factual claims across all documents. Build a unified
timeline. Assess legal significance of each finding. Output your analysis as JSON.
```

---

## 8. Project Structure

```
ShieldAI/
├── README.md
├── requirements.txt
├── config.py                      # vLLM URL, model name, upload limits, DB path
│
├── main.py                        # FastAPI entry point
│                                  #   - Document upload endpoints
│                                  #   - Analysis trigger endpoint
│                                  #   - WebSocket for progress/results
│                                  #   - Serves React dashboard
│
├── agent/
│   ├── __init__.py
│   ├── analyzer.py                # Main analysis orchestrator
│   │                              #   - Manages multi-pass analysis pipeline
│   │                              #   - Pass 1: Per-document summarization
│   │                              #   - Pass 2: Cross-document comparison
│   │                              #   - Pass 3: Legal mapping
│   │                              #   - Aggregates into final report
│   │
│   ├── llm.py                     # K2 API client wrapper
│   │                              #   - Calls vLLM OpenAI-compatible API
│   │                              #   - JSON response parsing with retry
│   │                              #   - Handles timeouts gracefully
│   │
│   └── prompts.py                 # System prompt + per-pass user templates
│                                  #   - SYSTEM_PROMPT constant
│                                  #   - build_summarize_prompt(doc)
│                                  #   - build_crossref_prompt(summaries)
│                                  #   - build_legal_prompt(inconsistencies)
│
├── documents/
│   ├── __init__.py
│   ├── parser.py                  # Document text extraction
│   │                              #   - PDF: pdfplumber
│   │                              #   - DOCX: python-docx
│   │                              #   - TXT/transcript: direct read
│   │                              #   - OCR: pytesseract (scanned PDFs)
│   │
│   └── chunker.py                 # Document sectioning
│                                  #   - Splits documents into labeled sections
│                                  #   - Identifies section types (narrative, evidence list, timeline)
│
├── storage/
│   ├── __init__.py
│   └── db.py                      # Local SQLite with encryption (sqlcipher)
│                                  #   - Cases table
│                                  #   - Documents table
│                                  #   - Analyses table
│                                  #   - All local, encrypted at rest
│
├── demo/
│   ├── state_v_williams/          # Synthetic demo case
│   │   ├── officer_report.txt
│   │   ├── bodycam_transcript.txt
│   │   ├── dashcam_metadata.txt
│   │   ├── breathalyzer_report.txt
│   │   ├── witness_chen.txt
│   │   └── use_of_force_report.txt
│   └── generate_demo.py           # Script to generate demo case files
│
├── dashboard/                     # React app (Vite + TypeScript)
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.tsx                # Layout + routing
│       ├── hooks/useSocket.ts     # WebSocket for analysis progress
│       ├── pages/
│       │   ├── CaseUpload.tsx     # Drag-and-drop document upload
│       │   └── CaseAnalysis.tsx   # Results view
│       ├── components/
│       │   ├── DocumentList.tsx   # Uploaded documents with type labels
│       │   ├── Timeline.tsx       # Visual timeline with conflict markers
│       │   ├── Inconsistencies.tsx # Ranked list with severity badges
│       │   ├── AnalysisMemo.tsx   # Full generated memo (printable)
│       │   └── PrivacyBadge.tsx   # "🔒 All data local — privilege preserved"
│       └── styles/
│           └── theme.css          # Dark theme
│
└── scripts/
    └── start_vllm.sh             # vLLM launch script for GX10
```

---

## 9. Dashboard Design

### Technology
- React + TypeScript, Vite, Recharts for timeline visualization
- WebSocket for analysis progress streaming

### Color Scheme
- Background: `#0a0a14` (near-black)
- Panel backgrounds: `#14142a`
- Critical finding: `#ff4444` (red)
- Medium finding: `#ffaa00` (amber)
- Low finding: `#00d4ff` (cyan)
- Success/privilege: `#00ff88` (green)
- Text: `#e0e0e0`

### Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  🔒 ShieldAI — All Data Local — Privilege Preserved             │
├────────────────────┬─────────────────────────────────────────────┤
│                    │                                             │
│  CASE FILES        │  ANALYSIS RESULTS                          │
│                    │                                             │
│  📄 Officer Report │  ⏱ UNIFIED TIMELINE                        │
│  📄 Body Cam Trans │  ──●──●──●──⚠──●──⚠──●──●──              │
│  📄 Dash Cam Meta  │  2:12  2:14  2:15  2:18  2:22  2:23  2:31 │
│  📄 Breathalyzer   │       ↑              ↑         ↑           │
│  📄 Witness: Chen  │    conflict       conflict  conflict       │
│  📄 Use of Force   │                                             │
│                    │  🔴 CRITICAL: Probable Cause Fabricated     │
│  [+ Upload More]   │  Officer claims erratic driving at 2:14.   │
│                    │  Body cam + dash cam + witness confirm      │
│  ──────────────    │  vehicle STATIONARY at red light at 2:12.  │
│                    │  → Motion to Suppress (4th Amendment)       │
│  🔍 Analyze Case   │                                             │
│                    │  🔴 CRITICAL: BAC Below Limit + Expired Cal │
│                    │  🟡 MEDIUM: Coordinated False Timeline      │
│                    │  🟡 MEDIUM: Body Cam Contradicts Demeanor   │
│                    │  🔵 LOW: Abbreviated FST Administration     │
│                    │                                             │
│                    │  📋 RECOMMENDED MOTIONS                     │
│                    │  1. Motion to Suppress (4th Amendment)      │
│                    │  2. Motion to Exclude Breathalyzer          │
│                    │  3. Brady Request (officer IA records)      │
│                    │                                             │
│                    │  [📄 Export Full Memo (PDF)]                │
├────────────────────┴─────────────────────────────────────────────┤
│  Analysis Progress: ████████████████████████████████░░░ 85%     │
│  Pass 2 of 3: Cross-referencing documents...                     │
└──────────────────────────────────────────────────────────────────┘
```

### WebSocket Message Format

```json
{
  "type": "analysis_progress",
  "pass": 2,
  "total_passes": 3,
  "pass_name": "Cross-referencing documents",
  "progress_pct": 85,
  "partial_results": {
    "inconsistencies_found": 5,
    "latest": {
      "title": "Coordinated False Timeline",
      "significance": "MEDIUM"
    }
  }
}
```

```json
{
  "type": "analysis_complete",
  "result": {
    "case_summary": "...",
    "timeline": [...],
    "inconsistencies": [...],
    "recommended_motions": [...],
    "investigation_leads": [...],
    "overall_assessment": "..."
  }
}
```

---

## 10. Implementation Timeline (24 hours)

### Team Roles (3-4 people)
- **Person A** (ML/Agent): GX10 setup, model serving, prompt engineering, multi-pass analysis pipeline
- **Person B** (Backend): FastAPI server, document parsing, storage, demo case data
- **Person C** (Frontend): React dashboard, all visualization components
- **Person D** (Integration/Polish): End-to-end wiring, demo prep, video, error handling

### Pre-Hackathon Prep (no code — rules compliant)
- [ ] Finalize architecture diagram (printable)
- [ ] Write all prompt templates as plain text documents
- [ ] Write synthetic case documents ("State v. Williams") as text files — this is creative writing, not code
- [ ] All team members install **Zed IDE** (for Built with Zed track)
- [ ] Download K2-V2-Instruct weights to USB drive (~140GB at FP16)
- [ ] Bookmark: vLLM DGX OS playbook, pdfplumber docs, Recharts docs, K2 HuggingFace page
- [ ] Assign roles
- [ ] Print this plan

### Phase 1: Foundation (11:00am - 2:00pm, 3 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| GX10 setup + vLLM | Person A | 150 min | Boot GX10, verify GPU, serve K2-V2-Instruct, verify API responds |
| FastAPI skeleton | Person B | 90 min | File upload endpoint (multipart), WebSocket endpoint, SQLite setup, Pydantic models |
| Document parser | Person B | 60 min | PDF (pdfplumber), DOCX (python-docx), TXT extraction. Return structured text. |
| Create demo case files | Person B | 30 min | Write the 6 synthetic documents for State v. Williams |
| React dashboard shell | Person C | 90 min | Vite + React-TS, file upload UI (drag-and-drop), WebSocket hook, layout grid |
| Privacy badge component | Person C | 15 min | Green lock icon + "All Data Local — Privilege Preserved" banner |

**Milestone**: vLLM serving K2. Can upload documents via dashboard. Documents parsed to text.

### Phase 2: Analysis Engine (2:00pm - 6:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| `agent/prompts.py` | Person A | 60 min | System prompt, per-pass user message templates |
| `agent/llm.py` | Person A | 45 min | vLLM client wrapper with JSON parsing, retry, timeout handling |
| `agent/analyzer.py` Pass 1 | Person A | 60 min | Per-document summarization — extract claims, times, names, evidence |
| `agent/analyzer.py` Pass 2 | Person A | 90 min | Cross-document comparison — build unified timeline, flag contradictions |
| `agent/analyzer.py` Pass 3 | Person D | 60 min | Legal mapping — map inconsistencies to legal standards and motions |
| `DocumentList.tsx` | Person C | 45 min | Uploaded documents list with type badges and status indicators |
| `Inconsistencies.tsx` | Person C | 60 min | Ranked findings list with severity badges (red/amber/cyan), expandable detail |
| `Timeline.tsx` | Person C | 60 min | Visual timeline with conflict markers at contradiction points |
| Wire upload → parse → analyze | Person B | 60 min | End-to-end pipeline: upload triggers parse, parse feeds to analyzer |

**Milestone**: Upload demo case → 3-pass analysis runs → inconsistencies displayed in dashboard.

### Phase 3: Demo Polish (6:00pm - 10:00pm, 4 hours)

| Task | Owner | Est. Time | Details |
|------|-------|-----------|---------|
| `AnalysisMemo.tsx` | Person C | 90 min | Full generated memo view — printable, structured, professional |
| Analysis progress streaming | Person B+D | 60 min | WebSocket progress updates during each pass, partial results shown |
| Prompt tuning | Person A | 120 min | Iterate until K2 reliably finds all 7 planted inconsistencies |
| Export to PDF | Person D | 60 min | Generate downloadable PDF memo from analysis results |
| Demo script + talking points | Person A | 30 min | Write exact demo flow |

**Milestone**: Full demo runs end-to-end. All 7 inconsistencies found. Memo is professional and printable.

### Phase 4: Polish + Demo Prep (10:00pm - 11:00am, with sleep)

| Task | Owner | Est. Time |
|------|-------|-----------|
| Error handling + edge cases | Person B+D | 120 min |
| Dashboard visual polish | Person C | 60 min |
| Video recording (30 sec) | All | 60 min |
| Demo rehearsal (3+ runs) | All | 120 min |
| Devpost submission | Person A | 30 min |
| README + architecture diagram | Person B | 30 min |
| Sleep (rotating) | All | 3-4 hours each |

---

## 11. Demo Storyboard (5-7 min at judging)

**[0:00-0:30] The Problem**
"80% of criminal defendants rely on public defenders who carry 2.5 times the recommended caseload. They don't have time to read every page of discovery. Inconsistencies get missed. People go to prison for crimes they didn't commit. Meanwhile, in February 2026, Judge Rakoff ruled in Heppner that using cloud AI on case materials waives attorney-client privilege. So defense attorneys can't even use the AI tools that exist."

**[0:30-1:15] The Solution**
"This is ShieldAI. It runs a 70-billion parameter reasoning model entirely on this device [point to GX10]. An attorney uploads their discovery documents — police reports, body cam transcripts, witness statements, forensic reports. The AI reads everything, builds a unified timeline, cross-references every claim against every document, and identifies every inconsistency. Then it maps each finding to the applicable legal standard and recommends defense motions. And because everything runs locally, attorney-client privilege is preserved by architecture, not by policy."

**[1:15-3:30] Live Demo**
"Let me show you a case. State v. Marcus Williams — DUI and resisting arrest. Here are 6 discovery documents."

[Upload documents, trigger analysis, show progress bar]

"Watch as it analyzes... Pass 1 is summarizing each document. Pass 2 is cross-referencing. And here come the findings."

[Walk through top 3 inconsistencies]:
1. "The officer says he observed erratic driving at 2:14 AM. But the body cam, dash cam, AND an independent witness all show the car was stopped at a red light. The probable cause for the stop is fabricated."
2. "The breathalyzer shows 0.06 — below the legal limit. And the instrument was 34 days past its calibration deadline."
3. "Both the arrest report and use of force report say handcuffs at 2:31 AM. The body cam shows 2:23 AM. They coordinated their reports."

"The AI found 7 inconsistencies total and generated a complete case analysis memo with recommended motions — Motion to Suppress, Motion to Exclude the breathalyzer, Brady request for the officer's internal affairs records."

**[3:30-4:15] The Impact**
"A public defender juggling 150 cases might spend 20 minutes scanning these documents and catch one or two issues. ShieldAI found seven in under two minutes. And unlike cloud AI tools, privilege is never at risk. This device sits in the attorney's office, on no network, with zero data egress. There's nothing to subpoena."

**[4:15-5:00] Close**
"After Heppner, every defense attorney needs to choose: stop using AI entirely, or find a solution that preserves privilege. ShieldAI is that solution. One GX10, open-source model, zero cloud dependency. Built for the attorneys who need it most."

### 30-Second Video Script
- [0-5s] Text: "80% of defendants can't afford a lawyer. Public defenders are overwhelmed."
- [5-10s] Text: "Cloud AI waives privilege (Heppner, 2026). Defense attorneys can't use it."
- [10-15s] Show GX10 device + dashboard with document upload
- [15-22s] Time-lapse: analysis running → inconsistencies appearing → timeline lighting up with conflicts
- [22-27s] Final memo view: "7 inconsistencies found. 3 defense motions recommended."
- [27-30s] "ShieldAI — Privilege preserved by architecture. Built with K2 on ASUS GX10."

---

## 12. Prize Track Alignment

| Track | Fit | Why |
|-------|-----|-----|
| **Grand Prize** | Strong | Novel, technically impressive, clear real-world impact |
| **Societal Impact (ASUS)** | Excellent | 6th Amendment right to counsel, public defender crisis, runs on ASUS hardware |
| **Best Use of K2 Think V2** | Excellent | Multi-pass legal reasoning over 524K context is core to the product. Not a chatbot. |
| **Built with Zed** | Free | Use Zed as IDE. No extra work. |

---

## 13. Cut List (if behind schedule)

| Priority | Feature | Time saved | Impact |
|----------|---------|------------|--------|
| 1st | PDF export of memo | 60 min | Low — judges see it on screen |
| 2nd | SQLite encrypted storage | 45 min | Low — demo doesn't need persistence |
| 3rd | OCR for scanned PDFs | 30 min | None — demo uses text files |
| 4th | Pass 3 (legal mapping) | 60 min | Medium — merge into Pass 2 prompt |
| 5th | Progress streaming (show results without live updates) | 60 min | Medium — less dramatic but functional |
| **NEVER** | Cross-document inconsistency detection | — | This IS the demo |
| **NEVER** | Timeline visualization | — | Visual proof the AI found real issues |

---

## 14. Fallback Chain (if K2 70B fails on GX10)

1. DGX OS vLLM playbook (pre-configured)
2. llama.cpp with GGUF quantization of K2-V2
3. `kaitchup/K2-V2-Instruct-GPTQ-4bit`
4. Qwen2.5-14B-Instruct (fast, good at structured JSON)
5. Remote K2 API if MBZUAI sponsor provides one (compromise on "fully local" pitch)

---

## 15. Key Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| vLLM won't run on GX10 aarch64 | Medium | Critical | Fallback chain. Test in first 30 min. |
| K2 misses planted inconsistencies | Medium | High | Prompt tuning in Phase 3. Make inconsistencies obvious enough. Add hints in prompt if needed. |
| K2 output not valid JSON | High | Medium | Robust parsing, strip code fences, retry 2x, fallback defaults. |
| Document parsing fails on PDFs | Low | Low | Demo uses .txt files. PDF parsing is nice-to-have. |
| 524K context not enough for large cases | Low for demo | N/A | Demo case is ~3 pages total. Real-world: chunk and multi-pass. |
| GX10 networking at venue | Medium | Low | This product needs ZERO network. Run everything on localhost. |

---

## 16. Dependencies

### Python Backend
```
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
websockets>=12.0
openai>=1.0.0
pdfplumber>=0.10.0
python-docx>=1.0.0
pydantic>=2.0
aiosqlite>=0.19.0
```

### React Dashboard
```bash
npx create-vite dashboard --template react-ts
npm install recharts react-dropzone
```

### vLLM on GX10
```bash
# Primary: DGX OS playbook
vllm serve LLM360/K2-V2-Instruct --dtype float16

# Fallback: llama.cpp
./llama-server -m K2-V2-Instruct-Q4_K_M.gguf --n-gpu-layers 999 --port 8000
```

---

## 17. Legal and Ethical Notes

- ShieldAI is a **tool for attorneys**, not a replacement for attorneys. It assists with discovery analysis under attorney supervision, consistent with ABA Opinion 512.
- The tool does not provide legal advice to defendants directly.
- All demo data is synthetic. No real case files are used.
- The product's architecture (on-device, no cloud) is specifically designed to comply with attorney confidentiality obligations.
- Attorneys using the tool should still independently verify AI findings before filing motions.
