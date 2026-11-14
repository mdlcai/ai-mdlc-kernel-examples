# RESEARCH.md — QUANTUM SDK

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "QUANTUM PHYSICS"

## Domain Signals

```yaml
domain_signals: ["has_scheduled_work"]
```

## Product Vision
**Problem:** # Ψ ◈ QAI // QUANTUM SDK
## Problem Statement

Quantum computing development remains complex, fragmented, and heavily dependent on specialized frameworks, simulation backends, and hardware ecosystems.

Existing platforms such as IBM Qiskit provide powerful capabilities but require developers to manage circuit construction, execution, optimization, validation, and integration across multiple components.

Furthermore, quantum-classical workflows often lack unified, deterministic governance for software engineering, security, reproducibility, and independent validation.

### The Problem

**There is a need for an accessible, extensible, and independently validated quantum computing framework that unifies quantum simulation, hybrid optimization, and governed software engineering.**

### The Solution

Develop **QAI Quantum SDK**, an independent Python-based quantum computing framework governed by the MDLC kernel.

The platform will:

1. **Build:** Construct quantum circuits and algorithms through a unified Python API.
2. **Simulate:** Execute quantum operations on conventional computing infrastructure.
3. **Optimize:** Apply quantum-classical algorithms and tensor-network techniques.
4. **Validate:** Benchmark results against established mathematical models and Qiskit reference implementations.
5. **Govern:** Enforce mandatory MDLC architecture, security, testing, and reviewer gates.
6. **Evolve:** Iteratively improve candidate solutions through measurable, validated optimization.

### Core Mathematical Model

\[
\Psi_{n+1}=G\left(V\left(Q\left(L(\Psi_n,E_n)\right)\right)\right)
\]

Where:

- **Ψₙ:** Current computational state
- **L:** Generate candidate solutions
- **Q:** Evaluate and optimize candidates
- **V:** Independently validate correctness, safety, and constraints
- **G:** Select and commit approved improvements
- **Ψₙ₊₁:** Next validated state

### Success Criteria

Deliver a reproducible Windows-based quantum computing SDK that demonstrates correct quantum circuit execution, measurable algorithm performance, compatibility testing against Qiskit, and auditable MDLC governance.

### Mission

**Make quantum computing more accessible, extensible, reproducible, and verifiable through governed hybrid quantum-classical engineering.**

**AI proposes. MDLC governs. QAI evolves.**

**Who it affects:** ## Who It Affects

The challenges surrounding quantum computing accessibility, scalability, integration, and validation affect researchers, developers, enterprises, and organizations exploring quantum-classical computing.

### 1. Quantum Researchers and Scientists
Researchers need reproducible experiments, reliable simulation, independent validation, and consistent benchmarking to evaluate quantum algorithms and distinguish meaningful advances from statistical variation.

### 2. Software Developers and AI Engineers
Developers face steep learning curves, specialized mathematical requirements, fragmented tooling, and complex integration when incorporating quantum algorithms into conventional software and AI applications.

### 3. Enterprises and Technology Organizations
Organizations exploring quantum computing require security controls, auditability, governance, measurable business value, and integration with existing infrastructure before adopting experimental technologies.

### 4. Quantum Computing Providers
Hardware and platform providers must manage different gate architectures, noise characteristics, execution environments, and compatibility requirements while supporting increasingly complex hybrid workloads.

### 5. Cybersecurity and Compliance Teams
Security professionals need visibility into quantum software dependencies, execution environments, sensitive data handling, cryptographic implications, and the integrity of computational results.

### 6. Universities and Educational Institutions
Students and educators need accessible, affordable environments to understand quantum mechanics, develop circuits, execute simulations, and validate results without requiring access to physical quantum hardware.

### 7. Industries Solving Complex Optimization Problems
Financial services, logistics, manufacturing, energy, telecommunications, and data center operators face computationally demanding optimization problems. They need credible evidence that quantum or hybrid methods provide advantages over established classical techniques.

### Primary Target Users

**Initial focus:** Quantum software developers, AI engineers, and researchers who need reproducible, validated quantum-classical experimentation on conventional computing infrastructure.

**Secondary focus:** Enterprises requiring governed experimentation, auditable results, and integration with existing software engineering processes.

### The Impact

Without integrated engineering governance and reliable validation, quantum computing adoption can remain costly, technically challenging, and difficult to justify through measurable outcomes.

**QAI Quantum SDK aims to reduce these barriers by making quantum development more accessible, reproducible, secure, and accountable.**

**AI proposes. MDLC governs. QAI evolves.**

**Why existing solutions fall short:** ## Why Existing Solutions Fall Short

Existing quantum computing frameworks, including IBM Qiskit, Cirq, PennyLane, and other specialized platforms, provide sophisticated tools for quantum circuit development, simulation, and hybrid optimization. However, important engineering challenges remain.

### 1. Fragmented Development Lifecycle
Quantum circuit design, simulation, optimization, testing, security, and deployment frequently require separate tools and workflows. This increases integration complexity and makes consistent engineering governance difficult.

### 2. Limited End-to-End Governance
Existing frameworks primarily focus on quantum computation rather than enforcing a complete software development lifecycle. Architecture decisions, security reviews, reproducibility requirements, and independent validation are generally left to developers and their organizations.

### 3. Simulation Scalability Constraints
Classical quantum simulation faces exponential memory and computational complexity in the general case. Statevector simulators become increasingly expensive as qubit counts grow, while tensor-network approaches depend heavily on circuit structure and entanglement.

### 4. Difficult Performance Comparisons
Quantum and classical optimization methods can be challenging to compare fairly. Differences in computational budgets, random seeds, noise models, and evaluation methods can produce misleading performance claims without standardized benchmarks.

### 5. Hardware and Backend Complexity
Quantum hardware differs in native gate sets, connectivity, noise characteristics, execution limits, and provider interfaces. Portability requires additional compilation, calibration awareness, and backend-specific adaptation.

### 6. Reproducibility and Validation Gaps
Quantum algorithms are probabilistic, making statistical confidence, experiment provenance, deterministic seeding, and independent verification essential. Although existing frameworks support many of these capabilities, consistent enforcement across projects is not automatic.

### The Opportunity

**The primary gap is not the absence of quantum computing tools. It is the lack of a unified, governance-first engineering environment that consistently connects quantum development, hybrid optimization, independent validation, and auditable software delivery.**

QAI Quantum SDK addresses this opportunity by integrating quantum computation with MDLC governance, reproducible benchmarking, modular execution backends, and controlled evolutionary optimization.

Rather than claiming to outperform established frameworks immediately, QAI will use them as reference implementations while developing independently validated capabilities.

**AI proposes. MDLC governs. QAI evolves.**

**Solution:** I'm building QAI // QUANTUM SDK, an independent quantum computing software framework in Python, governed by your MDLC kernel.

Think of it as a lightweight alternative to IBM Qiskit, with its own quantum simulation engine.
What we're building
1. Quantum Circuit Builder: Create qubits, quantum gates, superposition, and entanglement.
2. Quantum Simulator: Execute quantum circuits on a conventional Windows PC without quantum hardware.
3. Quantum Algorithms: Run Grover's search and QAOA optimization.
4. Quantum Compiler: In the next phase, translate and optimize circuits before execution.
5. Tensor Network Engine: In the next phase, use optimized tensor contractions to simulate suitable larger circuits more efficiently.
6. QAI Evolution Engine: Integrate hybrid quantum-classical optimization.
7. MDLC Governance: Enforce architecture, testing, security, validation, and reviewer gates throughout development.
The architecture
MDLC // GOVERNANCE KERNEL
Architecture · Security · Validation · Reviewer
QAI // QUANTUM SDK
Circuit API · Gates · Algorithms
Quantum Compiler


Statevector Simulator


Tensor Network Engine


Evolution Engine



Compiler, tensor network, and evolution integrations are planned extensions.VALIDATED RESULTS
Measurements · Benchmarks · Qiskit Comparisons

Why this matters
We're building more than a quantum simulator.

We're creating a test platform where MDLC governs the engineering process, QAI explores optimization strategies, and independent validation determines which results are acceptable.

The V1.0 foundation is built. The next objective is to prove correctness, expand performance, and eventually support real quantum hardware through compatible backends.

The ultimate goal: A governed, hybrid quantum-classical computing platform that can design, simulate, optimize, validate, and improve computational solutions.


## Users & Outcomes
**Key Workflows:**
# Ψ ◈ QAI // QUANTUM SDK
## Key Workflows

QAI Quantum SDK provides a governed development environment for designing, simulating, optimizing, validating, and evolving quantum-classical computational solutions.

### 1. Quantum Circuit Development
**Define → Construct → Compile → Execute → Measure**

Developers create quantum circuits using a Python API, configure qubits and gates, execute simulations, and analyze measurement results.

**Output:** Executable quantum circuits, statevectors, measurement distributions, and execution records.

### 2. Quantum Simulation
**Circuit → Backend Selection → Simulation → Results → Verification**

Execute quantum circuits using classical statevector simulation, with planned tensor-network and hardware-compatible backends.

**Output:** Simulated quantum states, measurement probabilities, runtime metrics, and reproducible results.

### 3. Hybrid Quantum-Classical Optimization
**Define Objective → Generate Candidates → Optimize → Validate → Select**

Combine quantum algorithms such as QAOA with classical optimization to evaluate candidate solutions under defined computational constraints.

**Output:** Optimized parameters, candidate rankings, convergence histories, and comparisons against classical baselines.

### 4. Independent Validation and Benchmarking
**Execute → Compare → Analyze → Verify → Report**

Compare computational results against mathematical expectations, Qiskit reference implementations, and classical algorithms using consistent seeds, resource budgets, and statistical evaluation.

**Output:** Correctness reports, performance benchmarks, reproducibility evidence, and validation status.

### 5. MDLC-Governed Software Engineering
**Research → Architecture → Specification → Build → Test → Security Audit → Reviewer Gate → Release**

Every material software change passes through mandatory MDLC governance stages. Failed validation or missing evidence prevents release approval.

**Output:** Versioned specifications, architecture decisions, test evidence, security assessments, reviewer approvals, and release records.

### 6. QAI Evolution and Continuous Improvement
**Observe → Generate → Evaluate → Validate → Commit → Repeat**

The QAI Evolution Engine explores improvements to algorithms, simulation strategies, and execution configurations while retaining validated checkpoints and rejecting candidates that violate defined constraints.

**Output:** Approved improvements, performance comparisons, decision histories, and reproducible checkpoints.

### 7. Enterprise Integration and Execution
**Request → Authenticate → Authorize → Execute → Monitor → Audit**

Planned enterprise interfaces expose governed computational capabilities to AI agents, research systems, and enterprise applications through controlled APIs and execution policies.

**Output:** Authorized execution records, telemetry, audit logs, and integration results.

---

## Unified Execution Model

**INPUT → MDLC GOVERNANCE → QAI COMPUTATION → INDEPENDENT VALIDATION → APPROVED OUTPUT → EVOLUTION**

\[
\Psi_{n+1}=G\left(V\left(Q\left(L(\Psi_n,E_n)\right)\right)\right)
\]

- **L:** Generate computational candidates.
- **Q:** Evaluate and optimize.
- **V:** Validate correctness, security, and constraints.
- **G:** Commit only approved improvements.
- **Ψₙ₊₁:** Produce the next validated state.

## Success Criteria

Every production release must demonstrate:

- Correct and reproducible quantum execution.
- Measurable performance against defined baselines.
- Independent validation of computational results.
- Mandatory security and governance checks.
- Traceable decisions and versioned evidence.
- Controlled improvement without bypassing release gates.

**Implementation scope:** V1.0 establishes foundational circuit construction, simulation, example algorithms, testing, and governance documentation. Advanced compiler optimization, tensor-network execution, continuous evolution, and enterprise APIs remain planned capabilities requiring separate implementation and validation.

**AI proposes. MDLC governs. QAI evolves.**

**Success Metrics:**
Ψ ◈ QAI // QUANTUM SDK
Success Matrix V1.0
The success matrix defines measurable acceptance criteria for quantum computation, simulation accuracy, performance, security, reproducibility, and MDLC governance.

Primary objective: Deliver a reliable, independently validated quantum computing SDK that demonstrates correct execution, reproducible results, and controlled software evolution.
RELEASE READINESS
Pending approval
Reviewer gate open

Previously reported: 8 automated tests passed, 1 optional Qiskit test skipped. This does not establish full acceptance or Windows compatibility.

1. Technical success matrix
Success metric	Target / acceptance criteria	Priority
Circuit correctness	100% of required reference circuit tests pass	Critical
Quantum gate fidelity	Statevector error ≤ 1×10⁻¹⁰ against reference	Critical
Measurement accuracy	Statistical tests pass at defined confidence levels	Critical
Quantum algorithms	Grover and QAOA pass correctness tests	High
Qiskit compatibility	100% of supported differential test cases pass	Critical
Simulation capacity	Execute 20-qubit supported circuits within resource limits	High
Reproducibility	Identical seeded configurations reproduce results	Critical
Performance	Benchmark runtime and memory against Qiskit Aer	High
Test coverage	≥ 90% statement coverage for core modules	High
Security	Zero unresolved critical or high release-blocking findings	Critical

The fidelity threshold applies to supported noiseless statevector cases after accounting for global phase. Performance targets are benchmark objectives, not verified V1.0 results.
2. MDLC governance success matrix
Governance gate	Acceptance requirement	Release condition
Research	Document assumptions and reference methods	Mandatory
Architecture	Approved module boundaries and interfaces	Mandatory
Specification	Testable functional requirements	Mandatory
Build	Implementation matches approved specification	Mandatory
QA	All required automated tests pass	Mandatory
Security Audit	Findings assessed and blockers remediated	Mandatory
Evidence Ledger	Results, environment, and versions recorded	Mandatory
Reviewer Gate	Independent approval recorded	Mandatory
Release	All required gates satisfied	Mandatory

No mandatory gate can be bypassed by an AI agent, optimization algorithm, or automated release process.
3. Performance and optimization metrics
Target thresholds
Reference correctness

100%



All supported reference tests pass
Core test coverage

90%



Minimum statement coverage
Governance compliance

100%



All mandatory release gates complete
Seed reproducibility

100%



Identical seeded results in a fixed environment
These bars represent required targets, not measured achievement.
4. QAI Evolution Engine acceptance
For a future integrated Evolution Engine, success requires demonstrable improvements without compromising correctness.
\[
\Psi_{n+1}=G(V(Q(L(\Psi_n,E_n))))
\]

A candidate is accepted only when:
\[
V(S)=1
\quad\land\quad
J(S)\leq J(\Psi_n)-\varepsilon
\]

Where \(V(S)\) confirms mandatory constraints, \(J\) is the defined minimization objective, and \(\varepsilon\) is the minimum meaningful improvement threshold.
Metric	Acceptance target
Optimization quality	Meets or improves the classical baseline
Statistical validation	30 independent seeds for research comparisons
Regression protection	Zero correctness regressions
Checkpoint recovery	100% of required recovery tests pass
Candidate traceability	100% of accepted changes have evidence
Optimization claims	Supported by held-out evaluation and uncertainty estimates

