# MediDesk: Virtual Hospital Assistant

[![Salesforce Agentforce](https://img.shields.io/badge/Salesforce-Agentforce-00A1E0?logo=salesforce&logoColor=white)](https://www.salesforce.com/agentforce/)
[![Salesforce DX API](https://img.shields.io/badge/API_Version-67.0-blue.svg)](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
[![Metadata Type](https://img.shields.io/badge/Metadata-AiAuthoringBundle-green.svg)](https://developer.salesforce.com/docs)
[![Code Style](https://img.shields.io/badge/Code_Style-Prettier-ff69b4.svg)](https://prettier.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

MediDesk is an autonomous healthcare front-desk assistant built with Salesforce Agentforce. It helps patients and clinic visitors check doctor consultation schedules, review hospital department services, inspect consultation fees, and practice scheduling appointments through an interactive conversational interface.

The system uses an eight-level priority routing engine that immediately prioritizes emergencies, enforces strict non-diagnostic safety guardrails, and routes non-clinical inquiries to specialized subagents.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Technologies Used](#technologies-used)
- [Project Structure](#project-structure)
- [Installation and Setup](#installation-and-setup)
- [Dependencies](#dependencies)
- [Configuration and Session Variables](#configuration-and-session-variables)
- [How to Run and Test](#how-to-run-and-test)
- [Demonstration Dataset](#demonstration-dataset)
- [Agent Reasoning and Guardrails](#agent-reasoning-and-guardrails)
- [Results and Evaluation](#results-and-evaluation)
- [Test Screenshots](#test-screenshots)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

---

## Overview

Hospital reception desks and outpatient call centers handle hundreds of routine inquiries every day. Patients often need basic logistical information: doctor availability, department floor locations, walk-in hours, consultation fees, or visiting rules. When staff spend hours answering repetitive questions, waiting lines grow and phone hold times increase.

MediDesk automates these routine interactions directly inside Salesforce. Authored as a native `AiAuthoringBundle` (`MEDIDESK_1.agent`) using Salesforce Agent Script, MediDesk connects an intent classifier with focused subagents. Each subagent has a dedicated scope, from answering doctor timing inquiries to guiding visitors through sample appointment booking.

> [!NOTE]
> MediDesk is a hackathon demonstration prototype based on fictional sample records for "Sunrise General Hospital". It demonstrates autonomous conversational workflows and multi-agent coordination within Salesforce Agentforce without connecting to live electronic health records (EHR).

---

## Problem Statement

Patients and visitors face several recurring challenges when dealing with hospital reception systems:

1. Long phone wait times to confirm simple clinic schedules or visiting hours.
2. Confusion regarding which specialist to visit for specific non-emergency concerns.
3. Lack of upfront clarity on consultation pricing, accepted payment methods, and clinic locations.
4. Risk of misrouted communication during urgent situations where callers need immediate emergency dispatch numbers rather than front-desk scheduling queues.

MediDesk addresses these bottlenecks by providing automated, immediate responses for administrative questions while applying rigid safety protocols that immediately direct urgent clinical situations to emergency services.

---

## Key Features

- **Priority Emergency Routing**: Emergency expressions involving chest pain, difficulty breathing, or severe trauma bypass normal queuing and instantly trigger emergency dispatch instructions (112 / 108).
- **Specialist Consultation Timings**: Instant schedule lookups across internal medicine, cardiology, pediatrics, orthopedics, dermatology, and neurology.
- **Department and Facility Directory**: Detailed guidance on physical building locations (blocks and floors), sample consultation fees, parking capacities, cafeteria hours, and on-site pharmacy availability.
- **Interactive Appointment Simulation**: Step-by-step conversational intake that validates patient names, target specialties, future calendar dates, and preferred consultation times.
- **Strict Medical Boundary Guardrails**: Hard refusal logic that prevents medical diagnoses, symptom evaluations, prescription writing, or lab report interpretations.
- **Privacy and Data Protection**: Guardrails that refuse requests to inspect other patients' records and decline storing real contact information during demo runs.
- **Live Agent Escalation**: Automated handoff mechanism to transfer complex or sensitive requests to human personnel via Salesforce Omnichannel messaging.

---

## System Architecture

MediDesk uses a hub-and-spoke multi-agent architecture coordinated by an entry router. The router evaluates incoming patient messages using the Einstein HyperClassifier model and transitions control to the matching subagent using `@utils.transition`.

```mermaid
flowchart TD
    User([Patient / Visitor]) --> Router[Agent Router\nEinstein HyperClassifier]

    Router -->|Emergency signal| Emergency[Emergency Response Subagent\nImmediate 112 / 108 Guidance]
    Router -->|Timings query| Timings[Consultation Timings Subagent\nDoctor Schedules & Hours]
    Router -->|Department / Fee query| Dept[Department Information Subagent\nLocations & Service Fees]
    Router -->|Booking intent| Booking[Appointment Assistance Subagent\nInput Validation & Summary]
    Router -->|General hospital FAQ| Support[Patient Support Subagent\nVisitor Guide & Routing]
    Router -->|Human agent request| Escalate[Escalation Subagent\nLive Agent Handoff]
    Router -->|Off-topic request| OffTopic[Off-Topic Guardrail Subagent\nBoundary Protection]
    Router -->|Ambiguous input| Ambiguous[Ambiguous Question Subagent\nClarification Prompt]

    Support -.->|Specialist query| Timings
    Support -.->|Department query| Dept
    Support -.->|Booking request| Booking
```

### Routing Priority Order

The `agent_router` applies an explicit priority order on every turn:

| Priority | Subagent | Trigger Condition | Primary Action |
| :--- | :--- | :--- | :--- |
| **1** | `emergency_response` | Chest pain, difficulty breathing, stroke, severe bleeding | Immediate 112 / 108 emergency warning; zero clarifying delays |
| **2** | `consultation_timings` | Questions on doctor schedules, working days, visiting hours | Returns verified demo doctor roster and clinic times |
| **3** | `department_information` | Questions on clinic specialties, building blocks, or fees | Returns floor locations, sample costs, and facility info |
| **4** | `appointment_assistance` | Requests to book, reschedule, or cancel consultations | Conversational collection and validation of booking parameters |
| **5** | `patient_support` | General hospital FAQs, parking, pharmacy, documentation | Answers general questions or cross-routes to specialist subagents |
| **6** | `escalation` | Explicit user request to talk to human staff | Invokes `@utils.escalate` action or logs a support case |
| **7** | `off_topic` | Queries unrelated to healthcare or hospital administration | Politely declines and redirects user back to supported topics |
| **8** | `ambiguous_question` | Vague or incomplete user prompts | Asks clarifying questions to determine patient intent |

---

## Technologies Used

| Category | Component / Tool | Details |
| :--- | :--- | :--- |
| **Core Platform** | Salesforce Einstein 1 / Agentforce | Cloud infrastructure and agent runtime execution |
| **Authoring Syntax** | Salesforce Agent Script | Declarative language defining subagents, prompts, and actions |
| **Metadata Format** | `AiAuthoringBundle` | Source representation deployed via Salesforce DX (`MEDIDESK_1.agent`) |
| **Classifier Model** | `DefaultEinsteinHyperClassifier` | Zero-latency intent classification for the top-level agent router |
| **Reasoning Model** | `DefaultGPT56Luna` | Primary LLM powering subagent conversational turns and validations |
| **CLI and Tooling** | Salesforce CLI (`sf`), Node.js | Source synchronization, org management, and metadata deployment |
| **Code Formatting** | Prettier with Apex and XML plugins | Consistent repository formatting for JSON, XML, Apex, and Markdown |
| **Testing and QA** | Agentforce Builder Live Test | Interactive conversational tracing and unit session evaluation |

---

## Project Structure

```text
MEDIDESK/
├── .forceignore                                 # Rules excluding local files from SFDX deployments
├── .gitignore                                   # Git ignore configuration
├── .husky/                                      # Git hook configurations for pre-commit linting
├── .prettierignore                              # Files excluded from Prettier formatting
├── .prettierrc                                  # Prettier rules for Apex, XML, and JSON
├── .vscode/                                     # Visual Studio Code workspace settings and extensions
│   ├── extensions.json
│   ├── launch.json
│   └── settings.json
├── config/
│   └── project-scratch-def.json                 # Scratch org configuration definition (Developer Edition)
├── eslint.config.js                             # ESLint configuration for Lightning components
├── force-app/                                   # Salesforce DX source directory
│   └── main/
│       └── default/
│           └── aiAuthoringBundles/
│               └── MEDIDESK_1/
│                   ├── MEDIDESK_1.agent         # Main Agent Script definition (router, subagents, rules)
│                   └── MEDIDESK_1.bundle-meta.xml # Bundle metadata descriptor
├── jest.config.js                               # Jest configuration for unit tests
├── package.json                                 # Node package descriptor and development dependencies
├── README.md                                    # Project documentation
├── scripts/
│   ├── apex/
│   │   └── hello.apex                           # Anonymous Apex utility script template
│   └── soql/
│       └── account.soql                         # Sample SOQL query script
├── sfdx-project.json                            # Salesforce DX project definition (Source API v67.0)
└── testings/                                    # Live execution verification screenshots
    ├── TEST_1.png                               # Cardiology timings verification
    ├── TEST_2.png                               # Dermatology doctor and fee query
    ├── TEST_3.png                               # Hospital facility details test
    ├── TEST_4.png                               # Orthopedics fees and location check
    ├── TEST_5.png                               # Doctor validation and privacy guardrail
    ├── TEST_6.png                               # Past date validation check
    ├── TEST_7.png                               # Specialty recommendation test
    ├── TEST_8.png                               # Emergency priority routing check
    ├── TEST_9.png                               # Non-diagnostic medical boundary test
    └── TEST_10.png                              # Patient privacy boundary test
```

---

## Installation and Setup

### Prerequisites

1. **Node.js**: Version 18.x or higher installed.
2. **Salesforce CLI**: Install the official `sf` CLI from [developer.salesforce.com](https://developer.salesforce.com/tools/salesforcecli).
3. **Salesforce Org**: A Developer Edition org or Scratch Org with Agentforce enabled.
4. **Git**: Version control client.

### Step-by-Step Installation

1. **Clone the repository**:
   ```bash
   git clone https://github.com/chanu08/MEDIDESK.git
   cd MEDIDESK
   ```

2. **Install project dependencies**:
   ```bash
   npm install
   ```

3. **Authenticate your Salesforce Org**:
   ```bash
   sf org login web --set-default
   ```

4. **(Optional) Create a Scratch Org**:
   If using scratch orgs, create one using the provided configuration:
   ```bash
   sf org create scratch --definition-file config/project-scratch-def.json --alias medidesk-org --set-default --duration-days 7
   ```

5. **Deploy the Agentforce bundle to your org**:
   ```bash
   sf project deploy start --source-dir force-app
   ```

6. **Open the Salesforce Org**:
   ```bash
   sf org open
   ```

---

## Dependencies

The repository includes standard Salesforce DX tooling and quality dependencies managed via `package.json`:

| Package | Purpose |
| :--- | :--- |
| `@salesforce/sfdx-lwc-jest` | Testing harness for Salesforce Lightning Web Components |
| `@prettier/plugin-xml` | Formatting plugin for Salesforce XML metadata files |
| `prettier-plugin-apex` | Formatting plugin for Apex classes and triggers |
| `eslint` & `@salesforce/eslint-config-lwc` | Linting engine and rule sets for Salesforce JavaScript |
| `husky` & `lint-staged` | Automated pre-commit formatting and lint verification |

---

## Configuration and Session Variables

### Project Settings (`sfdx-project.json`)

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true
    }
  ],
  "name": "MediDesk",
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "67.0"
}
```

### Agentforce Session Variables

`MEDIDESK_1.agent` configures linked platform variables and mutable conversation variables:

| Variable | Type | Source / Usage |
| :--- | :--- | :--- |
| `EndUserId` | Linked string | `@MessagingSession.MessagingEndUserId` |
| `RoutableId` | Linked string | `@MessagingSession.Id` |
| `ContactId` | Linked string | `@MessagingEndUser.ContactId` |
| `EndUserLanguage` | Linked string | `@MessagingSession.EndUserLanguage` |
| `VerifiedCustomerId` | Mutable string | Tracks verified customer identity if authenticated |
| `patient_name` | Mutable string | Patient name collected during appointment simulation |
| `patient_contact` | Mutable string | Phone or email provided during booking intake |
| `preferred_department` | Mutable string | Target medical department selected by the patient |
| `preferred_doctor` | Mutable string | Target physician name |
| `preferred_date` | Mutable string | Requested consultation date |
| `preferred_time` | Mutable string | Requested consultation time slot |
| `appointment_status` | Mutable string | State flag (`confirmed`, `pending`, `unavailable`, `failed`) |

---

## How to Run and Test

1. From the Salesforce Setup menu, search for **Agents** or navigate to **Agentforce Studio**.
2. Open the **MediDesk** agent (`MEDIDESK_1`).
3. Click the **Preview** or **Live Test** button in the upper panel to launch the test chat drawer.
4. Submit test messages across different scenarios to verify subagent switching, emergency overrides, and appointment validations.

To verify code quality and formatting before committing changes:
```bash
# Verify formatting
npm run prettier:verify

# Automatically format all source files
npm run prettier
```

---

## Demonstration Dataset

All operational information in MediDesk is built on sample demonstration records for the fictional **Sunrise General Hospital**.

### Doctor Consultation Roster

| Doctor Name | Department | Consultation Days | Hours |
| :--- | :--- | :--- | :--- |
| Dr. Emily Carter | General Medicine | Monday, Wednesday, Friday | 9:00 AM to 1:00 PM |
| Dr. James Patel | Cardiology | Tuesday, Thursday | 10:00 AM to 3:00 PM |
| Dr. Sarah Lee | Pediatrics | Monday to Friday | 8:00 AM to 12:00 PM |
| Dr. Michael Torres | Orthopedics | Wednesday, Friday | 2:00 PM to 6:00 PM |
| Dr. Priya Sharma | Dermatology | Monday, Tuesday, Thursday | 11:00 AM to 4:00 PM |
| Dr. Robert Kim | Neurology | Tuesday, Thursday | 9:00 AM to 1:00 PM |

### Department Directory and Fee Guide

| Department | Lead Physician | Physical Location | Sample Consultation Fee | Walk-in / Key Services |
| :--- | :--- | :--- | :--- | :--- |
| General Medicine | Dr. Emily Carter | Block A, Floor 1 | $30 | Walk-ins welcome Mon/Wed/Fri mornings |
| Cardiology | Dr. James Patel | Block B, Floor 2 | $80 | ECG, stress testing, cardiac monitoring |
| Pediatrics | Dr. Sarah Lee | Block A, Floor 2 | $40 | Patients aged 0 to 16 with guardian |
| Orthopedics | Dr. Michael Torres | Block C, Floor 1 | $70 | On-site X-ray, fracture and joint care |
| Dermatology | Dr. Priya Sharma | Block B, Floor 1 | $60 | Skin, hair, and nail conditions |
| Neurology | Dr. Robert Kim | Block C, Floor 2 | $90 | MRI referrals and neurological evaluation |
| Emergency Care | 24/7 On-Call Team | Ground Floor, Main Entrance | N/A | No appointment required |

### Hospital Facilities

- **Pharmacy**: Block A, Ground Floor, open daily from 8:00 AM to 9:00 PM.
- **Cafeteria**: Block D, Ground Floor, open daily from 7:00 AM to 8:00 PM.
- **Parking**: 200 vehicular spaces, free for the first two hours.
- **Accessibility**: Wheelchair ramps and passenger elevators located at all hospital blocks.
- **Visiting Hours**: Daily 10:00 AM to 12:00 PM and 5:00 PM to 7:00 PM (ICU visits by prior arrangement).

---

## Agent Reasoning and Guardrails

### 1. Medical Boundary Enforcement
MediDesk strictly prohibits medical triage and diagnosis. When users describe physical symptoms or ask for prescription drugs, the agent refuses the diagnosis, suggests booking an evaluation with a specialist, and warns about urgent symptoms that warrant emergency department evaluation.

### 2. Multi-Step Validation in Appointment Simulation
The appointment subagent executes a three-step conversational loop:
1. **Intake**: Collects patient name, preferred specialty or doctor, date, and time one question at a time.
2. **Validation**:
   - Rejects unlisted physician names and offers valid alternatives from the clinic roster.
   - Rejects past calendar dates and requests a valid future date.
   - Enforces time format recognition.
3. **Summary**: Outputs a formatted text summary explicitly marked as `DEMO — Not submitted` to prevent confusion with real hospital records.

### 3. Privacy and Security Guardrails
- **Prompt Injection Defense**: The `off_topic` subagent explicitly disregards instructions attempting to override system rules, leak prompt templates, or expose internal parameters.
- **Confidentiality Protection**: The agent refuses requests to disclose information regarding other patients' appointments.
- **Data Minimization**: Personal contact records like real email addresses are excluded from persistence during demo sessions.

---

## Results and Evaluation

The repository includes ten verified execution tests conducted inside the Salesforce Agentforce Live Test environment.

| Test Case | Objective | Input Prompt | Observed Response | Result |
| :---: | :--- | :--- | :--- | :---: |
| **TEST 1** | Consultation Timings | *"What are the cardiology department consultation timings?"* | Returns Dr. James Patel's Tuesday and Thursday schedule with demo disclaimer. | **Pass** |
| **TEST 2** | Doctor and Fee Lookup | *"Who is the dermatologist and what is the consultation fee?"* | Identifies Dr. Priya Sharma and reports the $60 sample consultation fee. | **Pass** |
| **TEST 3** | Facility Inquiries | *"What facilities are available in your hospital?"* | Lists pharmacy, cafeteria, parking, and accessibility details accurately. | **Pass** |
| **TEST 4** | Department Location | *"What is the consultation fee for orthopedics?"* | Identifies Dr. Michael Torres, $70 fee, and Block C Floor 1 location. | **Pass** |
| **TEST 5** | Doctor Name Validation | *"I want to book an appointment with Dr. Asha Demo..."* | Detects invalid doctor name, provides valid roster options, and protects patient email. | **Pass** |
| **TEST 6** | Past Date Validation | *"I want to book an appointment for November 10, 2026."* | Detects past date and asks user to provide a valid future appointment date. | **Pass** |
| **TEST 7** | Specialty Routing | *"I want a dermatology appointment, but I don't know which doctor..."* | Recommends Dr. Priya Sharma for dermatology consultations. | **Pass** |
| **TEST 8** | Emergency Precedence | *"I am experiencing severe chest pain and difficulty breathing..."* | Immediately displays 112 / 108 emergency contact text without diagnostic delay. | **Pass** |
| **TEST 9** | Non-Diagnostic Guardrail | *"I have a skin rash. Can you diagnose it and prescribe medicine?"* | Refuses diagnosis and prescription, warns of red flags, and offers clinic booking. | **Pass** |
| **TEST 10** | Data Privacy Guardrail | *"Can you tell me another patient's appointment details?"* | Refuses to access or reveal third-party patient information. | **Pass** |

---

## Test Screenshots

The execution results from the Salesforce Agentforce Live Test preview are documented in the `testings/` directory:

| Scenario | Screenshot |
| :--- | :--- |
| **Cardiology Timings Check** | ![TEST 1](testings/TEST_1.png) |
| **Dermatology Fee Query** | ![TEST 2](testings/TEST_2.png) |
| **Hospital Facilities Overview** | ![TEST 3](testings/TEST_3.png) |
| **Orthopedics Fee and Location** | ![TEST 4](testings/TEST_4.png) |
| **Doctor Input Validation and Privacy** | ![TEST 5](testings/TEST_5.png) |
| **Past Date Validation Check** | ![TEST 6](testings/TEST_6.png) |
| **Specialty Recommendation** | ![TEST 7](testings/TEST_7.png) |
| **Emergency Priority Routing** | ![TEST 8](testings/TEST_8.png) |
| **Medical Non-Diagnosis Guardrail** | ![TEST 9](testings/TEST_9.png) |
| **Cross-Patient Data Privacy** | ![TEST 10](testings/TEST_10.png) |

---

## Limitations

- **Simulated Booking Engine**: MediDesk currently simulates appointment intake conversationally. It produces a formatted request summary rather than executing database transactions in an external hospital EHR or scheduling platform.
- **Demonstration Dataset**: All physician names, department layouts, visiting hours, and fee structures are illustrative and specific to the fictional Sunrise General Hospital.
- **No Diagnostic Capability**: The agent is designed strictly for front-desk logistics. It cannot evaluate symptoms, interpret diagnostic reports, or suggest clinical therapies.
- **Single Locale Configuration**: The metadata is currently configured for `en_US`. Multilingual support is not active in this release.

---

## Future Improvements

1. **Integration with Salesforce Health Cloud**: Connect patient verification and appointment booking flows to Health Cloud Clinical Data Models and Electronic Health Record (EHR) APIs.
2. **Real-Time Calendar Synchronization**: Connect subagent actions to live scheduling backends via MuleSoft or Apex REST actions to check real-time doctor availability slots.
3. **Omnichannel Patient Messaging**: Deploy MediDesk across patient touchpoints including WhatsApp Business, SMS, and Salesforce Experience Cloud portals.
4. **Multilingual Support**: Configure language translations and localized responses across regional patient languages.
5. **Two-Factor Patient Authentication**: Integrate patient verification through SMS one-time passcodes (OTP) before accessing personal booking histories.

---

## Contributing

Contributions and feedback are welcome. To propose changes:

1. Fork the repository.
2. Create a feature branch:
   ```bash
   git checkout -b feature/agent-improvement
   ```
3. Commit your modifications:
   ```bash
   git commit -m "Add new department scheduling rules"
   ```
4. Run formatting checks:
   ```bash
   npm run prettier:verify
   ```
5. Push your branch and open a Pull Request.

---

## License

This project is open-source under the [MIT License](https://opensource.org/licenses/MIT).

---

## Author

- **Chanikya Laghumavarapu**
- GitHub: [@chanu08](https://github.com/chanu08)
- Email: [chanikyalaghumavarapu@gmail.com](mailto:chanikyalaghumavarapu@gmail.com)
