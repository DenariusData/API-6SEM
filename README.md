<br />
<span id="denarius-data"></span>

# <p align="center">Denarius Data</p>

<p align="center"><strong>AkaVision</strong> — Technical Document Classifier</p>

<p align="center">
    <a href="#challenge">Challenge</a>  |
    <a href="#solution">Solution</a>  |
    <a href="#product-decisions">Product Decisions</a>  |
    <a href="#product-backlog">Product Backlog</a>  |
    <a href="#dor">DoR</a>  |
    <a href="#dod">DoD</a>  |
    <a href="#sprint-schedule">Sprint Schedule</a>  |
    <a href="#technologies">Technologies</a> |
    <a href="#getting-started">Getting Started</a> |
    <a href="#api-documentation">API Documentation</a> |
    <a href="#database-modeling">Database Modeling</a> |
    <a href="#team">Team</a> |
    <a href="#project-guidelines">Project Guidelines</a>
</p>

> Project Status: **In Progress 🚧** <br /><br />
> Documentation Folder: [Link](https://github.com/DenariusData/API-6SEM/tree/main/docs) 📄 <br /><br />

---

<span id="challenge"></span>

# 🏅 Challenge

**AKAER** deals with the management of technical documents, an activity that demands significant time due to the volume and complexity of the concepts involved.

Today, a professional needs to read each document, manually associate it with technical standards and other related documents already stored in the company's databases, and assign labels to make future searches easier. Being a manual process, it takes hours of work and is prone to errors.

The core challenge of this initiative is to build a **technical document classifier** capable of ingesting, classifying, and indexing documents — supporting natural language search grounded in the actual archive, while enforcing access control by confidentiality level and compliance with LGPD.

---

<span id="solution"></span>

# 🏅 Solution

The proposed solution is a **document classification and retrieval platform** running entirely on **local infrastructure**, with no dependency on external APIs for document processing or answer generation.

The system will enable:

- Upload and classification of technical documents by confidentiality level (Public, Internal, Restricted, Confidential, Sensitive)
- Text extraction and OCR for scanned documents, preserving structure for citation by excerpt
- Semantic search over the archive using locally hosted embeddings
- Natural language question answering, with every claim traceable back to a document, revision, page and section
- Access control combining document classification and user profile, enforced at retrieval time
- Compliance with LGPD through data anonymization, access logging and de-characterization records

The goal is to reduce the manual effort of reading, classifying and cross-referencing technical documents, while keeping every answer auditable and grounded in the company's own archive.

→ [Back to top](#denarius-data)

---

## 📋 Non-Functional Requirements

| ID | Non-Functional Requirement | Description |
|---|---|---|
| RNF01 | API Documentation | The system must provide clear, comprehensive documentation for all API endpoints. |
| RNF02 | Data / Database Modeling | The system must implement a well-structured data model supporting the document taxonomy and classification rules. |
| RNF03 | LGPD Anonymization | The system must implement data anonymization mechanisms in compliance with the LGPD. |
| RNF04 | Access & Manipulation Logs | The system must record immutable logs of data access and manipulation. |
| RNF05 | De-characterization Records | The system must keep records of database de-characterization procedures. |

---

<span id="product-decisions"></span>

## 📐 Product Decisions

Closed decisions made by the Product Owner that orient every User Story. Changing one of these requires reviewing the affected stories.

| ID | Topic | Decision |
|---|---|---|
| D1 | Architecture | Document processing and answer generation run on a local model. No external AI API calls in the document pipeline. |
| D2 | AI | Every AI answer cites the source document and revision. |
| D3 | AI | The answer is split into "Extracted from document" and "AI interpretation" blocks, with a notice that the interpretation is not an official company determination. |
| D4 | Taxonomy | The taxonomy has three levels: Area, Category and Subcategory, plus free tags. |
| D5 | Data Model | A document can belong to multiple areas (N:N relationship), with exactly one marked as the **primary area** and the rest as secondary. A document is never duplicated across areas. |
| D6 | Metadata | Reference standard is an optional field. A document without an associated standard enters the archive normally. |
| D7 | Metadata | **Updated.** Required upload fields (16 total): title, identifier/code, document type, revision number, issue date, effective date (when applicable), responsible area, category and subcategory, status, classification level, origin/source, document owner, language, related standard (when applicable), validity date (when applicable), original file. |
| D8 | Security | **Updated.** Four classification levels exist: **Public, Internal, Restricted, Confidential** (the previous fifth level, Sensitive, was removed). |
| D9 | Security | **Updated.** Permission is the combination of **user/group + area + classification**. Belonging to an area does NOT automatically grant access to every document in that area. A document may also carry its own restriction outside the general rule, for exceptional cases. |
| D10 | Security | **Updated.** AI-processing authorization is a policy separate from classification: Public = allowed · Internal = allowed per policy · Restricted = requires explicit authorization · Confidential = blocked by default. The system records who authorized processing, when, and which document version. |
| D11 | Security | **Updated.** Without permission, minimal existence applies: the user sees only an "access not authorized" message — no title, excerpt, summary or content. For sensitive documents, not even the title is shown. |
| D12 | Process | Publishing flow: Upload → Initial classification → Validation → Available. Whoever uploads a document cannot approve their own document. |
| D13 | Process | **Updated.** Requests for a missing document are routed by the **area of the category**, with a central governance area handling exceptions where no owner is defined. |
| D14 | AI | The archive may contain English documents; questions may be asked in Portuguese or English. The answer is given in the language of the question, while the cited excerpt stays in its original language. Supported languages are kept in configuration, so new ones can be added without changing the system structure. |
| D15 | Performance | Term search under 2 seconds; AI query under 10 seconds. |
| D16 | Validation | Product validation scenario: material compatibility, in Engineering. |
| D17 | Data Model | **New.** Documents can be related to one another: Complements, Amends, References, Supersedes. Relations are registered once and shown on both linked documents' records. |
| D18 | Data Model | **New.** Document status: Current, Superseded, Cancelled, Under Review. Only the Current version appears in search; older versions remain visible in history with their corresponding status. |
| D19 | Security | **New.** Requests for access to a blocked document are always routed to the document's or owning area's responsible party — never to a fixed administrative area. |
| D20 | Taxonomy | **New.** Nine accepted document types: Standards/Regulations, Internal Procedures, Manuals, Instructions, Circulars, Bulletins, Reports, Manufacturer Documents, Complementary/Reference Documents. No closed taxonomy exists at the client; the team proposes the structure and validates it in writing before loading it into the system. |
 

---

<span id="product-backlog"></span>

## 🧵 Product Backlog

### 📋 Epics Legend

| Epic | Name |
|---|---|
| E1 | Ingestion and Document Processing |
| E2 | Classification and Access Control |
| E3 | AI-Powered Query |
| E4 | Infrastructure and Testing Base |
| E5 | Privacy and LGPD Compliance |

---

### ✅ Backlog Items Table

| ID | Epic | User Story | Priority | Points | Sprint | Status |
|---|---|---|---|---|---|---|
| US-01 | E4 | As the technical team, I want the LLM and embeddings model running on local infrastructure, so that we comply with the restriction of not using external APIs. | Very High | 8 | 1 | To Do |
| US-02 | E1 | As the technical team, I want the data structure implemented according to decisions D4, D7, so that it supports upload and search. | Very High | 5 | 1 | To Do |
| US-03 | E1 | As a collaborator, I want to upload a document with its metadata, so that it is added to the archive. | Very High | 5 | 1 | To Do |
| US-04 | E1 | As the system, I want to extract text from PDFs and Office files while preserving structure, so that citation by excerpt is possible. | Very High | 5 | 1 | To Do |
| US-05 | E4 | As the team, I want a representative test base loaded, so that we can develop and demo the product. | Very High | 3 | 1 | To Do |
| US-06 | E2 | As an administrator, I want to manage users and profiles, so that access control is supported. | Very High | 4 | 1 | To Do |
| US-19 | E5 | As the compliance lead, I want personal data stored in a separate, access-restricted database, so that the document archive never holds personal data directly. | Very High | 8 | 1 | To Do |
| US-20 | E5 | As the platform owner, I want a Terms of Use, Privacy Notice and Consent form published with version control, so that users accept them before their first access. | Very High | 13 | 1 | To Do |
| US-21 | E5 | As the system, I want to record an immutable log of every data access and action, so that we can audit usage without exposing personal data. | Very High | 5 | 1 | To Do |
| US-22 | E5 | As the compliance lead, I want a separate AI-processing authorization policy per document, so that sensitive content is never processed without explicit approval. | Very High | 5 | 1 | To Do |
| US-07 | E1 | As the system, I want to apply OCR to scanned documents, so that the historical archive becomes searchable. | High | 8 | 2 | To Do |
| US-08 | E1 | As the system, I want to split documents into chunks and index them, so that semantic search is possible. | High | 8 | 2 | To Do |
| US-09 | E2 | As a document manager, I want to classify documents as Public, Internal, Restricted or Confidential. | High | 3 | 2 | To Do |
| US-10 | E2 | As the document management lead, I want to validate the classification before a document becomes available. | High | 5 | 2 | To Do |
| US-11 | E2 | As an administrator, I want to grant access by combining user/group, area and document classification. | High | 8 | 2 | To Do |
| US-23 | E5 | As the compliance lead, I want the personal data database encrypted at rest, so that data stays protected even if storage is compromised. | High | 5 | 2 | To Do |
| US-24 | E5 | As a user, I want a "My Data" screen to view and edit my own name, phone and role, so that I control my personal information. | High | 5 | 2 | To Do |
| US-25 | E5 | As a user, I want to review, accept or cancel optional consent terms at any time, so that I stay in control of optional data uses. | High | 5 | 2 | To Do |
| US-26 | E5 | As a user, I want to permanently delete my account, so that my personal data is fully removed while my uploaded documents remain attributed to "Removed User." | High | 8 | 2 | To Do |
| US-12 | E1 | As a user, I want the system to identify the current revision, so that I don't consult an obsolete version. | Medium | 3 | 2 | To Do |
| US-13 | E3 | As a user, I want to ask a question in natural language and receive an objective, source-grounded answer. | Low | 13 | 3 | To Do |
| US-14 | E2 | As the system, I want to filter excerpts by user permission before building the model's context. | Low | 5 | 3 | To Do |
| US-15 | E3 | As a user, I want to ask about English documents in Portuguese and receive the answer in Portuguese. | Low | 5 | 3 | To Do |
| US-16 | E3 | As a user, I want the answer to make explicit the relationship between information from different documents. | Low | 5 | 3 | To Do |
| US-17 | E3 | As a user, I want to open the document at the exact cited excerpt, so that I can validate the information. | Low | 3 | 3 | To Do |
| US-18 | E1 | As a user, I want to request access to a blocked document or request the inclusion of a missing one. | Low | 5 | 3 | To Do |
| US-27 | E5 | As a user, I want to export all my personal data in a portable format, so that I can exercise my LGPD data-portability right. | Low | 5 | 3 | To Do |
| US-28 | E5 | As the security team, I want alerts on abnormal login attempts or mass downloads, so that we can react quickly to suspicious activity. | Low | 5 | 3 | To Do |
| US-29 | E5 | As an auditor, I want to query the access log by user, period and action with export, and be sure no one can alter or delete it, so that we have a trustworthy audit trail. | Low | 5 | 3 | To Do |
 

→ [Back to top](#denarius-data)

---

<span id="dor"></span>

# 🏃‍♂️ DoR — Definition of Ready

- User story written as "As [role], I want [action], so that [benefit]", with clear and testable acceptance criteria
- Applicable Product Decisions (business rules) identified and linked
- Wireframe attached, when the story involves a user-facing interface
- Known technical dependencies identified (other stories or decisions involved)
- Estimated in Story Points by the team
- No known blockers preventing the work from starting

---

<span id="dod"></span>

# 🏆 DoD — Definition of Done

- Code implemented and merged into `develop` via an approved Pull Request (minimum 1 reviewer)
- CI passing (build/lint), no open conflicts
- Acceptance criteria validated by the Product Owner
- Applicable Product Decisions (business rules) verified
- Documentation updated in Git, when applicable
- No known critical bugs open for the story

→ [Back to top](#denarius-data)

---

<span id="sprint-schedule"></span>

# 📅 Sprint Schedule

| Sprint | Period | Points | History |
|--------|--------|--------|---------|
| Sprint 1 | 09/07 – 09/27 | 30 | [Sprint 1 Docs](https://github.com/DenariusData/API-6SEM/tree/main/docs) |
| Sprint 2 | 10/05 – 10/25 | 35 | [Sprint 2 Docs](https://github.com/DenariusData/API-6SEM/tree/main/docs) |
| Sprint 3 | 11/02 – 11/22 | 36 | [Sprint 3 Docs](https://github.com/DenariusData/API-6SEM/tree/main/docs) |

→ [Back to top](#denarius-data)

---

<span id="technologies"></span>

# 💻 Technologies

<p align="center">
<img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
<img src="https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D" />
<img src="https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white" />
</p>

<p align="center">
Local Machine Learning models (embeddings & generation) · LGPD compliance
</p>

> Additional tools (web framework, containerization, etc.) will be added here as the team defines them.

→ [Back to top](#denarius-data)

---

<span id="getting-started"></span>

# 🚀 Getting Started

### Prerequisites

- [Git](https://git-scm.com/)

### 1. Clone the repository with submodules

```bash
git clone --recurse-submodules https://github.com/DenariusData/API-6SEM.git
cd API-6SEM
```

> **Already cloned without `--recurse-submodules`?** Run the command below to initialize the submodules:
>
> ```bash
> git submodule update --init --recursive
> ```

### 2. Configure the submodules to track their remote branches

By default, submodules are checked out in a detached HEAD state. To work on them as actual repositories (create branches, commit, push, etc.), run the following inside each submodule:

```bash
cd API-6SEM-BACKEND
git checkout develop
cd ..

cd API-6SEM-FRONTEND
git checkout develop
cd ..
```

### 3. Set up each service

Each submodule has its own setup instructions (dependencies, environment variables, run commands):

- [Backend setup](https://github.com/DenariusData/API-6SEM-BACKEND#getting-started)
- [Frontend setup](https://github.com/DenariusData/API-6SEM-FRONTEND#getting-started)

### Pulling submodule updates

To pull the latest changes from all submodules:

```bash
git submodule update --remote --merge
```

→ [Back to top](#denarius-data)

---

<span id="api-documentation"></span>

# 📓 API Documentation

🚧 Under construction

→ [Back to top](#denarius-data)

---

<span id="database-modeling"></span>

# 🖥️ Database Modeling

🚧 Under construction

→ [Back to top](#denarius-data)

---

<span id="team"></span>

# 👥 Team

<div align="center">

| Role | Name | LinkedIn & GitHub |
|------|------|-------------------|
| Scrum Master | Tiago Bernardo | [LinkedIn](https://www.linkedin.com/in/tiagobernardosantos/) · [GitHub](https://github.com/TiagoBernardoSantos) |
| Product Owner | Beatriz Sthefanny |[LinkedIn](https://www.linkedin.com/in/beatriz-santos-0b6773220/) · [GitHub](https://github.com/BeatrizSantos00) |
| Developer | Caio Osorio | [LinkedIn](https://www.linkedin.com/in/caio-o-a67224200/) · [GitHub](https://github.com/User-Business) |
| Developer | Aline Ramos | [LinkedIn](https://www.linkedin.com/in/aline-ramos-3186b130/) · [GitHub](https://github.com/allineramos) |
| Developer | Victor Ryan | [LinkedIn](https://www.linkedin.com/in/victor-ryan-51738b261) · [GitHub](https://github.com/yzvictorr) |
| Developer | Tiago Alberto | [LinkedIn](https://www.linkedin.com/in/tiago-alberto-303909167/) · [GitHub](https://github.com/tiago17santos) |

</div>

→ [Back to top](#denarius-data)

---

<span id="project-guidelines"></span>

# 📜 Project Guidelines

<details>
<summary>Click to expand — Project Rules and Commit Standard</summary>

## 👥 Team Participation Rules

> _To confirm with the team — carried over from the previous semester's template as a starting point._

- A maximum of **1 absence per month** is allowed for weekly meetings
- All members must respect deadlines and adhere to the **commit standard**
- Difficulties must be communicated proactively to avoid last-minute issues before final presentations
- Every team member is expected to **present at least one sprint**

---

## 📌 Commit Standard

All commits must follow the **"Commit Pattern – by Renato Adorno"** to ensure consistency and clarity across the repository.

### Commit Format

    <type>: <description in English>

The description must:
- Be written in **English**
- Use a **direct, imperative tone**
- Be **clear and concise**

---

### 🧩 Commit Types

- **fix** – Fixes a bug
- **feat** – Introduces a new feature
- **docs** – Documentation-only changes
- **style** – Code formatting changes with no logic impact
- **refactor** – Code improvements that do not alter behavior
- **build** – Changes to the build system or dependencies
- **test** – Adding or updating tests
- **chore** – Routine maintenance tasks

---

### ✅ Examples

    feat: add document upload endpoint
    docs: update backend README
    fix: correct classification level validation
    refactor: improve OCR pipeline performance
    test: add unit tests for permission filter

---

### ⚠️ Rules

- Always write commits in **English**
- Strictly follow the defined **commit types**
- Avoid vague or uninformative messages such as:
  - `update`
  - `fix stuff`

Always prefer descriptive, actionable messages such as:

    fix: correct null pointer exception in service layer

</details>


→ [Back to top](#denarius-data)