5. Release decision matrix
Acceptance checklist

0 / 7



[ ] Quantum correctness verified
[ ] Qiskit differential tests passed
[ ] Performance and resource limits verified
[ ] Test coverage ≥ 90%
[ ] Security audit approved
[ ] MDLC evidence complete
[ ] Independent reviewer approval

Release blocked
7 acceptance criteria outstanding.

Manual planning checklist only. Checking an item does not execute tests or grant formal approval. Copy acceptance checklist
Definition of Success
QAI Quantum SDK V1.0 succeeds when it can reliably construct, execute, measure, and validate supported quantum circuits on Windows, reproduce its results, pass independent Qiskit comparisons, and satisfy every mandatory MDLC release gate.

Future releases should demonstrate measurable computational or engineering improvements against classical and established quantum-framework baselines, without claiming quantum advantage where it has not been demonstrated.

AI proposes. MDLC governs. QAI evolves.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
monitoring: "basic health checks"
container_strategy: "Docker Compose"

# Data & Storage
database_preference: "PostgreSQL"

# Security & Compliance
rate_limiting: true
audit_logging: true

# Frontend
frontend_framework: "Next.js"
state_management: "React Context"

# Backend
backend_framework: "FastAPI"
orm_preference: "SQLAlchemy"
typescript_backend: false

# Scope & Platform
scale: "large — 50k+ concurrent"
multi_tenant: true
target_platforms: ["web"]
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
# Ψ ◈ QAI // QUANTUM SDK
## Brand Voice & Identity

### Brand Positioning
**QAI Quantum SDK is an engineering-first quantum computing platform built for precision, reproducibility, and continuous improvement.**

It combines quantum computation, classical optimization, and independent validation under the governance of the MDLC kernel.

### Brand Personality
- **Intelligent:** Scientifically grounded, technically sophisticated, and forward-thinking.
- **Precise:** Every claim is measurable, testable, and supported by evidence.
- **Bold:** Challenges conventional engineering limitations without exaggerating capabilities.
- **Innovative:** Explores emerging computational methods and new approaches to optimization.
- **Trustworthy:** Prioritizes transparency, security, reproducibility, and independent verification.
- **Autonomous:** Enables intelligent experimentation while maintaining deterministic execution controls.

### Tone of Voice

**Confident. Technical. Visionary. Minimalist.**

Communicate with the authority of an engineering organization and the ambition of a frontier research laboratory.

Avoid marketing hype, exaggerated quantum advantage claims, unnecessary jargon, and unsupported performance promises.

### Core Messaging

**Primary tagline:**

AI proposes. MDLC governs. QAI evolves.

**Product statement:**

Quantum computation meets governed intelligence.

**Mission statement:**

Make quantum computing accessible, reproducible, and verifiable through intelligent quantum-classical engineering.

**Vision statement:**

A future where quantum and classical computing operate together within independently validated, continuously improving computational systems.

### Messaging Pillars

1. **ENGINEER:** Build quantum circuits with precision.
2. **SIMULATE:** Explore quantum computation without requiring quantum hardware.
3. **OPTIMIZE:** Combine quantum algorithms with classical intelligence.
4. **VALIDATE:** Trust evidence, not assumptions.
5. **GOVERN:** Enforce security, correctness, and accountability.
6. **EVOLVE:** Improve through measurable, validated iteration.

### Writing Guidelines

**Use:**
- Direct, concise, technically accurate statements.
- Strong engineering terminology.
- Active voice and short sentences.
- Mathematical notation where it adds clarity.
- Quantifiable evidence and explicit limitations.
- Uppercase headings and restrained futuristic typography.

**Avoid:**
- Unsubstantiated claims of quantum supremacy or exponential speedups.
- Promises of guaranteed optimization.
- Excessive buzzwords.
- Anthropomorphizing AI systems.
- Presenting experimental capabilities as production-ready.
- Em dashes and overly promotional language.

### Visual Identity

**Design direction:** Advanced scientific computing meets aerospace mission control.

**Color palette:**
- Obsidian Black `#080B12`
- Deep Space Navy `#101B30`
- Quantum Cyan `#00D9FF`
- Electric Violet `#805BFF`
- Signal White `#F4F7FB`
- Validation Green `#00C896`

**Typography:** Modern geometric sans-serif for interfaces, monospaced typography for code, formulas, and telemetry.

**Visual language:** Quantum circuits, Bloch spheres, tensor networks, mathematical operators, computational graphs, precise grid systems, and restrained scientific visualization.

### Example Brand Copy

**Hero headline:**

QUANTUM COMPUTATION. VALIDATED INTELLIGENCE.

**Hero subheadline:**

Design quantum circuits. Execute hybrid simulations. Optimize computational systems. Validate every result.

**Developer message:**

Build quantum software with a framework designed for reproducibility, independent verification, and governed evolution.

**Enterprise message:**

Explore quantum-classical computing with auditable execution, measurable performance, and engineering controls.

**Research message:**

Experiment freely. Benchmark rigorously. Advance through evidence.

### Brand Signature

**Ψ ◈ QAI // QUANTUM SDK**

`BUILD → SIMULATE → OPTIMIZE → VALIDATE → EVOLVE`

**AI proposes. MDLC governs. QAI evolves.**

### Guiding Principle

**Innovation without validation is experimentation. Innovation with validation becomes engineering.**

### Art Direction
# Ψ ◈ QAI // QUANTUM SDK
## Design Language V1.0

### Design Philosophy

**Scientific precision meets computational intelligence.**

QAI Quantum SDK adopts a futuristic, engineering-first design language inspired by aerospace mission control, quantum research laboratories, advanced computing infrastructure, and scientific instrumentation.

The interface must communicate intelligence, precision, trust, and technological advancement without sacrificing usability.

**Core principle: Complexity beneath the surface. Clarity at every interaction.**

### 1. Visual Identity

**Design Style:** Quantum Minimalism / Scientific Futurism

- Dark, high-contrast interfaces.
- Precise geometric layouts and structured grids.
- Restrained neon accents.
- Mathematical notation and scientific visualizations.
- Technical typography with clear hierarchy.
- Subtle illumination, depth, and motion.
- Functional dashboards instead of decorative interfaces.

The overall appearance should resemble a next-generation quantum computing research environment, not a conventional SaaS dashboard.

### 2. Color System

| Color | Hex | Purpose |
|---|---|---|
| Obsidian Black | `#080B12` | Primary background |
| Deep Space Navy | `#101B30` | Panels and navigation |
| Quantum Cyan | `#00D9FF` | Active circuits and interactions |
| Electric Violet | `#805BFF` | Quantum states and optimization |
| Signal White | `#F4F7FB` | Primary text |
| Slate Gray | `#8D9AAF` | Secondary information |
| Validation Green | `#00C896` | Verified results |
| Warning Amber | `#FFB547` | Warnings and pending gates |
| Critical Red | `#FF5364` | Failed validation and security findings |

Colors must convey operational meaning, not merely decoration. Critical states must also use labels and icons rather than color alone.

### 3. Typography

**Primary:** Inter or an equivalent modern sans-serif.

**Technical:** JetBrains Mono or an equivalent monospaced font.

**Display:** Space Grotesk or a similarly geometric typeface.

Typography hierarchy:

- Hero headings: 48–64px
- Section headings: 28–36px
- Panel headings: 18–24px
- Body text: 14–16px
- Telemetry and metadata: 12–14px

Use uppercase typography selectively for product identity, technical section labels, and system states.

### 4. Layout Architecture

The application follows a modular, responsive workspace architecture.

**Primary Navigation**
- OVERVIEW
- CIRCUIT BUILDER
- QUANTUM SIMULATOR
- ALGORITHMS
- OPTIMIZATION
- BENCHMARKS
- VALIDATION
- MDLC GOVERNANCE
- SETTINGS

**Workspace Structure**

Left: Navigation and project explorer.

Center: Quantum circuit canvas, simulation workspace, or active experiment.

Right: Circuit properties, execution configuration, state information, and validation controls.

Bottom: Execution console, telemetry, logs, and experiment results.

Panels should support resizing, collapsing, and contextual expansion.

### 5. Quantum Circuit Visualization

Quantum circuits are a primary visual element.

**Circuit design requirements:**

- Horizontal qubit timelines.
- Clearly defined quantum gate symbols.
- Color-coded gate categories.
- Connected control and target qubits.
- Measurement indicators.
- Drag-and-drop gate placement.
- Zoom and pan controls

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #5e68d2 | Buttons, links, active states |
| Secondary | #8a8fa8 | Accents, badges, highlights |
| Accent | #6364f1 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #ffffff | Cards, elevated containers |
| Text | #08090a | Headings, body text |
| Text Secondary | #6c6f7a | Captions, muted text |
| Success | #10b981 | Success states, confirmations |
| Warning | #f59e0b | Warnings, pending states |
| Error | #ef4444 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #7370ff | Buttons, links, active states |
| Secondary | #b4b6c1 | Accents, badges, highlights |
| Accent | #818af8 | Callouts, hover states |
| Background | #08090a | Page background |
| Surface | #131318 | Cards, elevated containers |
| Text | #f7f8f8 | Headings, body text |
| Text Secondary | #8a8f98 | Captions, muted text |
| Success | #34d399 | Success states, confirmations |
| Warning | #fbbf24 | Warnings, pending states |
| Error | #f87171 | Errors, destructive actions |

