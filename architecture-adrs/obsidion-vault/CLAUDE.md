# Project Context
Objective is to capture all context around OBP Milestone 4 along with the Bet representation options from Upstream systems (FBP) to enable analysis on the options, compare in terms of impacts to Risk downstream tools, alignment with To-Be Future Bet model, differences between them, challenges in terms of business needs and any other technical issues and create an ADR.

# Goals

The ADRs produced in this repository should:

Follow a consistent structure and writing style
Be understandable by both technical and non-technical stakeholders
Clearly document decision rationale
Capture assumptions, constraints, risks, and impacts
Support future architectural reviews and audits
Align with enterprise architecture principles and standards

# Repository Context

Before creating or updating ADRs, always review:

context/context.md
context/specs.md
context/daily.md
context/decisions.md
context/retro.md
context/architecture-principles.md
context/non-functional-requirements.md
templates/adr-template.md

These files provide the business context, architectural principles, quality attributes, and ADR structure that all decisions must align with.

# ADR Standards

When creating ADRs:

Use concise, professional language.
Focus on decision rationale rather than implementation details.
Include measurable benefits where possible.
Consider business, operational, security, scalability, and cost impacts.
Present multiple options before recommending a decision.
Explicitly document trade-offs.
Avoid vendor bias unless justified by requirements.

Recommended ADR sections:

Title
Status
Date
Context
Problem Statement
Decision Drivers
Options Considered
Option Analysis
Decision
Consequences
Risks & Mitigations
Implementation Considerations
References

# Preferred ADR Evaluation Criteria

When comparing options, assess:

Business Value
Strategic Alignment
Security
Reliability
Scalability
Performance
Maintainability
Operational Complexity
Cost
Vendor Lock-in
Time to Deliver
Risk

# Tools to use
# Context
obsidion-vault (local Markdown files)

# ADR Authoring
GitHub Copilot
Claude
VS Code

# Diagramming
Draw.io

# Documentation
Markdown

# ADR Generation Instructions

Priority order when generating ADRs:

architecture-principles.md
non-functional-requirements.md
context.md
specs.md
decisions.md
daily.md
retro.md
adr-template.md

Decision hierarchy:

Architecture Principles
↓
Non-Functional Requirements
↓
Business Context
↓
Project Specifications
↓
Existing Decisions
↓
Current Work (Daily Log)
↓
Lessons Learned (Retro)
↓
ADR Generation

Use Markdown.
Follow the standard ADR template.
Include at least min.two to three viable options.
Recommend one option with supporting rationale.
Document trade-offs and consequences.
Include assumptions and risks.
Maintain an objective architecture viewpoint.
Write for enterprise architecture audiences including architects, engineering leaders, product owners, and stakeholders.

The output should be decision-focused, evidence-based, and suitable for architecture governance review.

# File Map
```
obsidion-vault/
│
├── CLAUDE.md
│
├── context/
│   ├── context.md
│   ├── specs.md
│   ├── daily.md
│   ├── decisions.md
│   ├── retro.md
│   ├── architecture-principles.md
│   └── non-functional-requirements.md
│
├── templates/
│   └── adr-template.md
│
└── adrs/
    ├── proposed/
    ├── accepted/
    ├── superseded/
    └── rejected/
    
```

# Do Not
- ** Do not deviate from any of the above standard instructure when generating ADRs. **