### Typography
- Heading: Manrope (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 14px, scale ratio: 1.2
- Scale: 9.7 / 11.7 / 14 / 16.8 / 20.2 / 24.2 / 29px

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Sharp
- Border radius: 2px (sm: 0px, lg: 6px, xl: 10px)
- Shadows: None — `none`
- Theme: Light only — single fixed palette, no theme switcher

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#5e68d2',
    secondary: '#8a8fa8',
    accent: '#6364f1',
    background: '#fafafa',
    surface: '#ffffff',
    foreground: '#08090a',
    muted: '#6c6f7a',
    success: '#10b981',
    warning: '#f59e0b',
    destructive: '#ef4444',
  },
  fontFamily: {
    heading: ['Manrope', 'system-ui', 'sans-serif'],
    body: ['Inter', 'system-ui', 'sans-serif'],
    mono: ['JetBrains Mono', 'monospace'],
  },
  borderRadius: {
    DEFAULT: '2px',
    sm: '0px',
    lg: '6px',
    xl: '10px',
  },
}
```

### CSS Custom Properties
```css
/* Light mode */
:root {
  --color-primary: #5e68d2;
  --color-secondary: #8a8fa8;
  --color-accent: #6364f1;
  --color-background: #fafafa;
  --color-surface: #ffffff;
  --color-text: #08090a;
  --color-text-secondary: #6c6f7a;
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --font-heading: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 2px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #7370ff;
  --color-secondary: #b4b6c1;
  --color-accent: #818af8;
  --color-background: #08090a;
  --color-surface: #131318;
  --color-text: #f7f8f8;
  --color-text-secondary: #8a8f98;
  --color-success: #34d399;
  --color-warning: #fbbf24;
  --color-error: #f87171;
}
```

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

An HTML design template was uploaded with this project (built by the customer
in Claude Design / artifacts). Fetch it at build time via the
`get_design_template` MCP tool — that is the source of truth, not this section.

### How to use it
- **Copy the template's `:root` design tokens verbatim FIRST.** The template
  declares its palette, typography, spacing scale, radius, and shadow as CSS
  custom properties on `:root` (`--color-*`, `--font-*`, `--space-*`,
  `--radius-*`, `--shadow-*`). Transcribe those values EXACTLY into the
  project's token layer (CSS variables / Tailwind theme / equivalent) — do not
  re-interpret, round, average, or "improve" them. This copied set is the
  canonical design-token system for the whole build.
- If the build needs a token the template did not declare (e.g. a missing
  state color or extra elevation), derive it to sit alongside the copied
  palette — never replace a value the template set.
- Use the markup as the visual scaffold for the landing/home surface — port
  its sections, hero, nav, and components into the project's framework rather
  than reinventing the layout.
- The copied tokens are binding for the ENTIRE build, OVERRIDING the values in
  `get_project_config` / the Design Language section below wherever they
  conflict. Express archetype-driven screens (anything not in the template)
  through these same tokens so the whole app stays visually coherent.
- Save the raw template to the project root as `DESIGN-TEMPLATE.html`, and
  record in `DECISIONS.md` the exact `:root` block you copied.

### If the template carries no `:root` block
- Some Claude Design exports style every element inline instead of declaring
  CSS custom properties. The template is still binding: derive the token set
  FROM the markup — background/surface/text/accent colours from the inline
  `style` attributes (ranked by frequency and role), the `font-family` stack
  from its `@font-face` + inline declarations, and radius/spacing/shadow from
  the values the template actually uses. Write that derived set as the
  project `:root` block, use it exactly as you would a copied one, and record
  it in `DECISIONS.md` with a note that it was derived rather than copied.
- `{{placeholder}}` tokens, `<sc-for>` loops and `<script type="text/x-dc"
  data-props="…">` tags are Claude Design component scaffolding. Treat each
  `data-props` default as the chosen value, treat `<sc-for>` bodies as the
  item template for a data-bound list, and never render a placeholder
  literally.
- If the file begins with a loader shell (`<noscript>This page requires
  JavaScript</noscript>` plus `<script type="__bundler/template">`), the real
  page is the JSON string inside that script and the assets are the base64
  entries in `__bundler/manifest`; decode those before reading tokens.

---

# Research Sections (AI fills these — do not write manually)

> Stage 0 research run: 2026-10-31. build_depth: comprehensive (all §3.1–§3.9 categories mandatory, threat model in §5). Every version below was read from the vendor's own registry or reference page on 2026-10-31 (PyPI JSON API, npm registry, Docker Hub, GitHub Releases API, vendor docs), never from memory. GitHub stars and last-push dates come from the GitHub REST API on the same day.

---

## 3. Source Categories

### 3.1 — Official Documentation & Vendor Sources

| Source | URL | What It Covers | Notes |
|--------|-----|----------------|-------|
| Qiskit SDK install guide | https://quantum.cloud.ibm.com/docs/en/guides/install-qiskit | Installing Qiskit 2.x on Windows/macOS/Linux | Docs moved from docs.quantum.ibm.com to quantum.cloud.ibm.com; use the new host in all links. PyPI: qiskit 2.6.1 (2026-10-22), requires Python >=3.10, win_amd64 wheels published. |
| Qiskit `Statevector` API | https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.quantum_info.Statevector | Reference statevector, `from_instruction`, `probabilities_dict`, `equiv(other, rtol, atol)` | `equiv` is "equivalent as a statevector up to global phase", which matches our fidelity rule (≤1e-10 after global phase). Basis index x means |x⟩ with qubit 0 as LSB. |
| Qiskit `StatevectorSampler` (V2 primitive) | https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.primitives.StatevectorSampler | `BaseSamplerV2` reference sampler; keyword-only `default_shots`, `seed`; PUB-based `run()` | Cannot handle mid-circuit measurement (built on `Statevector`). Results are `BitArray` per classical register; `get_counts()`. This is the differential oracle for our seeded sampling, distributionally (seeds will not match ours bit-for-bit). |
| Qiskit transpiler / compiler API | https://quantum.cloud.ibm.com/docs/en/api/qiskit/compiler and https://quantum.cloud.ibm.com/docs/en/api/qiskit/transpiler | `transpile()`, `generate_preset_pass_manager()` | Needed only for the reference side of differential tests and for the next-phase compiler design. |
| Qiskit 2.0 release notes | https://quantum.cloud.ibm.com/docs/en/api/qiskit/release-notes/2.0 | Breaking removals | BackendV1, Qiskit Pulse, V1 primitives, `.c_if()`, `qobj` removed; 32-bit Windows dropped; new C API. Any tutorial using `execute()` / `BasicAer` / `Sampler` V1 is obsolete. |
| Qiskit bit-ordering guide | https://quantum.cloud.ibm.com/docs/en/guides/bit-ordering | Little-endian convention; bitstring printing order | Bit 0 is rightmost in strings. QAI must pick and document one convention and map at the Qiskit boundary. |
| Qiskit Aer `AerSimulator` | https://qiskit.github.io/qiskit-aer/stubs/qiskit_aer.AerSimulator.html | Methods (`statevector`, `matrix_product_state`, `tensor_network` GPU-only, etc.), `max_parallel_threads`, `precision`, `max_memory_mb` | "A state vector of n-qubits uses 2^n complex values (16 Bytes)". 20 qubits = 16 MiB. PyPI qiskit-aer 0.18.2 (2026-10-26): Python >=3.11, win_amd64 wheels for cp311–cp315; requires `qiskit>=1.1,<3`. |
| NumPy release notes / `Generator` | https://numpy.org/doc/stable/release.html and https://numpy.org/doc/stable/reference/random/generator.html | NumPy 2.5.x; `np.random.default_rng(seed)` / PCG64 | numpy 2.5.4 (2026-10-10), Python >=3.12, win_amd64 wheels cp312–cp315. Use `Generator` (never global `np.random.seed`) for reproducible sampling. |
| SciPy `minimize(method='COBYLA')` | https://docs.scipy.org/doc/scipy/reference/optimize.minimize-cobyla.html | COBYLA options `rhobeg`, `tol`, `maxiter`, `catol`, `f_target` | Doc is SciPy 1.18; COBYLA "uses the pure-python implementation of the algorithm from PRIMA". The PRIMA switch changed iterates vs. old Fortran COBYLA, so pin SciPy exactly for reproducible QAOA traces. scipy 1.18.1 (2026-08-21). |
| FastAPI lifespan events | https://fastapi.tiangolo.com/advanced/events/ | `FastAPI(lifespan=asynccontextmanager)` | `on_event` is deprecated; lifespan and events cannot be mixed. fastapi 0.162.0 (2026-10-23), depends on starlette>=0.46, pydantic>=2.9. |
| FastAPI dependencies / async / Docker | https://fastapi.tiangolo.com/tutorial/dependencies/ , https://fastapi.tiangolo.com/async/ , https://fastapi.tiangolo.com/deployment/docker/ | DI for DB session, tenant, auth; `def` vs `async def` threadpool semantics; container guidance | CPU-bound simulation must never run inside an `async def` handler. |
| SQLAlchemy 2.1 asyncio | https://docs.sqlalchemy.org/en/21/orm/extensions/asyncio.html | `create_async_engine("postgresql+asyncpg://…")`, `async_sessionmaker(expire_on_commit=False)` | Page is for SQLAlchemy 2.1.6 (2026-10-21). "A single instance of AsyncSession is not safe for use in multiple, concurrent tasks." One session per request/task. |
| asyncpg | https://magicstack.github.io/asyncpg/current/ | Async PostgreSQL driver | asyncpg 0.32.0 (2026-10-06), win_amd64 wheels cp39–cp315. Prefer over psycopg async on Windows dev hosts (ProactorEventLoop incompatibility, see §3.3). |
| Alembic | https://alembic.sqlalchemy.org/en/latest/ | Schema migrations, async env.py template | alembic 1.20.1 (2026-10-22). |
| Pydantic settings | https://docs.pydantic.dev/latest/concepts/pydantic_settings/ | `BaseSettings`, env/.env/secrets-dir loading | pydantic 2.14.2 (2026-10-29), pydantic-settings 2.16.2 (2026-10-28, Python >=3.11). |
| Next.js `proxy.ts` file convention | https://nextjs.org/docs/app/api-reference/file-conventions/proxy | Request interception (formerly middleware) | v16.0.0: "Middleware is deprecated and renamed to Proxy. Proxy defaults to the Node.js runtime"; `runtime` config not allowed in proxy. Matcher must be static. Server Functions need their own authz checks. Codemod: `npx @next/codemod@canary middleware-to-proxy .` |
| Next.js CSP guide | https://nextjs.org/docs/app/guides/content-security-policy | Per-request nonce in `proxy.ts`, `x-nonce` header, `'strict-dynamic'` | Nonces force dynamic rendering for every page (no static/ISR/PPR); experimental SRI hash-based CSP is the alternative. `'unsafe-eval'` dev-only. |
| Next.js 16 release post | https://nextjs.org/blog/next-16 | Turbopack default, React 19.2, async `params`/`headers()`/`cookies()` | Node.js >=20.9 required. npm `next` latest = 16.4.3 (engines node >=20.9.0). |
| React 19 | https://react.dev/blog/2024/12/05/react-19 | Actions, `use`, Context as provider | npm `react`/`react-dom` 19.3.1. React Context is the mandated state layer. |
| PostgreSQL 18 `SELECT … FOR UPDATE SKIP LOCKED` | https://www.postgresql.org/docs/current/sql-select.html | Queue-claim semantics | Doc is PostgreSQL 18.6: SKIP LOCKED "can be used to avoid lock contention with multiple consumers accessing a queue-like table." Docker Hub `postgres:18.6`; 19rc1 also published, not for production. |
| PostgreSQL streaming replication / RLS | https://www.postgresql.org/docs/current/warm-standby.html and https://www.postgresql.org/docs/current/ddl-rowsecurity.html | Hot standby read replicas; row-level security | RLS gives defence-in-depth for multi-tenant isolation; streaming replica for read scale and benchmark-history queries. |
| Caddy `tls` directive | https://caddyserver.com/docs/caddyfile/directives/tls | `tls internal` | "use Caddy's internal, locally-trusted CA". Inside Docker the trust-store install fails, so export the root (`caddy trust` or copy `/data/caddy/pki/authorities/local/root.crt`). Leaf certs are 12 h. Caddy v2.11.7 (2026-10-03). |
| Caddy `reverse_proxy` directive | https://caddyserver.com/docs/caddyfile/directives/reverse_proxy | `lb_policy` (default `random`), `health_uri`, `health_interval`, passive `fail_duration` | Enough to load-balance N API/web replicas inside one Compose project. |
| Docker Compose file reference | https://docs.docker.com/reference/compose-file/ and https://docs.docker.com/compose/how-tos/startup-order/ | Compose Specification; `depends_on: condition: service_healthy` | Docker Compose v5.6.1 (2026-10-21). Healthchecks gate startup order. |
| argon2-cffi | https://argon2-cffi.readthedocs.io/en/stable/ | `PasswordHasher` (Argon2id), `check_needs_rehash` | argon2-cffi 25.1.0; argon2-cffi-bindings 26.1.0 has abi3 Windows wheels. |
| slowapi / limits | https://slowapi.readthedocs.io/en/latest/ | Decorator-based FastAPI rate limiting on top of `limits` | slowapi 0.1.10; its `redis` extra pins `redis<4.0.0`, which conflicts with redis-py 8.x. Use `limits` 5.8.0 directly (async storage) or a Redis Lua limiter. |
| Redis / Valkey | https://redis.io/docs/latest/ , https://redis.io/legal/licenses/ , https://valkey.io/ | Rate-limit counters, cache, queue broker | Redis 8+ is tri-licensed RSALv2 / SSPLv1 / AGPLv3. Valkey (BSD-3) is the drop-in alternative. Docker Hub: `redis:8.10.3-alpine`, `valkey/valkey:9.1.3-alpine`. redis-py 8.1.0. |
| Job queues: arq, RQ, Celery, Dramatiq, Procrastinate | https://arq-docs.helpmanual.io/ , https://python-rq.org/ , https://docs.celeryq.dev/en/stable/ , https://dramatiq.io/ , https://procrastinate.readthedocs.io/en/stable/ | Background workers / scheduling | Versions: arq 0.28.0 (last release 2026-04), rq 3.0.0 (2026-10-28), celery 5.6.3, dramatiq 2.2.1 (LGPL-3.0), procrastinate 3.10.0 (PostgreSQL-native, uses SKIP LOCKED + LISTEN/NOTIFY, built-in periodic tasks). Comparison in §3.9 / §4. |
| OpenQASM 3 live spec | https://openqasm.com/ | Circuit interchange language | Interop target for import/export; Qiskit offers `qiskit.qasm2` / `qiskit.qasm3`. |

**Key Takeaways:**
- The whole Python stack has Windows x86-64 wheels for CPython 3.12–3.15 (numpy, scipy, qiskit, qiskit-aer, asyncpg, argon2 bindings). NumPy 2.5 and SciPy 1.18 need Python >=3.12, so **Python 3.14 (3.14.10)** is the sweet spot: supported by every dependency, one year of patch releases, EOL 2030-10.
- Qiskit 2.x is the reference: use `Statevector`, `Statevector.equiv` and `StatevectorSampler` only. Pre-2.0 APIs (`execute`, `BasicAer`, V1 `Sampler`) are removed, so ignore older tutorials.
- Next.js 16 renamed middleware to `proxy.ts` (Node runtime). Nonce CSP is the strict option but forces dynamic rendering of every page. At 50k concurrent users that cost needs explicit horizontal scaling of the web tier, or the SRI alternative.
- slowapi's Redis extra is pinned to an ancient redis-py. Rate limiting should use `limits` directly or a hand-rolled atomic Lua script on Redis/Valkey.
- `tls internal` works for local HTTPS, but the root cert has to be exported from the Caddy container for browsers and test clients to trust it.

---

### 3.2 — GitHub & Open Source

| Repository | URL | Stars | Last Active | Relevance |
|------------|-----|-------|-------------|-----------|
| Qiskit/qiskit | https://github.com/Qiskit/qiskit | 7,940 | 2026-10-31 | Apache-2.0. Reference implementation for differential tests (test-only dependency, not runtime). |
| Qiskit/qiskit-aer | https://github.com/Qiskit/qiskit-aer | 708 | 2026-10-30 | Apache-2.0. Benchmark baseline (runtime/memory vs. our numpy engine). Repo contains its own QV benchmark suite to borrow methodology from. |
| quantumlib/Cirq | https://github.com/quantumlib/Cirq | 5,099 | 2026-10-30 | Apache-2.0. Competitor and API-design reference (Moments, `Simulator(seed=…)`). |
| PennyLaneAI/pennylane | https://github.com/PennyLaneAI/pennylane | 3,515 | 2026-10-31 | Apache-2.0. Hybrid-optimization reference (QNode, differentiable circuits). |
| jcmgray/quimb | https://github.com/jcmgray/quimb | 671 | 2026-10-21 | Apache-2.0 (GitHub reports NOASSERTION; PyPI says Apache-2.0). Prior art for the next-phase tensor-network engine (`CircuitMPS`, cotengra contraction paths). |
| quantumlib/qsim | https://github.com/quantumlib/qsim | 710 | 2026-10-21 | Apache-2.0. Gate-fusion and vectorized statevector techniques, reference only. |
| qulacs/qulacs | https://github.com/qulacs/qulacs | 529 | 2026-10-27 | MIT. Fast C++ statevector simulator, extra benchmark comparator. |
| ProjectQ-Framework/ProjectQ | https://github.com/ProjectQ-Framework/ProjectQ | 976 | 2026-10-19 | Apache-2.0. Compiler-engine pipeline design reference for the next-phase compiler. |
| NVIDIA/cuda-quantum | https://github.com/NVIDIA/cuda-quantum | 1,173 | 2026-10-31 | Apache-2.0. GPU simulation, out of scope for V1 (Windows/CPU target). |
| openqasm/openqasm | https://github.com/openqasm/openqasm | 1,526 | 2026-09-02 | Apache-2.0. Grammar and spec source for QASM import/export. |
| fastapi/fastapi | https://github.com/fastapi/fastapi | 104,390 | 2026-10-31 | MIT. Backend framework (hard constraint). |
| fastapi/full-stack-fastapi-template | https://github.com/fastapi/full-stack-fastapi-template | 46,130 | 2026-10-26 | MIT. Reference layout for FastAPI + Postgres + Docker Compose + Alembic, with a React (not Next.js) frontend. Reference only. |
| sqlalchemy/sqlalchemy | https://github.com/sqlalchemy/sqlalchemy | 12,275 | 2026-10-30 | MIT. ORM (hard constraint). |
| MagicStack/asyncpg | https://github.com/MagicStack/asyncpg | 8,118 | 2026-10-08 | Apache-2.0. Driver. |
| sqlalchemy/alembic | https://github.com/sqlalchemy/alembic | 4,461 | 2026-10-30 | MIT. Migrations. |
| pydantic/pydantic | https://github.com/pydantic/pydantic | 29,259 | 2026-10-31 | MIT. Schemas and settings. |
| laurentS/slowapi | https://github.com/laurentS/slowapi | 2,070 | 2026-10-29 | MIT. Rate limiting (see the redis<4 pin risk). |
| procrastinate-org/procrastinate | https://github.com/procrastinate-org/procrastinate | 1,438 | 2026-10-31 | MIT. PostgreSQL-native task queue with periodic tasks. |
| janbjorge/pgqueuer | https://github.com/janbjorge/pgqueuer | 1,548 | 2026-10-29 | MIT. Alternative PostgreSQL SKIP LOCKED queue (asyncpg). |
| python-arq/arq | https://github.com/python-arq/arq | 3,018 | 2026-04-16 | MIT. Asyncio Redis queue; slowest-moving (6.5 months since last push). |
| rq/rq | https://github.com/rq/rq | 10,701 | 2026-10-28 | BSD-2-Clause. Simple Redis queue, sync workers. |
| celery/celery | https://github.com/celery/celery | 29,101 | 2026-10-31 | BSD-3-Clause. Heavyweight, mature. |
| Bogdanp/dramatiq | https://github.com/Bogdanp/dramatiq | 5,336 | 2026-10-29 | LGPL-3.0. Fine as an unmodified dependency but needs a license review. |
| hynek/argon2-cffi | https://github.com/hynek/argon2-cffi | 739 | 2026-10-01 | MIT. Password hashing. |
| caddyserver/caddy | https://github.com/caddyserver/caddy | 78,601 | 2026-10-31 | Apache-2.0. TLS reverse proxy. |
| vercel/next.js | https://github.com/vercel/next.js | 143,315 | 2026-10-31 | MIT. Frontend (hard constraint). Example: https://github.com/vercel/next.js/tree/canary/examples/with-strict-csp |
| react/react | https://github.com/react/react | 251,906 | 2026-10-31 | MIT. UI library (repo now under the `react` org). |

**Evaluation Checklist:**
- [x] License compatible with our use case? Every runtime dependency is MIT / BSD / Apache-2.0. Exceptions: Dramatiq is LGPL-3.0 (avoid or review), and Redis 8 is RSALv2/SSPLv1/AGPLv3 (use as an unmodified network service, or pick Valkey BSD-3).
- [x] Actively maintained (commits in last 6 months)? All yes except arq (last push 2026-04-16, borderline).
- [x] Good documentation / examples? Yes for every hard-constraint component.
- [x] Community size & issue response time? Large for FastAPI, Next.js, SQLAlchemy, Qiskit; small but active for quimb, procrastinate, pgqueuer.
- [x] Use as dependency vs. reference only? Runtime dependencies: numpy, scipy, fastapi, sqlalchemy, asyncpg, alembic, pydantic(-settings), argon2-cffi, limits, redis-py, procrastinate (or arq). Test/bench-only: qiskit, qiskit-aer. Reference only: Cirq, PennyLane, quimb, qsim, qulacs, ProjectQ, CUDA-Q, full-stack-fastapi-template.

**Key Takeaways:**
- Qiskit and Aer belong in an optional extra (`qai-sdk[reference]`). The core SDK depends only on numpy (plus scipy for QAOA), which keeps the independence claim honest and the install light.
- Procrastinate (PostgreSQL-native, MIT, built-in periodic tasks, active) covers `has_scheduled_work` without adding a broker; Redis/Valkey is then only needed for rate limiting.

---

### 3.3 — Video & Tutorial Sources (YouTube, Courses, Talks)

| Title | URL | Creator | Length | Why It's Useful |
|-------|-----|---------|--------|-----------------|
| Solving MaxCut with QAOA: Quantum Tutorial for 100+ Qubits (Quantum Computing in Practice) | https://www.youtube.com/watch?v=P3s7TIMIvZ0 | Qiskit / IBM Quantum | ~1 h (series episode) | Canonical MaxCut cost Hamiltonian, ansatz and optimizer loop. Companion text: https://quantum.cloud.ibm.com/docs/en/tutorials/quantum-approximate-optimization-algorithm |
| Solving Optimization Problems with Quantum Algorithms (Qiskit Global Summer School 2024) | https://www.youtube.com/watch?v=RyRKmZ6A25k | Daniel Egger, IBM Quantum | Lecture (~1 h) | Fair-comparison framing of quantum vs. classical optimization, which feeds the "meets or improves classical baseline" acceptance metric. |
| QAOA on a real quantum computer, step-by-step Qiskit tutorial | https://www.youtube.com/watch?v=ZQbCUpAj1ck | Qubits & Coffee webinar | Webinar | End-to-end QAOA with current Qiskit primitives. |
| Lecture 20: Variational Quantum Algorithms, QAOA | https://www.youtube.com/watch?v=vG2zBowYVu4 | University lecture series | Lecture | Mathematical derivation of the QAOA expectation and parameter landscape. |
| Grover's Algorithm, Coding with Qiskit S2E3 | https://www.youtube.com/watch?v=0RPFWZj7Jm0 | Qiskit | ~15 min | Oracle plus diffuser construction and the optimal ⌊π/4·√(N/M)⌋ iteration count. Older Qiskit API, so take the concepts only. |
| Async SQLAlchemy: Async Database Engines, Sessions and FastAPI Dependencies | https://www.youtube.com/watch?v=T0JXFhJ7pM0 | BugBytes | ~30 min | Session-per-request dependency pattern with `async_sessionmaker`. |
| Python FastAPI Tutorial (Part 15): PostgreSQL and Alembic | https://www.youtube.com/watch?v=e8NnDz8uT7o | Corey Schafer | ~40 min | Production migration flow. A comment thread records the Windows `ProactorEventLoop` failure with psycopg async, fixed by moving to `postgresql+asyncpg`. |
| Quantum Wednesday: Efficient tensor-network simulations of weakly-measured circuits | https://www.youtube.com/watch?v=MzAPCULvJzs | Leonardo Banchi (seminar) | Seminar | Background for the next-phase tensor-network engine. |
| IBM Quantum Learning, Quantum Computing in Practice course | https://quantum.cloud.ibm.com/learning/en/courses/quantum-computing-in-practice | IBM Quantum | Course | Structured text course matching the video series. |

**Key Takeaways:**
- Most QAOA and Grover video material predates Qiskit 2.0, so treat it as conceptual and check every API call against §3.1.
- The Windows psycopg/ProactorEventLoop problem is real and recurring. asyncpg avoids it, which matters for the "reproducible on Windows" success criterion when the API runs outside Docker.

---

### 3.4 — Articles, Blogs & Written Tutorials

| Title | URL | Author/Site | Date | Relevance |
|-------|-----|-------------|------|-----------|
| Benchmarking Quantum Computer Simulation Software Packages: State Vector Simulators | https://arxiv.org/abs/2401.09076 (published https://www.scipost.org/SciPostPhysCore.7.4.075) | Jamadagni, Läuchli, Hempel | 2024 | Containerized, fair benchmarking method across 24 statevector packages (QFT validation, scaling curves). Template for our benchmark protocol vs. Aer. |
| Simulating Quantum Computations on Classical Machines: A Survey | https://arxiv.org/abs/2311.16505 | arXiv survey | 2023 | Statevector vs. tensor-network vs. stabilizer trade-offs. Supports the "tensor networks depend on circuit structure" claim. |
| Tensor networks for quantum computing | https://arxiv.org/abs/2503.08626 | arXiv review | 2025 | Design background for the next-phase TN engine. |
| Hyper-optimized tensor network contraction | https://quantum-journal.org/papers/q-2021-03-15-410/ | Gray & Kourtis, Quantum 5, 410 | 2021 | Contraction-path optimization (cotengra), the main TN scaling lever. |
| quimb: a python package for quantum information and many-body calculations | https://joss.theoj.org/papers/10.21105/joss.00819 | J. Gray, JOSS | 2018 | Citation for quimb. |
| Postgres job queues & failure by MVCC | https://brandur.org/postgres-queues | Brandur Leach | 2015 (still cited) | Warns that long-running transactions bloat a Postgres queue table. Claims must be short transactions with leases, not held locks. |
| What is SKIP LOCKED for in PostgreSQL 9.5? | https://www.2ndquadrant.com/en/blog/what-is-select-skip-locked-for-in-postgresql-9-5/ | 2ndQuadrant (Craig Ringer) | 2016 | Canonical explanation of the claim pattern used by the stuck-job watchdog. |
| Asyncio and concurrency best practices (Ray Serve) | https://docs.ray.io/en/master/serve/advanced-guides/asyncio-best-practices.html | Anyscale / Ray docs | current | NumPy releases the GIL so threads can help, but CPU-bound work should scale by replicas/processes and never block the event loop. |
| Multi-tenant data isolation with PostgreSQL Row Level Security | https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/ | AWS Database Blog | current | `SET app.tenant_id` plus RLS policy pattern for a pooled (shared-schema) tenancy model. |

**Key Takeaways:**
- The benchmark protocol should copy the published methodology: containerized or fixed environment, warm-up runs, repeated trials, median and IQR, memory high-water mark, recorded versions. Raw single-run timings are not evidence.
- Simulation jobs must run outside the request path (worker processes), and queue claims must be short transactions with a lease or heartbeat, or the queue table bloats.

---

### 3.5 — Standards, RFCs & Specifications

| Standard | Reference | URL | Applicability |
|----------|-----------|-----|---------------|
| OpenQASM 2.0 | Cross et al., arXiv:1707.03429 | https://arxiv.org/abs/1707.03429 | Circuit import/export (`qelib1.inc` gate names are the de-facto gate vocabulary). |
| OpenQASM 3 | Cross et al., arXiv:2104.14722; live spec | https://arxiv.org/abs/2104.14722 , https://openqasm.com/ | Forward-compatible interchange. V1 can export the QASM 3 subset (gates, measure). |
| Grover's search (original) | L. K. Grover, "A fast quantum mechanical algorithm for database search", arXiv:quant-ph/9605043 | https://arxiv.org/abs/quant-ph/9605043 | Correctness oracle: success probability sin²((2k+1)θ) with sin θ = √(M/N). |
| QAOA (original) | Farhi, Goldstone, Gutmann, arXiv:1411.4028 | https://arxiv.org/abs/1411.4028 | Ansatz definition for MaxCut. p=1 analytic expectation available for test oracles. |
| HTTP Semantics | RFC 9110 | https://www.rfc-editor.org/rfc/rfc9110 | Status codes, idempotency of methods (POST job submit needs an Idempotency-Key). |
| TLS 1.3 | RFC 8446 | https://www.rfc-editor.org/rfc/rfc8446 | "HTTPS only". Caddy defaults to TLS 1.2+/1.3 with HSTS-friendly config. |
| 429 Too Many Requests | RFC 6585 | https://www.rfc-editor.org/rfc/rfc6585 | Rate-limit response plus `Retry-After`. |
| RateLimit header fields | IETF draft-ietf-httpapi-ratelimit-headers | https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/ | Optional `RateLimit` / `RateLimit-Policy` headers. Still a draft, so treat as optional. |
| Problem Details for HTTP APIs | RFC 9457 | https://www.rfc-editor.org/rfc/rfc9457 | Uniform error envelope for the API. |
| OWASP ASVS | v5.x | https://owasp.org/www-project-application-security-verification-standard/ | Security verification baseline (OWASP Top 10 floor) for the Security Audit Gate. |
| OWASP Password Storage Cheat Sheet | — | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html | Argon2id parameters (min m=19 MiB, t=2, p=1). |
| NIST SP 800-63B (rev 4) | Digital Identity Guidelines | https://pages.nist.gov/800-63-4/sp800-63b.html | Password length/breach-list rules, session reauthentication. |
| Compose Specification | Docker | https://docs.docker.com/reference/compose-file/ | Deployment artifact format. |

**Key Takeaways:**
- Grover and QAOA both have closed-form or analytic test oracles (Grover success probability, p=1 QAOA MaxCut expectation on small graphs). Tests should assert against mathematics, not only against Qiskit, which gives two independent validators.
- No regulatory regime applies (no PII beyond account email, no payments). ASVS L2 plus the OWASP Top 10 is the right security bar.

---

### 3.6 — Competing / Adjacent Products

| Product | URL | Pricing | Strengths | Weaknesses | Our Differentiation |
|---------|-----|---------|-----------|------------|---------------------|
| IBM Qiskit + IBM Quantum Platform | https://www.ibm.com/quantum/pricing | SDK free (Apache-2.0). QPU: Open Plan 10 min / 28 days free; paid per-minute tiers | Largest ecosystem, real hardware, Aer simulators | Governance, reproducibility evidence and validation left to the user | Governed pipeline, evidence ledger, accept-only-if-validated evolution engine; Qiskit used as reference, not dependency |
| Google Cirq / qsim | https://quantumai.google/cirq | Free (Apache-2.0) | Clean API, fast qsim | No hosted governance or benchmarking product | Same as above, plus a web workspace |
| Xanadu PennyLane | https://docs.pennylane.ai/en/stable/ | Free (Apache-2.0) | Differentiable hybrid optimization | Optimization-centric, no audit trail | Hybrid optimization with statistical acceptance (30 seeds, ε threshold) |
| Amazon Braket | https://aws.amazon.com/braket/pricing/ | SV1 simulator $0.075/min (min 3 s/task); QPU per-task + per-shot | Managed simulators and multiple QPU vendors | Cloud cost, vendor lock-in, no governance layer | Free local and Windows execution, self-hostable Compose stack |
| Microsoft Azure Quantum | https://azure.microsoft.com/en-us/products/quantum | Provider-specific credits / per-job (https://learn.microsoft.com/en-us/azure/quantum/pricing) | Resource estimator, Q#, enterprise integration | Azure-centric | Vendor-neutral, open benchmark evidence |
| Classiq | https://www.classiq.io/ | Enterprise, no public price list (AWS Marketplace contract pricing) | High-level synthesis/compiler | Closed source, expensive | Open, transparent validation |
| qBraid | https://www.qbraid.com/pricing | Free / $19.99 / $99.49 per month; credits for QPU access | Hosted Jupyter with many SDKs pre-installed | An environment, not a validated SDK | Built-in validation and governance gates |
| NVIDIA CUDA-Q | https://developer.nvidia.com/cuda-q | Free (Apache-2.0) | GPU-scale simulation | Needs a CUDA GPU; Windows support via WSL | CPU/Windows-first reproducibility |

**Key Takeaways:**
- Nobody sells "governance-first quantum engineering". Competitors sell compute (IBM, AWS, Azure), SDK ergonomics (Cirq, PennyLane) or synthesis (Classiq). The differentiation is evidence: reproducible seeds, differential tests, and a release gate. The product copy must not claim a performance advantage over Aer, which is C++/OpenMP and will be faster than a NumPy engine.
- The web platform's value is the tenant-scoped experiment history, benchmark ledger and audit trail. The simulator itself is a commodity.

---

### 3.7 — Community & Forums

| Thread/Post | URL | Platform | Key Insight |
|-------------|-----|----------|-------------|
| qiskit-optimization #92, bit-ordering mismatch in application modules | https://github.com/qiskit-community/qiskit-optimization/issues/92 | GitHub Issues | Endianness mismatches caused real bugs even inside Qiskit's own ecosystem. QAI needs one documented convention and explicit boundary conversion tests. |
| qiskit-aer #2108, install failure (conda) | https://github.com/Qiskit/qiskit-aer/issues/2108 | GitHub Issues | Use pip wheels, not conda, for Aer. Current wheels cover Windows cp311–cp315. |
| Qiskit issue tracker | https://github.com/Qiskit/qiskit/issues | GitHub Issues | Watch for primitive/BitArray API changes between 2.x minors. Pin `qiskit==2.6.1` in the reference extra. |
| FastAPI GitHub Discussions | https://github.com/fastapi/fastapi/discussions | GitHub Discussions | Recurring threads on CPU-bound work blocking the loop and on lifespan resource management. |
| SQLAlchemy Discussions | https://github.com/sqlalchemy/sqlalchemy/discussions | GitHub Discussions | Async session lifetime and `MissingGreenlet` errors from lazy loads in async; use `selectinload` / `lazy="raise"`. |
| Caddy Community forum | https://caddy.community/ | Discourse | `tls internal` trust inside Docker and multi-upstream health checks are frequent topics. |
| PennyLane forum, Windows support for plugins | https://discuss.pennylane.ai/t/windows-support-for-pennylane-qskit-plug-in-install/569 | Discourse | Windows install friction for quantum plugins is a long-running pain point, which validates the Windows reproducibility goal. |

**Key Takeaways:**
- The top real-world failure classes are bit ordering, Windows installs, and async misuse (blocking loop, lazy loads). Each needs a dedicated test in SPEC.
- Stack Overflow and Reddit block automated fetching, so community evidence here comes from GitHub and Discourse. See §6 (non-blocking gap).

---

### 3.8 — APIs & Integrations

| API/Service | URL | Auth Method | Rate Limits | Docs Quality | SDK Available? |
|-------------|-----|-------------|-------------|--------------|----------------|
| QAI Platform REST API (ours, FastAPI) | (to be specified in SPEC.md; OpenAPI at `/api/v1/openapi.json`) | Session cookie (web, HttpOnly, SameSite=Lax, CSRF token) and tenant-scoped API keys (hashed at rest) for programmatic use | Per-tenant and per-key token bucket in Redis/Valkey; stricter on auth and job-submit routes | Auto-generated OpenAPI | Python SDK is the product itself |
| PostgreSQL 18 | https://www.postgresql.org/docs/current/ | Password (SCRAM-SHA-256) over the internal network | n/a | Excellent | asyncpg / SQLAlchemy |
| Redis 8 / Valkey 9 | https://redis.io/docs/latest/ , https://valkey.io/ | `requirepass` / ACL user, internal network only | n/a | Excellent | redis-py 8.1.0 (`redis.asyncio`) |
| Qiskit (reference, in-process) | https://quantum.cloud.ibm.com/docs/en/api/qiskit | None (library) | n/a | Excellent | `qiskit==2.6.1` |
| Qiskit Aer (benchmark, in-process) | https://qiskit.github.io/qiskit-aer/ | None (library) | n/a | Good | `qiskit-aer==0.18.2` |
| IBM Quantum Runtime (future hardware backend) | https://quantum.cloud.ibm.com/docs/ | IBM Cloud API key | Plan-based QPU minutes | Good | qiskit-ibm-runtime. Out of scope for V1. |
| Caddy admin API | https://caddyserver.com/docs/api | Local socket only (disable or bind to localhost) | n/a | Good | n/a |

**Key Takeaways:**
- V1 has no third-party network APIs: no email, no payments, no webhooks. Every integration is in-process or on the internal Compose network, which keeps the attack surface small.
- Programmatic access needs tenant-scoped API keys (prefix + hashed secret) on top of web sessions so the SDK can submit jobs to the platform.

---

### 3.9 — Architecture & Design Patterns

| Pattern | Source | Why It Applies |
|---------|--------|----------------|
| Pure-core / ports-and-adapters SDK | Qiskit/Cirq package layout (https://github.com/Qiskit/qiskit, https://github.com/quantumlib/Cirq) | `qai_sdk` (circuit, gates, simulator, algorithms, evolution) is a pure-Python library with zero web/DB imports. The FastAPI platform wraps it, so the SDK stays reproducible on Windows on its own. |
| Backend interface (`Simulator` protocol) | Aer methods (https://qiskit.github.io/qiskit-aer/stubs/qiskit_aer.AerSimulator.html) | Statevector now; tensor-network and hardware later behind the same `run(circuit, shots, seed)` contract. |
| Differential testing against a reference | arXiv:2401.09076 benchmarking method; Qiskit `Statevector.equiv` | Every supported gate and circuit is compared with Qiskit up to global phase (atol 1e-10). Sampling is compared with chi-square/G-tests at fixed α. |
| Seeded RNG injection | NumPy `Generator` (https://numpy.org/doc/stable/reference/random/generator.html) | Seeds are explicit parameters, never global state. Record seed, versions and platform in the evidence ledger. |
| Governed acceptance gate (V(S)=1 ∧ J(S) ≤ J(Ψn) − ε) | Original RESEARCH.md §Success Metrics | Implemented as a pure function over candidate evidence; append-only decision log; 30-seed statistical comparison with uncertainty for research claims. |
| PostgreSQL job table + `FOR UPDATE SKIP LOCKED` lease claim | PostgreSQL docs (https://www.postgresql.org/docs/current/sql-select.html); Procrastinate | Simulation jobs, scheduled benchmarks, retention sweeps: idempotent claim, lease expiry, stuck-job watchdog, no extra broker. |
| Lease / heartbeat | Fowler, Patterns of Distributed Systems (https://martinfowler.com/articles/patterns-of-distributed-systems/lease.html) | Stuck-job watchdog requeues or fails jobs whose lease expired (injected clock for tests). |
| Process-pool CPU isolation | `concurrent.futures` (https://docs.python.org/3/library/concurrent.futures.html) | Simulation runs in worker processes with memory/time caps (qubits ≤ 20 → 16 MiB state; reject above limit before allocation). |
| Pooled multi-tenancy with RLS defence-in-depth | Azure tenancy models (https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models); PostgreSQL RLS | `tenant_id` on every row, enforced in the repository layer and by RLS policies via `SET LOCAL app.tenant_id`. |
| Transactional outbox for audit events | microservices.io (https://microservices.io/patterns/data/transactional-outbox.html) | Audit log rows written in the same transaction as the state change, so no lost audit events. |
| Stateless app tier behind a load balancer | Caddy `reverse_proxy` lb + health checks | Required for 50k concurrent users: N FastAPI and N Next.js replicas, sessions in the DB/Redis, not in memory. |
| Nonce CSP via `proxy.ts` | Next.js CSP guide | Strict CSP without `unsafe-inline`; costs dynamic rendering. |

**Key Takeaways:**
- The decisive structural choice: **SDK core is a standalone package; the platform is a governed service around it.** It satisfies both the "Windows-reproducible SDK" criterion and the "web, multi-tenant, 50k concurrent" constraints without coupling them.
- `has_scheduled_work` is met with a Postgres-native queue (Procrastinate periodic tasks or a custom SKIP LOCKED table) plus an injected clock, an idempotent claim and an observable outcome row per run.

---

## 4. Technology Stack Candidates

| Layer | Option A | Option B | Recommendation | Rationale |
|-------|----------|----------|----------------|-----------|
| Language (SDK + API) | Python 3.14 (3.14.10) | Python 3.13 (3.13.18) | **Python 3.14** | NumPy 2.5 / SciPy 1.18 need >=3.12; every dependency ships cp314 Windows wheels; EOL 2030-10. 3.15.2 is out but brand-new; reassess at the next minor. |
| Language (frontend) | TypeScript 5.x | TypeScript 7.0.2 (npm latest) | **TypeScript 5.9.x pinned via Next's supported range (≥5.1)** | TS 7 (native Go compiler) is new; confirm Next 16.4 compatibility before adopting (see §6). |
| Numerical core | NumPy 2.5.4 + SciPy 1.18.1 | Numba / C extension | **NumPy + SciPy** | Matches "numpy statevector up to ~20 qubits". 2^20 × 16 B = 16 MiB, well within limits; vectorized tensordot/einsum gate application. |
| Reference / benchmark | qiskit 2.6.1 + qiskit-aer 0.18.2 (optional extra) | Cirq 1.7.0 | **Qiskit + Aer** | Mandated by success criteria; Windows wheels verified. |
| Backend framework | FastAPI 0.162.0 + uvicorn 0.54.0 | — | **FastAPI** (hard constraint) | Lifespan API, DI, OpenAPI. |
| ORM / DB driver / migrations | SQLAlchemy 2.1.6 async + asyncpg 0.32.0 + Alembic 1.20.1 | SQLAlchemy sync + psycopg 3 | **Async + asyncpg** | Avoids the Windows ProactorEventLoop issue; high concurrency. |
| Validation / config | Pydantic 2.14.2 + pydantic-settings 2.16.2 | — | **Pydantic v2** | FastAPI native. |
| Database | PostgreSQL 18.6 (`postgres:18.6`) + streaming replica | PostgreSQL 17.11 | **PostgreSQL 18.6** | Current major; RLS, SKIP LOCKED; EOL 2030-11. |
| Job queue / scheduler | Procrastinate 3.10.0 (Postgres-native, periodic tasks) | arq 0.28.0 (Redis, asyncio) / RQ 3.0.0 / Celery 5.6.3 / Dramatiq 2.2.1 | **Procrastinate** (or a custom SKIP LOCKED table if the architect wants zero framework) | No extra broker durability concern, transactional enqueue with business data, periodic tasks built in. arq is stale-ish (last release 2026-04); Celery is heavy; Dramatiq is LGPL. |
| Rate limiting / cache | `limits` 5.8.0 async + Valkey 9.1.3 / Redis 8.10.3 | slowapi 0.1.10 | **`limits` (or Lua token bucket) on Valkey** | slowapi's redis extra pins redis<4; Valkey is BSD-3 versus Redis 8's AGPL/SSPL/RSAL. redis-py 8.1.0 client works with both. |
| Password hashing | argon2-cffi 25.1.0 (Argon2id) | bcrypt | **argon2-cffi** | OWASP-preferred. |
| Frontend | Next.js 16.4.3 App Router + React 19.3.1 + React Context | — | **Next.js 16** (hard constraint) | `proxy.ts` for auth redirects and CSP nonce; Node 24 LTS (24.22.2) runtime image; Node 26 LTS (26.13.0) just entered LTS. |
| Styling | Tailwind CSS 4.3.3 | CSS Modules | **Tailwind 4** | Design tokens map directly to `@theme` variables (the original doc gives a v3-style config, so the architect must translate it). |
| Reverse proxy / TLS | Caddy 2.11.7 (`tls internal` local, ACME in prod) | Traefik / nginx | **Caddy** | Automatic HTTPS, simple lb + health checks. |
| Hosting / Infra | Docker Compose v5.6.1 | Kubernetes | **Compose** (hard constraint) | Replicas via `deploy.replicas`; document the single-host ceiling for 50k concurrent (see R4). |
| Auth | First-party email+password sessions (DB-backed, Argon2id) + tenant API keys | OIDC (Keycloak) | **First-party sessions + API keys** | No external IdP mandated; RBAC per tenant (owner/admin/member/viewer). SSO can be added later. |
| Testing | pytest 9.2.0, hypothesis 6.170.14, coverage ≥ 90 % | — | **pytest + hypothesis** | Property tests for unitarity/normalization; Playwright for web E2E. |
| Logging / audit | structlog 26.1.0 (JSON) + append-only `audit_events` table | — | **structlog** | Structured logs; audit in DB. |
| CI/CD | GitHub Actions matrix (windows-latest + ubuntu-latest) | — | **GitHub Actions** | Proves the Windows-reproducibility criterion on every change. |

**Verified pins (source of truth for the architect):**

| Package / Image | Version | Proof URL |
|-----------------|---------|-----------|
| python | 3.14.10 | https://endoflife.date/python (cycle 3.14 latest 3.14.10) / https://www.python.org/downloads/ |
| numpy | 2.5.4 | https://pypi.org/project/numpy/2.5.4/ |
| scipy | 1.18.1 | https://pypi.org/project/scipy/1.18.1/ |
| qiskit | 2.6.1 | https://pypi.org/project/qiskit/2.6.1/ |
| qiskit-aer | 0.18.2 | https://pypi.org/project/qiskit-aer/0.18.2/ |
| fastapi | 0.162.0 | https://pypi.org/project/fastapi/0.162.0/ |
| uvicorn | 0.54.0 | https://pypi.org/project/uvicorn/0.54.0/ |
| sqlalchemy | 2.1.6 | https://pypi.org/project/SQLAlchemy/2.1.6/ |
| asyncpg | 0.32.0 | https://pypi.org/project/asyncpg/0.32.0/ |
| alembic | 1.20.1 | https://pypi.org/project/alembic/1.20.1/ |
| pydantic | 2.14.2 | https://pypi.org/project/pydantic/2.14.2/ |
| pydantic-settings | 2.16.2 | https://pypi.org/project/pydantic-settings/2.16.2/ |
| argon2-cffi | 25.1.0 | https://pypi.org/project/argon2-cffi/25.1.0/ |
| limits | 5.8.0 | https://pypi.org/project/limits/5.8.0/ |
| slowapi (if used) | 0.1.10 | https://pypi.org/project/slowapi/0.1.10/ |
| redis (redis-py) | 8.1.0 | https://pypi.org/project/redis/8.1.0/ |
| procrastinate | 3.10.0 | https://pypi.org/project/procrastinate/3.10.0/ |
| arq (alt) | 0.28.0 | https://pypi.org/project/arq/0.28.0/ |
| rq (alt) | 3.0.0 | https://pypi.org/project/rq/3.0.0/ |
| celery (alt) | 5.6.3 | https://pypi.org/project/celery/5.6.3/ |
| dramatiq (alt) | 2.2.1 | https://pypi.org/project/dramatiq/2.2.1/ |
| structlog | 26.1.0 | https://pypi.org/project/structlog/26.1.0/ |
| pytest | 9.2.0 | https://pypi.org/project/pytest/9.2.0/ |
| hypothesis | 6.170.14 | https://pypi.org/project/hypothesis/6.170.14/ |
| httpx | 0.28.1 | https://pypi.org/project/httpx/0.28.1/ |
| cirq (reference only) | 1.7.0 | https://pypi.org/project/cirq/1.7.0/ |
| pennylane (reference only) | 0.45.1 | https://pypi.org/project/pennylane/0.45.1/ |
| quimb (next phase) | 1.15.0 | https://pypi.org/project/quimb/1.15.0/ |
| next | 16.4.3 | https://www.npmjs.com/package/next |
| react / react-dom | 19.3.1 | https://www.npmjs.com/package/react |
| tailwindcss | 4.3.3 | https://www.npmjs.com/package/tailwindcss |
| node (image) | 24.22.2 LTS "Krypton" (26.13.0 LTS "Lithium" available) | https://nodejs.org/dist/index.json |
| postgres (image) | 18.6 | https://hub.docker.com/_/postgres |
| caddy (image) | 2.11.7 | https://github.com/caddyserver/caddy/releases/tag/v2.11.7 |
| valkey (image) | 9.1.3-alpine | https://hub.docker.com/r/valkey/valkey |
| redis (image, alt) | 8.10.3-alpine | https://hub.docker.com/_/redis |
| docker compose | v5.6.1 | https://github.com/docker/compose/releases |

---

## 5. Risk & Unknowns Register

| ID | Unknown / Risk | Severity | Mitigation | Status |
|----|----------------|----------|------------|--------|
| R1 | **Qubit-ordering / endianness mismatch** between QAI and Qiskit silently corrupts differential tests (documented real bug class, qiskit-optimization #92) | high | Adopt Qiskit's little-endian convention (qubit 0 = LSB) in the SDK, document it, and add explicit boundary tests (asymmetric states such as X on q0 only, CX direction) | open |
| R2 | **Sampling reproducibility across platforms/versions**: NumPy `Generator` streams and SciPy COBYLA (PRIMA) iterates can change between versions; Windows vs. Linux BLAS can differ in last-ulp results | high | Pin exact versions; seeds and environment captured in the evidence ledger; reproducibility asserted within the same pinned environment; cross-platform comparisons use tolerances and distribution tests, not bit-equality; CI matrix on windows-latest + ubuntu-latest | open |
| R3 | **CPU-bound simulation on the API event loop** (20 qubits × deep circuits, QAOA loops of hundreds of evaluations) starves the API | high | All simulation runs in separate worker processes via the job queue; API only enqueues; per-tenant quotas; hard caps on qubits (≤ 20), depth, shots, optimizer iterations and wall time, validated before allocation | open |
| R4 | **"50k+ concurrent" on Docker Compose**: one Compose host has finite CPU/RAM; simulations are compute-heavy; nonce CSP disables static caching of Next pages | high | Stateless replicas behind Caddy lb; separate web/API/worker scaling; read replica; connection pooling (PgBouncer or SQLAlchemy pool limits sized against `max_connections`); load test in the Security/QA gate; document that 50k concurrent means sessions/API traffic, with simulation throughput bounded by worker count and queued; multi-host is an ADR item | open |
| R5 | **Cross-tenant data leakage** (experiments, benchmark results, audit logs) | high | `tenant_id` on every table + repository-layer enforcement + PostgreSQL RLS via `SET LOCAL`; tests that try cross-tenant ID access (IDOR) on every endpoint; UUIDv7 / random IDs | open |
| R6 | **Resource-exhaustion DoS via circuit payloads** (huge qubit counts → 2^n allocation, giant gate lists, QASM bombs) | high | Pydantic limits (max qubits 20, max gates, max params, max body size at Caddy); reject before allocation; worker memory limit (`mem_limit`) and timeout; rate limits on submit | open |
| R7 | **Evolution-engine acceptance gaming**: candidates that bypass V(S) or overfit J on the evaluation seeds | med | V and J computed by the platform, never by the candidate; held-out seeds; ε and 30-seed protocol fixed in config with an audit trail; append-only decision log; no code execution of candidate artifacts (candidates are parameter/config sets, not code) | open |
| R8 | **Rate-limiter dependency drift**: slowapi's redis extra pins `redis<4` | med | Use `limits` 5.8.0 async storage or a custom Lua token bucket; fail-closed on auth routes and fail-open with a local fallback elsewhere (decision for ADR) | open |
| R9 | **Redis licensing** (Redis 8 = RSALv2/SSPLv1/AGPLv3) | low | Use Valkey 9 (BSD-3) or run Redis unmodified as a separate service (AGPL obligations only apply on modification) | open |
| R10 | **Caddy `tls internal` trust** in containers and on Windows hosts; E2E tests failing TLS | med | Export the root CA from the Caddy volume; `SSL_CERT_FILE` / `NODE_EXTRA_CA_CERTS` for test clients; document `caddy trust`; production uses ACME | open |
| R11 | **Qiskit API churn** within 2.x (primitives/BitArray) breaks reference tests | low | Reference extra pinned (`qiskit==2.6.1`, `qiskit-aer==0.18.2`); tests marked `reference`; skip cleanly if absent but the release gate requires them to run | open |
| R12 | **Numerical precision**: accumulated float error in deep circuits exceeds 1e-10 | med | complex128 throughout; renormalization check (‖ψ‖ = 1 ± 1e-12) as a test invariant, not silent renormalization; gate unitarity property tests (hypothesis) | open |
| R13 | **Scheduled-work failure modes** (has_scheduled_work): double-run of scheduled benchmarks, stuck jobs never recovered, retention deleting audit evidence | high | Idempotent claim (unique key per schedule window); lease + watchdog with injected clock; retention excludes audit/evidence-ledger tables or follows a separate policy; one test per trigger asserting effect, not liveness | open |
| R14 | **Over-claiming performance** vs. Aer (brand voice forbids it) | med | Benchmarks published with methodology and raw data; UI copy reviewed; "Aer is faster" stated plainly where true | open |
| R15 | **Windows dev experience**: Docker Desktop/WSL2 required for the platform; SDK must work natively | med | SDK installable by `pip install` on native Windows with no compiler; platform via Docker Desktop; CI proves both | open |
| R16 | **Supply-chain risk** (typosquatting, compromised wheels) | med | Hash-pinned lockfiles (`uv lock` / `pip-compile --generate-hashes`, `package-lock.json`); `pip-audit` / `npm audit` in the Security gate; SBOM | open |

### 5.1 Threat Model (STRIDE over trust boundaries)

**Trust boundaries:**
- **TB1** Internet → Caddy (TLS termination, body-size limits).
- **TB2** Caddy → Next.js web tier (SSR, `proxy.ts`).
- **TB3** Caddy/Next → FastAPI API (session cookie or API key).
- **TB4** API → PostgreSQL / Valkey (internal network).
- **TB5** API → job queue → simulation workers (untrusted circuit payloads executed as computation).
- **TB6** Scheduler / watchdog → DB (no user present).
- **TB7** Tenant ↔ tenant (logical boundary inside shared tables).
- **TB8** Evolution candidates → acceptance gate G.

| STRIDE | Boundary | Threat | Mitigation |
|--------|----------|--------|------------|
| **S**poofing | TB1/TB3 | Credential stuffing, session theft, stolen API keys | Argon2id; login rate limits plus progressive delay; HttpOnly+Secure+SameSite cookies; session rotation on login; API keys stored hashed with prefix lookup, scoped to a tenant, revocable; HSTS |
| **S**poofing | TB4 | Rogue container talks to the DB | Internal-only Compose network; DB/Valkey not published to host in prod; strong passwords from secrets/env |
| **T**ampering | TB3/TB5 | Malicious circuit JSON/QASM (oversized, NaN params, unknown gates) | Strict Pydantic schemas, allow-listed gate set, finite-float validation, size caps at Caddy and API |
| **T**ampering | TB8 | Candidate falsifies its own validation result or objective | Gate recomputes V and J server-side from stored inputs; results hashed into the evidence ledger; append-only `governance_decisions` |
| **T**ampering | TB2 | XSS via circuit labels / experiment names | React escaping; nonce CSP with `strict-dynamic`; no `dangerouslySetInnerHTML`; RFC 9457 errors without reflected HTML |
| **R**epudiation | all | User denies submitting a job or approving a gate | Append-only `audit_events` (actor, tenant, action, target, IP, UA, request_id, timestamp) written in the same transaction; no UPDATE/DELETE grant for the app role on audit tables |
| **I**nformation disclosure | TB7 | IDOR across tenants; leaking benchmark data | Tenant scoping in every query plus RLS; 404 (not 403) for foreign IDs; per-endpoint cross-tenant tests |
| **I**nformation disclosure | TB1 | Stack traces, OpenAPI, Caddy admin exposed | Generic errors in prod; docs behind auth or disabled; Caddy `admin off` or localhost; security headers |
| **D**enial of service | TB1/TB5 | Submit floods, 2^n memory blow-up, long QAOA loops | Per-tenant and per-IP token buckets; per-tenant concurrent-job quotas; qubit/depth/shots/iteration caps; worker `mem_limit` and timeouts; queue back-pressure (429 + Retry-After) |
| **D**enial of service | TB6 | Watchdog/retention jobs locking hot tables | Batch deletes with LIMIT; SKIP LOCKED; run off-peak; statement_timeout |
| **E**levation of privilege | TB3 | Member performs admin/governance actions (approve release gate) | RBAC checked in a FastAPI dependency on every route; governance approval requires a distinct reviewer role and cannot be self-approved; Server Functions re-check authz (Next docs warning) |
| **E**levation of privilege | TB5 | Code execution via payload (pickle, eval of expressions in params) | No pickle/eval anywhere; parameters are numbers only; workers run as non-root with a read-only filesystem |

---

## 6. Research Gaps

- [ ] Gap 1: **TypeScript 7.0 compatibility with Next.js 16.4** not verified. Pin TS 5.9.x unless the architect confirms (non-blocking).
- [ ] Gap 2: **Exact Aer baseline numbers on the target Windows hardware**: no public Windows benchmark found. The benchmark harness must generate them; there is no target to beat, only to report (non-blocking).
- [ ] Gap 3: **Procrastinate vs. custom SKIP LOCKED table**: Windows-native worker behaviour (LISTEN/NOTIFY via asyncpg on the ProactorEventLoop) unverified. Workers run in Linux containers, so this only matters for native-Windows dev (non-blocking; ADR).
- [ ] Gap 4: **`limits` async Redis/Valkey storage API in 5.8.0**: the exact storage URI scheme and the client library it requires (coredis vs. redis-py) need confirmation from https://limits.readthedocs.io before SPEC (non-blocking).
- [ ] Gap 5: **Sizing for "50k concurrent"**: what fraction submit simulations? A throughput assumption (e.g. jobs/sec and p95 queue wait) must be set in SPEC and validated by load test (non-blocking but must be decided in ARCHITECTURE).
- [ ] Gap 6: **Statistical test choice and α for "measurement accuracy"** (chi-square vs. G-test, multiple-comparison correction across the test suite) must be fixed in SPEC (non-blocking).
- [ ] Gap 7: **Design-token conflict**: the art direction says dark mission-control (#080B12), the token table says "Light only", and a design template exists. Precedence per DESIGN.md: template → art direction → archetype. Must be resolved and logged in DECISIONS.md (non-blocking).
- [ ] Gap 8: **Tailwind v4 vs. the v3-style `tailwind.config.ts`** supplied in the brief: translate to `@theme` CSS variables (non-blocking).
- [ ] Gap 9: **Community evidence from Stack Overflow/Reddit** could not be fetched (crawler blocked). GitHub/Discourse sources were used instead (non-blocking).
- [ ] Gap 10: **Data retention policy** is not specified in Build Constraints (needed for the retention sweep). Default proposal: job payloads/results 90 days; audit and evidence ledger kept indefinitely. Log as ADR (non-blocking).
- [ ] Gap 11: **QASM import scope**: whether V1 parses OpenQASM 2 input or only exports it. Recommend export-only in V1 to shrink the parser attack surface (non-blocking).
- [ ] Gap 12: **Email delivery** for password reset/invites is not specified. V1 can use admin-issued invites/reset tokens with no email provider (non-blocking; ADR).

**Blocker?** No. All gaps are design decisions the architect can resolve with logged ADRs; none invalidate the stack or the approach.

---

## 7. Summary & Recommendation

The SDK core is technically straightforward and well-bounded. A NumPy complex128 statevector engine up to 20 qubits needs only 16 MiB per state. Grover and QAOA have closed-form test oracles (quant-ph/9605043, arXiv:1411.4028), and Qiskit 2.6.1's `Statevector.equiv` and `StatevectorSampler` provide an independent reference. Every required package ships Windows x86-64 wheels for Python 3.14, so the Windows-reproducibility criterion is achievable with exact pins and a windows+linux CI matrix.

The platform constraints (Next.js 16 with `proxy.ts`, FastAPI 0.162 + SQLAlchemy 2.1 async/asyncpg, PostgreSQL 18.6, Caddy 2.11.7, Compose v5.6.1) are all current and compatible. The real engineering risks are elsewhere:
- CPU isolation of simulations from the API.
- Tenant isolation.
- Payload-driven DoS.
- Endianness and reproducibility discipline.
- Honest scaling claims for "50k concurrent" on Compose.

Each of these has a known mitigation pattern: worker processes with a Postgres SKIP LOCKED/Procrastinate queue, RLS, hard caps, a documented convention, and load tests.

Recommended shape: a standalone `qai-sdk` Python package, with Qiskit/Aer only in a `[reference]` extra, wrapped by a governed, multi-tenant FastAPI service. The service adds an append-only audit log, an evidence ledger, and an evolution acceptance gate that recomputes V(S) and J(S) server-side.

**Go / No-Go Decision:** `GO`

Reasoning: no blocking unknowns. All hard constraints are satisfiable with verified, current, permissively licensed components. The open gaps (§6) are ADR-level decisions, not feasibility questions.

**Next Step:** → [ARCHITECTURE.md](./ARCHITECTURE.md)

---

## Pre-Build Review (Step 3b — applied 2026-10-31)

1. `Key Workflows W7` — "Planned enterprise interfaces" vs. `multi_tenant: true` + web platform — **Fix:** V1 ships tenant-scoped API keys + `/api/v1` REST + audit as the enterprise interface; agent-specific connectors deferred (ADR-05).
2. `Key Workflows W1/W2` — "Compile" step and "tensor-network / hardware backends" vs. statevector-only V1 — **Fix:** V1 ships a minimal compile pass (validate + gate decomposition + adjacent-inverse cancellation); tensor-network and hardware backends deferred behind a `Backend` protocol (ADR-06).
3. `Design Language` — config "Light only" vs. dark-first binding template — **Fix:** template wins, single dark theme (ADR-01).
4. `Success Criteria` — "Windows-based SDK" vs. `container_strategy: Docker Compose` (Linux containers) — **Fix:** SDK pure-Python, pip-installable on native Windows and tested on windows+linux CI; platform runs via Docker Desktop (ADR-07).
5. `Success Criteria` — "measurable performance against defined baselines" has no threshold — **Fix:** SPEC §9 defines: simulator fidelity ≤1e-10 vs Qiskit; QAOA approximation ratio reported vs exact/greedy baseline; API p95 < 300 ms on reads at target load (ADR-08).
6. `W5/W6` — "release gate cannot be bypassed" vs. `review_gates: auto` — **Fix:** `review_gates` governs this build pipeline only; the product's release-readiness gate is computed from evidence server-side and requires a distinct reviewer role (no self-approval).
7. `scale: large` — "50k+ concurrent" on single-host Compose — **Fix:** horizontal stateless replicas, read replica, cache, queue back-pressure in compose; 50k concurrent = sessions/API traffic, simulation throughput bounded by workers; multi-host is documented as a residual (ADR-09).
8. `Gap 10/12` — retention + email unspecified — **Fix:** retention 90 days for job payloads/results, audit + evidence kept; no email provider in V1, admin-issued invite/reset tokens (ADR-10).

## Re-Entry R1 — Enhancement: Hybrid Photonic Pipeline (2026-11-01)

**Requirement (user, verbatim):** "continue with ENHANCEMENT Classical AI > Photonics > Quantum Operations > Validation".

**Interpretation.** One new end-to-end workflow, run as one audited job. Each stage feeds the next:
1. **Classical AI.** A small neural network learns the inverse map from a target per-qubit probability p to the RY angle θ = 2·asin(√p). It runs on the classical side, before any quantum work.
2. **Photonics.** The predicted angles are compiled to a linear-optical interferometer (beam splitters and phase shifters) in dual-rail encoding: qubit q is one photon in modes (2q, 2q+1). The interferometer is simulated exactly in Fock space.
3. **Quantum operations.** The same angles are run as a gate-model circuit (`ry` per qubit, compiled and simulated by the existing statevector engine).
4. **Validation.** The platform checks the two physical models against each other, and both against the target. The result is a hashed evidence record, so a release can be gated on it (W5).

**Physics and algorithms.**
- In a passive linear interferometer with unitary U on m modes, a single-photon creation operator transforms as a†ᵢ → Σⱼ Uⱼᵢ a†ⱼ. The amplitude from input occupation S to output occupation T is ⟨T|U|S⟩ = Perm(U_{T,S}) / √(∏ sᵢ! ∏ tⱼ!), where U_{T,S} repeats row j tⱼ times and column i sᵢ times (Scheel 2004; Aaronson & Arkhipov 2011).
- The permanent is #P-hard in general. Ryser's formula costs O(2ᵏ·k²) for k photons, which is trivial at the cap of 4 photons and 8 modes (C(11, 4) = 330 output patterns).
- Hong–Ou–Mandel interference (two photons on a 50:50 beam splitter never exit in different ports) is the standard correctness check for a permanent-based simulator.
- Any single-qubit unitary is realised exactly on one dual-rail pair by a Mach–Zehnder arrangement of phase shifters and a beam splitter (ZYZ: Rz·Ry·Rz, with Ry(γ) a real beam splitter of angle γ/2). Arbitrary m-mode unitaries decompose into O(m²) beam splitters (Reck et al. 1994; Clements et al. 2016).
- **Entangling gates are out of reach for deterministic linear optics.** A CNOT on dual-rail qubits needs ancilla photons, measurement and feed-forward, and it succeeds only probabilistically (Knill–Laflamme–Milburn 2001). The photonic stage therefore accepts single-qubit gates only, and it reports a multi-qubit gate as a field error instead of approximating it.

**Classical AI choice.**
- Use a 1→H→1 tanh MLP in NumPy, trained with full-batch Adam and seeded `PCG64` initial weights.
- It adds no dependency. PyTorch or JAX would add about 1 GB of wheels to a pure SDK for a 50-parameter model, and would make bitwise reproducibility on Windows and Linux harder.
- Training on angle-space MSE is reliable. Training on marginal-space MSE (sin²(θ/2) − p)² is not identifiable: sin² is periodic, and in a scratch sweep 4 of 15 seeds converged to wrapped angles with a marginal error of 1.0.
- Measured (scratch prototype, 129 training points):
  - hidden 16 with 1,000 epochs takes about 0.4–0.7 s and has a worst marginal error of 0.025 over seeds 0–39, so the default tolerance is 0.03;
  - 100 epochs gives 0.03–0.11, so an undertrained surrogate fails validation honestly.

**Adjacent products.** Xanadu Strawberry Fields (continuous-variable and Fock backends) and Quandela Perceval (discrete-variable linear optics, permanent-based) are the reference designs. QAI does not depend on either. Like Qiskit, they could serve as an optional differential reference later.

**Risks.**
- The surrogate is approximate by design. The validation tolerance makes that explicit, and the pipeline returns `fail` rather than hiding the error.
- The photonic model is ideal: lossless, perfectly indistinguishable photons, and no detector inefficiency. Noise models are a non-goal for V1.
- Training CPU cost is bounded by `epochs ≤ 5000` and `hidden ≤ 64` (about 5 s worst case), and it runs in the existing isolated worker child.

**Sources.**
- E. Knill, R. Laflamme, G. J. Milburn, "A scheme for efficient quantum computation with linear optics", Nature 409, 46–52 (2001).
- S. Aaronson, A. Arkhipov, "The computational complexity of linear optics", STOC 2011 / Theory of Computing 9 (2013).
- S. Scheel, "Permanents in linear optical networks", arXiv:quant-ph/0406127 (2004).
- H. J. Ryser, *Combinatorial Mathematics*, MAA (1963), permanent formula.
- M. Reck, A. Zeilinger, H. J. Bernstein, P. Bertani, PRL 73, 58 (1994); W. R. Clements et al., Optica 3, 1460 (2016).
- C. K. Hong, Z. Y. Ou, L. Mandel, PRL 59, 2044 (1987).
- D. P. Kingma, J. Ba, "Adam: A Method for Stochastic Optimization", ICLR 2015.

## Re-Entry R3 — Enhancement: AI Simulation (circuit surrogates) (2026-11-02)

**Requirement (user, verbatim):** "Let's add AI powered simulation https://simutechgroup.com/ai-simulation/".

**What the reference describes.** The page is a consulting offer (SimuTech Group, an Ansys channel partner). It promises to "Automate the Simulation Loop", to "Make Your Data AI-Ready" ("train models, and turn simulation results into reliable, reusable assets") and to "Integrate AI Simulation Fast with Ready-Made Models". It names no method. The established pattern behind that language is the **simulation surrogate** (Ansys SimAI, physics-ML surrogates generally): sample the design space, run the full solver on the samples, train a model on the results, use the model to predict and explore new designs quickly, and confirm the chosen designs with the full solver.

**Interpretation for QAI.** The "design" is a saved circuit and its rotation angles are the design parameters. One audited job (W9):
1. **Design of experiments.** Latin-hypercube samples of the chosen angles in [value − span, value + span].
2. **Exact simulation.** The statevector solver produces the full output distribution for every sample (the training data).
3. **Training.** A surrogate learns angles → distribution. 20 % of the samples are held out.
4. **Prediction and exploration.** The surrogate scores thousands of candidate settings for the probability of a target outcome; the top-k are re-run on the exact solver.
5. **Validation.** Held-out accuracy, prediction-vs-exact agreement, normalisation and "no worse than the starting design" become a hashed evidence record, so a release can be gated on it (W5).

**Model choice — measured (scratch prototype, NumPy, this machine).**
- **Neural network (rejected as the default).** A tanh MLP on [sin θ, cos θ] features with a softmax output, trained with Adam on cross-entropy:
  - 2–3 parameters, 205 training samples: held-out mean TVD 0.003–0.011, 63–109× faster than the exact solver per evaluation.
  - 6 parameters over ±π: held-out mean TVD 0.16–0.57 even with 2,048 samples (training TVD 0.005: it overfits). This is no better than predicting the mean distribution (0.51).
  - 6–8 parameters over ±π/4: 0.025–0.067. Usable only locally, and only with tuning.
- **Physics-informed Fourier surrogate (chosen).** Every parametrised gate in the allow-list (rx, ry, rz, p, cp, and each of u's three angles) enters the state through e^{±iθ/2} or e^{iθ}·projector. Each probability is therefore a real trigonometric polynomial of degree ≤ 1 in each angle: P(θ) = Σ_k c_k Π_j b_{k_j}(θ_j) with b ∈ {1, cos θ, sin θ}. That is 3^d basis functions for d angles. Ridge regression (λ = 1e-9) on that basis learns the c_k:
  - Once the training count exceeds 3^d, the model is exact up to floating point: held-out max TVD 1e-10 to 8e-9 for d = 2, 3 and 6 (seeds 0 and 1).
  - Below 3^d it is underdetermined and fails honestly: d = 6 with 410 training samples gives a mean TVD of 0.24; d = 8 with 3,277 gives 0.25.
  - Per evaluation it is 230–500× faster than the exact solver at d ≤ 3, and 25–33× at d = 6. At d = 8 it is 4–5× and fitting takes about 8 s; at d = 9 there are 19,683 features and fitting takes 96 s.
  - **Cap: d ≤ 6.** The default sample count is max(64, 2·3^d), capped at 4,096, so the default training set (80 %) is ≥ 1.6·3^d.
- The surrogate is still learned from data, like any other: it is fitted only to the solver's outputs and never reads the circuit's gates. The physics fixes the hypothesis class (which features exist), not the answer.

**Exploration.** Candidates are another seeded Latin hypercube in the same box, with the starting design included as candidate 0. Only the target column of the weight matrix is evaluated, in chunks of 4,096, so 65,536 candidates at d = 6 cost about 0.2 s and bounded memory. The top-k by predicted P(target) are verified exactly.

**Risks.**
- **Cost.** It grows with 3^d × 2^n. The caps are d ≤ 6, n ≤ 10, samples ≤ 4,096 and candidates ≤ 65,536. The worst case is 4,096 exact 10-qubit simulations of a 2,000-op circuit plus a 729×729 solve. Simulating each sample separately measured about 300 s, over the 120 s job timeout. So the exact stage batches the samples: ops before the first design angle run once, and each run of fixed ops between angles (at most 7) is fused into one dense 2^n unitary and applied to the whole batch by matrix product. That measured 47.7 s for the worst case with single-qubit gates (10 qubits, 2,000 ops, 6 angles, 4,096 samples, 65,536 candidates, top-k 10); with ccx gates Security pass 6 measured about 65 s single-threaded, 54 % of the 120 s timeout (ADR-44), and 0.4 s for the §9 4-qubit default.
- **Shared angles.** If one logical parameter appears in several gates (QAOA's γ on every edge), each occurrence is a separate design parameter. Tying parameters is a non-goal for V1.
- **Noise.** None: the exact solver is ideal, as in W2.

**Sources.**
- SimuTech Group, "AI Simulation" (accessed 2026-11-02): https://simutechgroup.com/ai-simulation/
- M. D. McKay, R. J. Beckman, W. J. Conover, "A comparison of three methods for selecting values of input variables in the analysis of output from a computer code", Technometrics 21 (1979). Latin hypercube sampling.
- M. Schuld, R. Sweke, J. J. Meyer, "Effect of data encoding on the expressive power of variational quantum machine-learning models", PRA 103, 032430 (2021). Circuit outputs as Fourier series in the angles.
- K. Mitarai, M. Negoro, M. Kitagawa, K. Fujii, "Quantum circuit learning", PRA 98, 032309 (2018). Single-angle dependence is sinusoidal.
- A. E. Hoerl, R. W. Kennard, "Ridge regression", Technometrics 12 (1970).

## Re-Entry R4 — Enhancement: Orb Lab (ORB-0 energy validation) and the Evolution Engine demo (2026-11-03)

**Requirement (user).** An infographic, "Ψ ◈ QAI // ORB LAB — Orb Energy Theory Validation Platform", to "ingest and update program accordingly", followed by "start R4". The image names:
- six things to test: field confinement, resonance, energy retention, information alignment (Rényi divergence D_α(P‖Q)), QAI optimisation and direct extraction;
- a six-step pipeline: Model, Simulate (Meep / NumPy / SciPy), Optimize (Qiskit + hybrid QAI), Validate (energy, stability, ΔE), Compare (classical vs QAI), Report (pass / fail / inconclusive);
- a core equation set:
  - dU_orb/dt = P_in − P_out − P_loss;
  - U_EM = ∫(ε₀|E|²/2 + |B|²/2μ₀) dV;
  - ΔE = E_initial + E_supplied − E_extracted − E_lost − E_final;
  - min_{h∈H} D_α(P‖Q_h) s.t. C̃(h) ≤ B and Stability(Ψ) ≥ 0;
  - v_α(r, B) and Ψ_{n+1} = G(V(Q(L(Ψ_n, E_n))));
- the badges "Reproducible, Governed, Physically Grounded, Falsifiable".

The user also asked whether the demo should say more about the Evolution Engine. The answer was yes: a landing-page section with the loop and its limits, and a pre-filled example in the app.

**Interpretation: ORB-0, a falsifiable resonant-cavity experiment.** The image names no device, so ORB-0 is the simplest physical system that exercises every equation: an ideal sphere with finitely conducting walls, driven through one port and drained into a load through a second.
1. **Model.**
   - The modes of a perfectly conducting sphere have closed forms. TE_ln resonates at j_l(ka) = 0 and TM_ln at [t j_l(t)]′ = 0 at t = ka. Each family is (2l+1)-fold degenerate.
   - The unloaded Q0 = ωU/P_wall is computed by quadrature of the fields. With the surface resistance R_s = 1/(σδ), it matches Q0 δ/a = 1 for TE and 1 − l(l+1)/x² for TM.
   - The equality of electric and magnetic stored energy at resonance (the U_EM integral split into its two terms) is checked by quadrature too.
   - The driven mode's time-averaged energy density on the x–z plane is the field map.
2. **Simulate.**
   - The driven mode obeys coupled-mode theory: da/dt = −(γ0 + γ_in + γ_ext) a + √(2γ_in) s(t), with U = |a|² and γ = ω/2Q.
   - Supplied (incident minus reflected), extracted and wall-lost energy are integrated as extra ODE states (DOP853, rtol 1e-11). dU/dt = P_in − P_out − P_loss then holds by construction, and ΔE is a measured residual.
   - A closed form for the extracted energy over the pulse and the full ring-down cross-checks the integrator.
3. **Optimize.**
   - The design variable is the extraction coupling Q_ext, chosen from 2^n log-spaced settings (Q0/100 to 100·Q0). The objective is the extracted fraction of the incident energy.
   - Classical: exhaustive evaluation of all 2^n settings, plus a bounded Brent search on log Q_ext as a continuous cross-check.
   - QAI: Dürr–Høyer maximum finding, which is Grover search with a moving threshold on the BBHT iteration schedule. It is simulated exactly on the state vector and stops on a query budget B without peeking at the answer.
4. **Validate.** Physics checks fail the run (see Checks).
5. **Compare.**
   - The image's min_h D_α(P‖Q_h) s.t. C̃(h) ≤ B maps onto the search directly. P is the point mass on the optimum, Q_h is the output distribution after h Grover iterations, and C̃(h) = h oracle queries.
   - For a point-mass P, every Rényi order gives −ln Q_h(opt). The image's integral form is read as this budgeted minimum, v(B) = min_{h ≤ B} D.
   - Success probability against the budget is also measured: classical sampling without replacement gives b/N, and QAI is the empirical rate over 64 seeded repeats.
6. **Report.**
   - `fail` if any physics check fails.
   - `inconclusive` if the physics holds but the optimisers disagree (typically a budget that is too small).
   - `pass` otherwise.
   - When the search used no fewer queries than N, the report says so.
- **Ψ_{n+1} = G(V(Q(L(Ψ_n, E_n)))).** This is the same propose, evaluate, validate and accept loop as the Evolution Engine. The Dürr–Høyer rounds are presented as that sequence: L is the ledger objective, Q the Grover search, V the comparison with the threshold, and G accepting the new best.

**Checks.**
- Physics checks, each of which fails the run:
  - `mode_roots` (|residual| ≤ 1e-10);
  - `q_closed_form` and `equipartition` (≤ 1e-8);
  - `energy_conservation` and `ledger_closed_form` (≤ the tolerance, default 1e-6);
  - `extraction_bound` (E_ext/E_sup ≤ 1 and E_sup/E_inc ≤ 1);
  - `stability` (positive decay rate, monotone ring-down, U_final/U_peak ≤ 1e-12);
  - `grover_circuit_agreement` (the Grover model against the SDK's gate-level `grover_circuit` on the state-vector simulator, ≤ 1e-10).
- Comparison checks, which make the run inconclusive: `optimum_agreement` and `continuous_agreement` (within one grid step).

**Measured (prototype and `qai.orb`, this machine).**
- Mode zeros match Abramowitz & Stegun table 10.6 (TM_1,1 2.743707, TE_1,1 4.493409, TM_2,1 3.870239). The quadrature Q and equipartition errors are ≤ 5e-16.
- A copper sphere with a = 0.1 m has TM_1,1 at 1.3091 GHz with Q0 ≈ 4.02 × 10⁴.
- The ledger residual is about 1e-13 relative and the closed-form agreement about 1e-13.
- The long-pulse optimum tends to the matched coupling 1/(1/Q0 + 1/Q_in). Finite pulses shift it (0.477·Q0 at Q_in = Q0 and a pulse of 10 τ0), which is why the closed form, not the steady-state formula, is the reference.
- Dürr–Høyer success over 200 seeds:
  - budget 8√N gives ≥ 0.985 for N = 8 to 256;
  - budget 4√N gives 0.78–0.83.
  - The default budget is therefore ⌈8√N⌉. At N = 256 that is 128 queries against 256 classical evaluations, where classical sampling at the same budget succeeds with probability 0.5.
  - At N = 8 the default budget (23) exceeds N. There is no advantage at that size, and the report says so.
- A run takes 2–3 s. The mode table is cached after the first run, and the 64-repeat success curve is about 1 s.

**Non-goals and honesty.**
- No plasma physics, no nonlinear media and no Meep/FDTD. Meep has no native Windows build, and the ideal sphere has closed forms that make the model checkable.
- Meep could be added later as an optional backend for non-spherical geometries.
- Nothing produces energy. "Direct extraction" is the fraction of the supplied energy delivered to the load, and it is bounded by 1.
- The oracle is evaluated classically. The quantum resource being compared is oracle queries, not wall-clock time.
- No Qiskit: the Grover search runs on the QAI state-vector simulator and is checked against the SDK's gate-level circuit.

**Evolution Engine demo.**
- The landing page gets a section on the loop (propose, run, validate, accept only if J improves by ε and the checks pass, checkpoint) and states its limits: QAOA MaxCut on ≤ 12 nodes, simulator only, 1–20 iterations.
- The Evolution Engine screen gets a "Load example" button that fills a known-good setup.
- The Orb Lab screen sits in the nav after AI Simulation.

**Sources.**
- J. D. Jackson, *Classical Electrodynamics*, 3rd ed., §8.7–8.9 (Q of a cavity, the spherical cavity) and §9.7 (vector spherical harmonics).
- M. Abramowitz, I. A. Stegun, *Handbook of Mathematical Functions*, table 10.6 (zeros of spherical Bessel functions and their derivatives).
- H. A. Haus, *Waves and Fields in Optoelectronics* (1984), ch. 7: coupled-mode theory of resonators.
- C. Dürr, P. Høyer, "A quantum algorithm for finding the minimum", arXiv:quant-ph/9607014 (1996).
- M. Boyer, G. Brassard, P. Høyer, A. Tapp, "Tight bounds on quantum searching", Fortschr. Phys. 46 (1998).
- A. Rényi, "On measures of entropy and information", Proc. 4th Berkeley Symp. (1961).
- T. van Erven, P. Harremoës, "Rényi divergence and Kullback-Leibler divergence", IEEE Trans. Inf. Theory 60 (2014).

## Re-Entry R5 — Enhancement: Formula Lab (user formulas on the website) (2026-11-07)

**Requirement (user).** "How do I test my formulas with this app", then "I would like to do this with the website". The user chose:
- formulas of all four kinds: an objective to optimise, a physics formula checked against the Orb Lab ledger, an evolution map step Ψₙ₊₁ = f(Ψₙ, Eₙ), and a quantum state or a circuit with formula angles;
- a job with a result page, charts, checks and a signed evidence record;
- math notation such as `sin(7*x)*exp(-x)`, `x^2`, `sqrt`, `log` and `pi`, parsed by a whitelisted grammar without code execution.

**The language (qai.expr).** A recursive-descent parser over a regex tokenizer builds a tree of plain tuples, and an evaluator dispatches on a fixed table of NumPy functions. Python's own parser never sees the text. Measured on the prototype:
- Precedence follows mathematical convention: −x^2 = −9 at x = 3, 2^3^2 = 512, 2**−1 = 0.5. Multi-line definitions chain, and `#` comments are stripped.
- Rejected with a line and column: `2x` (missing operator), `foo(x)`, `sin` without parentheses, the wrong arity, `__import__('os')` (the `_` is not a token), `x.real`, `x[0]`, strings, `;`, `==`, `1e400`, unknown names (listing the available ones), redefining `pi` or a variable, and `i` outside amplitudes.
- A first draft checked depth only when it built a node, so 2,000 `(` characters would have reached Python's recursion limit before the check ran. The check now runs on entry: 2,000 `(`, 1,999 unary minuses and 999 chained `^` are all rejected at column 34–67 with no RecursionError.
- Cost: parsing a 219-term formula takes 1.1 ms, and evaluating it over 65,536 points 41 ms.

**The four modes (qai.formula), measured.**
1. **Objective.** `sin(7*x)*exp(-x)` on [0, 1] at 8 qubits, seed 7: the classical maximum is index 52, and QAI Dürr–Høyer found it at query 40 of 128 (2.7 s, mostly the 64-repeat success curve). A 2-D `(x−0.3)^2 + (y+0.2)^2 + 0.1·sin(9x)` minimised on 10 qubits (32 × 32) was found at query 71 of 256 against 1,024 evaluations (5.0 s). `budget: 4` is inconclusive. `log(x)` on [−1, 1] fails `finite`, which names the first bad x. A constant objective passes trivially.
2. **Physics vs the ledger.** The extracted-energy closed form typed as five formula lines (amp, e1, during, aend, result) matched the DOP853 ledger to 3.2e-13 relative at 8 points and 1.4e-13 at 32 (the ledger's own residual is ≤ 2e-13). The steady-state guess `power*pulse*4*gin*gext/g^2` is 11 % off at pulse 10 τ0, so it fails at 1e-6; at pulse 50 τ0 it is within 2 % and passes at 0.05. `gext/(g0+gext)` matches `efficiency` to 1.9e-13. This is the falsification the user asked for: a formula is either reproduced by the numerics within a stated tolerance or rejected with the Q_ext where it is worst. The ledger costs about 40 ms a point.
3. **Evolution map.** Newton's map `psi − (E − 2)/(2·psi)` with E = psi^2 converged to √2 in a few steps, with derivative 0 at the fixed point (superattracting). The Lyapunov estimate therefore needs a floor, ln 1e-16, so that one log 0 does not dominate the mean. `cos(psi)` converged to 0.7390851332 with |f′| = 0.674. The logistic map `4·psi·(1−psi)` stays bounded but never converges; its Lyapunov estimate is 0.673 ≈ ln 2, the known value, so the run is inconclusive and not a failure. `psi^2 + 1` leaves the bound, so it fails, and `sqrt(psi − 3)` goes non-finite, so it fails. A map with a slow n-dependent drift misses a 1e-9 convergence tolerance by 2e-9 and is reported honestly as inconclusive. 1,000 steps take 0.15 s.
4. **Quantum state / circuit.** Amplitudes `exp(2*pi*i*3*k/N)/sqrt(N)` on 4 qubits (a Fourier basis state) and a Gaussian on 10 qubits both pass. A Gaussian sampled at 100,000 shots had TVD 0.027 against a bound of 0.042, where the bound is the expected TVD 0.5·Σ√(p(1−p)/shots) plus McDiarmid's √(ln 10⁶ / 2·shots), so a correct sampler fails it with probability ≤ 1e-6. `0*k` fails `nonzero`. Angle formulas `theta + t*(j+1)` on a 2-qubit H/RY/CX/RZ circuit swept 16 values of t, kept the norm within 2.2e-16 and sampled within the bound.

All four are deterministic: the same params give byte-identical results, timings excluded, and `sin(7*x)` and `sin(7 * x)` give identical outputs, because the canonical form is what is evaluated. The largest result is 18 KB at the defaults and about 90 KB at the limits.

**Why not eval, ast, numexpr or SymPy.** `eval` runs code, even with a stripped namespace (the dunder-attribute escapes are well known). SymPy's `sympify` calls `eval`. Whitelisting `ast` nodes still hands untrusted text to CPython's parser and is easy to get subtly wrong. numexpr is a native dependency with its own parser. A small grammar can be audited in one sitting, enforces the limits structurally and is easy to mirror in TypeScript for the browser.

**Scope and limits.** Maps are scalar, which covers the Evolution formula's shape (vector state is a later extension). Objective grids are at most 1,024 points, the same scale as the existing Grover limit. Physics formulas are checked against ORB-0's single driven mode. State mode is limited to 10 qubits (1,024 amplitudes), so the result page stays readable.

**Success criteria.** Each mode's AC in SPEC F-28 is met; a malformed formula is rejected at submit with its line and column; W11 passes end to end; there is no `eval(`, `exec(` or `compile(` in the SDK (INV-10).

**References.**
- C. Dürr, P. Høyer, "A quantum algorithm for finding the minimum", arXiv:quant-ph/9607014 (1996).
- C. McDiarmid, "On the method of bounded differences", Surveys in Combinatorics (1989).
- R. M. May, "Simple mathematical models with very complicated dynamics", Nature 261 (1976): the logistic map, λ = ln 2 at r = 4.
- N. Wirth, *Compiler Construction* (1996), ch. 4: recursive-descent parsing.

## Re-Entry R6 — Enhancement: evidence protocol for the evolution engine (2026-11-07)

**Requirement (user).** A six-test table, then "go R6":

| Test | What the simulator should do | Passing evidence |
|---|---|---|
| Correctness | Solve a small graph and compare against exhaustive enumeration | The exact optimum and the engine's gap from it |
| Improvement | Compare the initial configuration with the final adopted configuration | Improvement on independent evaluation, beyond sampling uncertainty |
| Baselines | Compare with random search and a classical optimizer | Results under the same stated resource budget |
| Repeatability | Repeat across 30 independently seeded runs | Distributions and confidence intervals |
| Generalization | Tune on training instances, select on validation, evaluate once on held-out test instances | Test results without further tuning |
| Governance | Submit worse candidates, invalid outputs and missing evidence | Adoption rejected and the last accepted state preserved |

**What exists.** `qai.evolution` decides candidates on one graph with a fixed seed set (V = every check passes, J = mean(1 − ratio), accept iff V ∧ J ≤ J(Ψn) − ε). `exact_maxcut` enumerates up to 16 nodes, `greedy_maxcut` is a one-pass heuristic, and QAOA uses SciPy COBYLA. The evolution record shows that the gate fired, but nothing in it is measured on data the engine did not see.

**Prototype** (`scratchpad/r6/proto.py`, 8-node G(n, 0.5), Ψ₀ = {p 1, maxiter 10}, 4 candidates, splits 2/2/2, 2 decision seeds and 3 fresh seeds per split, 256 shots, 30 runs from SeedSequence(2026).spawn):
- **Time.** 163 s in total, 5.4 s per run, on a host shared with other projects' containers. About 85 % of it is COBYLA's pure-Python iterations inside QAOA, about 100 ms per QAOA run at maxiter 10–20. Simulating the 8-qubit state is cheap (400 energy evaluations at p = 2 take 0.7 s).
- **Adoption and improvement.**
  - Selection on validation adopted a checkpoint in 28 of 30 runs.
  - The paired improvement J(Ψ₀) − J(Ψ*) on the held-out test set with fresh seeds was 0.106, with bootstrap 95 % CI [0.084, 0.128]. That clears sampling uncertainty.
- **Baselines** at an equal budget of 5 configuration evaluations on the train set, test J (mean, sd, 95 % CI):

  | Method | Mean | sd | 95 % CI |
  |---|---|---|---|
  | evolved | 0.155 | 0.044 | [0.141, 0.172] |
  | random search | 0.188 | 0.046 | [0.172, 0.205] |
  | COBYLA on the angles | 0.189 | 0.024 | [0.181, 0.198] |
  | greedy MaxCut (unbudgeted reference) | 0.051 | 0.040 | [0.037, 0.065] |

  The evolved configuration beats both budgeted baselines, but one-pass greedy beats every QAOA method by a wide margin at 8 nodes. The protocol must show this rather than hide it, which is why "baselines" passes on an equal, counted budget and reports the winner instead of requiring the engine to win.
- **At 3 runs with the larger sizes** (splits 3/2/3, 3 seeds, maxiter 20), a run took 19 s. That would be 10 minutes for 30 runs, so the defaults use the smaller sizes and the larger sizes stay available up to the 900 s limit.

**Design notes.**
- **Unit of replication.** The run is the unit for the 30-run CIs: one paired value per run, averaged over its test instances and seeds. Instances and seeds inside a run are not independent of the run's engine choices, so resampling them would understate the uncertainty.
- **Selection by validation.** Selection uses the paired bootstrap over the validation (instance, seed) pairs. With 6 pairs it is coarse, so it is conservative: it adopts nothing unless every resample favours the candidate.
- **Fresh seeds.** Decision, validation and test seeds come from separate SeedSequence children, and the protocol checks that the sets are disjoint.
- **Lock.** The sha256 of the selected config's canonical JSON is written before the test step. The test step is guarded by a counter that raises on a second call, so "evaluate once" holds by construction and does not depend on convention.
- **Correctness controls.** The 8-ring (C* = 8), K₆ (C* = ⌊36/4⌋ = 9) and K₇ (C* = ⌊49/4⌋ = 12) check the enumerator itself. Every sampled and greedy cut must be ≤ C*.
- **Governance.** The injections reuse the real gate on a real checkpoint:
  - worse: Ψ₀ re-scored against an adopted Ψ*;
  - invalid: a NaN probability and counts of shots + 1;
  - missing: one record removed.
  The checkpoint's hash must be unchanged after each one.
- **Bootstrap.** 2,000 percentile resamples, seeded from the protocol seed.

**Success criteria.** At the defaults, all six tests pass, with an improvement CI lower bound > 0. Each injection is rejected. W12 passes end to end. The protocol is deterministic for a given seed.

**References.**
- B. Efron, R. Tibshirani, *An Introduction to the Bootstrap* (1993), ch. 13: percentile intervals.
- E. Farhi, J. Goldstone, S. Gutmann, "A Quantum Approximate Optimization Algorithm", arXiv:1411.4028 (2014).
- M. J. D. Powell, "A direct search optimization method that models the objective and constraint functions by linear interpolation" (1994): COBYLA.
- J. Bergstra, Y. Bengio, "Random search for hyper-parameter optimization", JMLR 13 (2012).
- T. Hastie, R. Tibshirani, J. Friedman, *The Elements of Statistical Learning* (2009), §7.2: train, validation and test splits.
