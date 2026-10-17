# RESEARCH.md — STΛRDANCΞR

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Security & Operations Intelligence"  # pre-build review PBR-2 (was "Productivity")

## Domain Signals

```yaml
domain_signals: ["has_geo", "has_scheduled_work", "has_payments"]
```

## Product Vision
**Problem:** Modern organizations operate increasingly complex digital environments.

Their infrastructure, applications, cloud platforms, security products, identities, networks, and business processes generate enormous quantities of telemetry and events.

However, this information is fragmented across disconnected tools.

Organizations can often identify individual events without understanding the relationships between them.

A configuration change may affect an application. That application may depend on an exposed service. The service may provide access to sensitive data. An identity change may increase the impact of an existing vulnerability.

Each individual system might report only one piece of the situation.

The larger relationship remains hidden.

STARDANCER addresses this problem by connecting observations into an evolving model of the environment.

The central challenge

Organizations can see individual events but struggle to understand the system those events collectively describe.

STARDANCER aims to make those relationships visible, explainable, and actionable.

# Problem Statement

Modern enterprises have become too complex for humans or individual security and monitoring tools to understand as a complete system.

Organizations operate thousands of interconnected assets across on-premises infrastructure, cloud environments, networks, applications, databases, identities, endpoints, security controls, and third-party services. Every change creates new relationships and dependencies, while security events, vulnerabilities, outages, deployments, and configuration changes continuously alter the environment.

Yet most enterprise tools remain fragmented.

Security tools see threats.

Monitoring tools see availability.

Vulnerability platforms see weaknesses.

Identity systems see users.

Cloud platforms see resources.

Application platforms see applications.

SIEMs see events.

CMDBs attempt to catalog assets.

AI systems can analyze individual streams of information.

**Very few systems understand how all of these things interact as one living environment.**

As a result, organizations struggle to answer basic but critical questions:

* What actually changed?
* What systems are affected?
* What depends on the affected system?
* Is an isolated alert part of a larger attack?
* What is the potential business impact?
* Which recent change caused an outage?
* Which vulnerabilities create a realistic attack path?
* What identities or systems are exposed?
* What will happen if a proposed change is made?
* Can an AI agent safely take action?
* What evidence supports the conclusion?

The result is an environment dominated by fragmented telemetry, alert fatigue, manual investigation, incomplete asset relationships, stale documentation, and disconnected operational decisions.

The problem becomes even more significant as organizations introduce autonomous AI agents.

AI agents require context to operate safely. An agent that can execute actions without understanding the systems, dependencies, identities, policies, business impact, and security conditions surrounding those actions creates a new class of operational and security risk.

Organizations therefore need a system that can continuously construct and maintain a trustworthy representation of their digital environment and reason across it.

**STARDANCER addresses this problem by creating a continuously evolving, evidence-backed System Graph and intelligence layer that connects telemetry, entities, relationships, changes, threats, dependencies, policies, and business context.**

Instead of treating every event as an isolated signal, STARDANCER asks:

> **What does this signal mean in the context of the entire system?**

It transforms:

**Signals → Relationships → System State → Intelligence → Risk → Prediction → Validated Action**

The fundamental problem STARDANCER solves is therefore:

> **Organizations have enormous amounts of information about their systems, but lack a continuously validated understanding of the systems themselves.**

STARDANCER provides that missing layer of system intelligence.

**Who it affects:** Primary users

CISOs and security leaders

Understand relationships between exposures, identities, vulnerabilities, and critical assets.
Evaluate how changes affect organizational risk.
Connect security findings to operational context.

CIOs and enterprise architects

Visualize application and infrastructure dependencies.
Understand the impact of architectural changes.
Identify emerging complexity and operational fragility.

Security operations teams

Investigate connected events.
Reconstruct attack paths and sequences of changes.
Understand why a finding matters in the wider environment.

SRE and infrastructure teams

Identify dependencies associated with failures.
Understand configuration drift and service relationships.
Investigate emerging instability.

AI and automation engineers

Connect intelligent agents to a governed system model.
Evaluate proposed changes against observed state.
Develop evidence-backed automation workflows.
Secondary users
IT operations leaders
Cloud architects
GRC and compliance teams
Enterprise risk teams
Platform engineering teams
Managed security service providers

**Why existing solutions fall short:** Traditional monitoring, observability, configuration management, SIEM, and security platforms solve important individual problems.

However, they commonly organize their information around their own products, data models, and operational purposes.

This creates several limitations:

Fragmented visibility: Information remains distributed across different tools.
Limited relationship awareness: Dependencies and indirect consequences can be difficult to identify.
Point-in-time understanding: Dashboards often emphasize current metrics rather than how the environment evolved.
Reactive investigation: Analysts must manually reconstruct sequences of events.
Disconnected risk signals: Individual findings may not reveal their combined significance.
Limited predictive context: Existing state and historical patterns do not automatically explain what might happen next.
Difficult root-cause analysis: Teams may know what failed without understanding the chain of changes that caused it.

STARDANCER should complement existing systems rather than require an organization to replace them.

Its differentiation is the system-level model that connects their observations.

# Why Existing Solutions Fall Short

Enterprise technology has become highly specialized. Organizations have powerful tools for nearly every individual domain, yet those tools generally operate within their own boundaries.

The problem is not that existing platforms are incapable. The problem is that **the enterprise itself does not operate in those boundaries.**

A security incident can begin with an identity, move through an endpoint, reach an application, access a database, traverse a network, trigger a cloud event, and ultimately affect a business process. No single traditional platform necessarily maintains the complete chain.

## 1. Tools Understand Domains, Not the Whole System

SIEMs understand security events.

EDR platforms understand endpoints.

Vulnerability platforms understand vulnerabilities.

APM platforms understand applications.

Network monitoring understands network behavior.

Cloud platforms understand cloud resources.

CMDBs catalog infrastructure.

Identity platforms understand authentication and authorization.

Each provides valuable information, but the information remains fragmented.

**STARDANCER connects the domains.**

---

## 2. Alerts Are Not Understanding

Most platforms are optimized to detect events:

> Something happened.

But organizations need to understand:

> **What does this event mean?**

An alert by itself does not necessarily reveal:

* What the asset supports
* What depends on it
* Which identities are involved
* What changed recently
* Whether the behavior is normal
* What other events are related
* What business processes are affected
* What could happen next

STARDANCER moves from **alert detection to system understanding**.

---

## 3. Asset Inventories Become Stale

Traditional inventories and CMDBs often depend heavily on periodic discovery, manual updates, integrations, or declared configuration.

Modern infrastructure changes continuously.

Cloud resources appear and disappear.

Applications are deployed continuously.

Containers are ephemeral.

Identities change.

Network paths change.

Configurations drift.

AI agents create and modify resources.

A static inventory cannot adequately represent a dynamic environment.

STARDANCER treats system state as continuously changing.

---

## 4. Relationships Are More Important Than Assets

Knowing that an organization owns 10,000 servers is useful.

Knowing that:

**Server A → Application B → Database C → Business Process D**

is far more useful.

The risk of an asset depends heavily on its relationships.

A vulnerability on an isolated development machine is different from the same vulnerability on a server supporting a critical payment system.

STARDANCER therefore models **relationships and dependencies**, not just inventories.

---

## 5. Existing AI Often Lacks System Context

Generative AI can summarize logs, investigate alerts, write queries, and recommend remediation.

But an AI model does not inherently know the customer's complete environment.

Without reliable system context, an AI can:

* Miss dependencies
* Misinterpret events
* Recommend inappropriate actions
* Lack awareness of business impact
* Operate from stale information
* Hallucinate relationships
* Take actions outside intended scope

STARDANCER gives AI a governed representation of the environment in which it operates.

---

## 6. AI Recommendations Are Not Governance

An AI model can recommend:

> "Disable this account."

But the important questions are:

* Is the account authorized to be disabled?
* What systems depend on it?
* Is it a service account?
* What business process could fail?
* What policy applies?
* Is human approval required?
* What evidence supports the recommendation?

Existing AI systems frequently blur the boundary between **intelligence and authority**.

STARDANCER separates them.

**AI proposes. Governance validates.**

---

## 7. Security, Operations, and Business Context Are Disconnected

Security teams may know an asset is vulnerable.

Operations may know it is critical.

Application teams may know what it supports.

GRC may know which control applies.

Executives may know the business process it affects.

The information exists, but it is distributed across teams and systems.

STARDANCER brings these dimensions together into a common system model.

---

## 8. Historical Context Is Often Lost

A current snapshot can tell you what exists.

It cannot necessarily explain:

> **Why is it like this?**

Without historical system state, organizations struggle with:

* Root-cause analysis
* Incident reconstruction
* Change attribution
* Configuration drift
* Attack-path reconstruction
* Deployment impact
* Recurring failures

STARDANCER treats time as a first-class dimension of system intelligence.

It can reason across:

**System(t₀) → System(t₁) → System(t₂) → System(tₙ)**

---

## 9. Tools Generate Data Faster Than Humans Can Correlate It

The enterprise has entered an era of telemetry abundance.

More endpoints.

More cloud resources.

More logs.

More APIs.

More identities.

More security products.

More monitoring.

More AI-generated events.

Adding another dashboard does not solve this problem.

Organizations need systems that **correlate information before presenting it to humans**.

STARDANCER is designed around that principle.

---

## 10. Existing Platforms Are Often Reactive

Traditional workflows frequently look like:

**Alert → Analyst → Investigation → Decision → Ticket → Remediation**

STARDANCER moves toward:

**Observe → Understand → Predict → Validate → Act → Verify**

The goal is to identify relationships and potential consequences before they become incidents.

---

## 11. Existing Platforms Were Not Designed for AI-Native Environments

The emergence of autonomous AI agents creates a new requirement.

Organizations will increasingly have software capable of:

* Reading data
* Modifying infrastructure
* Deploying applications
* Rotating credentials
* Changing configurations
* Responding to incidents
* Creating resources
* Calling APIs

These agents need a trustworthy understanding of the environment and clearly defined boundaries.

STARDANCER provides the context and governance layer required for this model.

---

## 12. Cloud, On-Prem, and Hybrid Environments Complicate Everything

Modern enterprises rarely operate in a single environment.

They combine:

* Data centers
* Public cloud
* SaaS
* Remote endpoints
* Branch networks
* Containers
* APIs
* Third-party services
* Legacy systems

The resulting environment is constantly changing and difficult to model with a single traditional source of truth.

STARDANCER is designed to build a unified representation across those environments.

---

## 13. Data Is Frequently Sent Away From the Environment It Protects

Many modern AI and analytics architectures require organizations to send sensitive telemetry to external services.

For security-sensitive organizations, that creates additional concerns around:

* Data sovereignty
* Privacy
* Regulatory requirements
* Intellectual property
* Security telemetry
* Credentials and secrets
* Customer information

STARDANCER is designed to support sovereign and on-premises deployment, including local processing and local AI.

---

## 14. Most Platforms Do Not Create a Closed Feedback Loop

Traditional platforms often stop at:

**Detect → Report**

More advanced platforms reach:

**Detect → Recommend**

STARDANCER is designed for:

**Observe → Model → Reason → Predict → Validate → Act → Verify → Learn**

The system continuously compares the resulting environment against its previous understanding.

That creates a living intelligence loop rather than a static reporting platform.

---

# The Fundamental Gap

Existing enterprise platforms provide powerful pieces of the puzzle.

What is missing is the layer that understands how the pieces fit together.

The gap is:

**Data → Understanding**

**Assets → Relationships**

**Events → Context**

**Alerts → System State**

**AI → Governed Intelligence**

**Recommendations → Validated Action**

**Snapshots → Continuous State**

STARDANCER is being built to occupy that layer.

> **Existing tools tell you what they can see. STARDANCER is designed to understand how what they see fits together.**

**Solution:** STARDANCER™ — Claude Design & Site Context

Product: STARDANCER
Tagline: Intelligence in Motion.
Category: AI Systems Intelligence Platform
Core intelligence model: Mirror Brain Ψ
Design direction: Coral-accented cosmic interface
Development framework: MDLC.ai

1. Project Identity

Product definition

STARDANCER is a systems-intelligence platform designed to model an organization's digital environment as an evidence-backed, evolving system graph.

It connects information about infrastructure, applications, identities, security events, dependencies, configurations, and operational changes to help users understand relationships across their environment.

Its Mirror Brain Ψ analyzes observed system state, identifies meaningful changes, develops hypotheses, and proposes potential future states for validation.

STARDANCER is intended to complement existing operational platforms, not require organizations to replace them.

Core positioning: Don't just monitor systems. Understand what they are becoming.

Product philosophy

Every enterprise is a constellation.

The System Graph makes relationships visible.

Mirror Brain Ψ reasons over observed state and historical evidence.

Predictions are hypotheses, not facts.

AI proposes. Governance validates. Evidence establishes what is known.

2. Problem Statement

Organizations operate complex digital environments across infrastructure, applications, cloud platforms, security products, identities, networks, and business processes. These systems produce large volumes of telemetry and events, often stored in disconnected tools.

Teams can identify individual events but may struggle to understand how they relate. A configuration change can affect an application, which depends on an exposed service, which provides access to sensitive data. An identity change may increase the impact of an existing vulnerability.

The larger relationship is often difficult to reconstruct manually.

STARDANCER aims to connect observations into an evolving model of the environment so that users can investigate changes, understand dependencies, and evaluate potential consequences.

Why existing solutions can fall short

Information is fragmented across tools.

Dependencies and indirect consequences are difficult to identify.

Point-in-time dashboards may not explain how the environment evolved.

Investigations require manual event correlation.

Individual risk findings may hide combined significance.

Current-state visibility does not automatically explain what may happen next.

STARDANCER should integrate with and complement existing platforms.

3. Users and Use Cases

Primary users

CISOs and security leaders

Understand relationships between exposures, identities, vulnerabilities, and critical assets.

Evaluate how changes may affect organizational risk.

Connect findings to operational context.

CIOs and enterprise architects

Visualize application and infrastructure dependencies.

Assess the potential impact of architectural changes.

Identify complexity and operational fragility.

Security operations teams

Investigate connected events and changes.

Reconstruct event sequences and possible attack paths.

Understand why an individual finding matters in the wider environment.

SRE and infrastructure teams

Investigate dependency failures.

Understand configuration drift and service relationships.

Identify emerging instability.

AI and automation engineers

Connect agents to a governed system model.

Evaluate proposed changes against observed state.

Develop evidence-backed automation workflows.

Secondary users

IT operations leaders, cloud architects, GRC and compliance teams, enterprise risk teams, platform engineering teams, and managed security service providers.

4. Product Architecture

Use six conceptual layers. These are responsibilities, not necessarily six separate deployable services.

Observe: Collect telemetry, events, configurations, and other source data.

Connect: Normalize observations and construct an evidence-backed System Graph.

Understand: Model dependencies, relationships, and changes over time.

Mirror Brain Ψ: Reason over observed and historical system state.

Predict: Generate hypotheses and evaluate candidate future states.

Govern: Validate evidence, policy, risk, and authorization before action.

Observe

Potential sources include cloud infrastructure, network devices and firewalls, identity providers, endpoint security, SIEM platforms, application monitoring, configuration management, vulnerability management, asset inventories, and change-management systems.

Treat these as potential integrations until they are implemented and tested.

Connect

Represent the environment as a temporal graph.

Possible nodes

Assets

Applications

Services

Identities

Networks

Vulnerabilities

Events

Configurations

Business processes

Possible relationships

Depends on

Connects to

Authenticates to

Runs on

Exposes

Controls

Changed by

Affects

Important relationships should include provenance, observation time, and confidence or inference status.

Understand

Compare current observations with previous state to identify:

New or removed entities

Changed relationships

Configuration drift

Emerging dependencies

Anomalous changes

Increasing exposure

Potential cascading effects

Distinguish directly observed facts from inferred relationships.

Mirror Brain Ψ

Maintain an evidence-backed model of the environment. Analyze changes, construct candidate interpretations, identify missing evidence, and preserve a history of accepted states.

Predict

Use available evidence, historical observations, dependencies, and explicitly defined models to generate candidate future states.

Examples:

Potential impact of a configuration change

Possible consequences of an exposed service

Emerging dependency failure

Risk changes after an identity modification

Operational consequences of infrastructure drift

Predictions must include assumptions, evidence, and uncertainty. Do not present a hypothesis as an observed fact.

Govern

Enforce authorization, policy, evidence requirements, risk classification, validation gates, approval requirements, and audit records independently of AI reasoning.

AI proposes. Governance validates.

5. Mirror Brain Ψ: Conceptual Model

State transition

[
\Psi_n \rightarrow \psi(\Psi_n) \rightarrow S_n
\rightarrow \operatorname{Validate}(S_n)
\rightarrow \Psi_{n+1}
]

Where:

(\Psi_n): current Mirror Brain state

(\psi(\Psi_n)): analysis or transformation of the current state

(S_n): candidate system model

(\operatorname{Validate}): evidence, policy, and consistency checks

(\Psi_{n+1}): next accepted model state

Candidate improvement model

[
S = \lambda_\psi \cdot \psi(\Psi)
]

This is a conceptual design equation for creating a candidate state from the current model and an improvement transformation. It is not, by itself, a complete mathematical specification or an empirically established physical law. Define the actual transformation, inputs, validation criteria, and acceptance conditions in implementation.

Temporal relationship model

[
\Psi(t)=
\frac{\partial}{\partial t}
\sum_{i,j}
\left[S_i(t)\cap S_j(t)\right]
\rightarrow R(t)
]

This is a conceptual expression for analyzing changing relationships and deriving a risk or response signal. Implementation must define what the intersection represents, how relationships are weighted, and how the signal is calculated.

Governing loop

OBSERVE
   ↓
ESTABLISH EVIDENCE
   ↓
UPDATE SYSTEM GRAPH
   ↓
ANALYZE STATE Ψ
   ↓
IDENTIFY CHANGE
   ↓
PROPOSE HYPOTHESIS
   ↓
VALIDATE EVIDENCE + POLICY
   ↓
ACCEPT / REJECT / REQUEST REVIEW
   ↓
UPDATE MIRROR Ψ
   ↺

The Mirror Brain may propose changes to its models, but it must not rewrite governance rules or bypass validation.

6. Visual Identity and Design Language

Creative direction

A living digital universe represented as an interconnected constellation.

The website should combine the STARDANCER cosmic artwork with the clarity and usability of a modern systems-intelligence product. The interface should feel like an intelligent universe, not a conventional dashboard template.

The System Graph is the main product visualization. A feminine synthetic AI entity may support the brand identity or represent Mirror Brain Ψ, but it should not dominate operational screens.

Color palette

Token

Value

Use

Primary coral

#FF806F

Primary actions, active states, selection

Light coral

#FFB0A4

Highlights

Deep background

#070A12

Main canvas

Elevated background

#101626

Panels and secondary surfaces

Cyan

#62D9FF

Relationships and telemetry

Violet

#9D8CFF

AI and Mirror Brain visualizations

Magenta

#E88DDB

Secondary relationship category

Primary text

#F1F4FC

Main text

Secondary text

#9AA8C5

Supporting information

Success

#7BE0B5

Healthy or validated state

Warning

#FFC777

Attention required

Critical

#FF647C

Critical risk

Coral defines interface controls and the brand accent. Cyan, violet, and magenta distinguish categories of graph information. Avoid using every color everywhere or applying glow effects to all components.

Typography

Use a clean, highly legible sans-serif for interface text.

Use a restrained monospaced typeface for equations, identifiers, event records, and technical metadata.

Make the STARDANCER wordmark futuristic but readable.

Maintain clear hierarchy, spacing, and contrast.

Visual behavior

Use subtle, purposeful effects:

Slowly moving constellation points

Restrained orbital paths

Selected graph nodes that illuminate

Relationship paths that highlight on selection

State transitions over time

Quantum-inspired waveforms

Optional synthetic AI centerpiece

Animations must be performant and respect prefers-reduced-motion.

Quantum visuals are metaphors for state space and possibility. Do not imply quantum-computing capability unless it exists.

7. Website Structure

Page 1: Landing page

Hero wordmark: STARDANCER
Tagline: Intelligence in Motion.

Headline: Understand what your systems are becoming.

Supporting copy:

STARDANCER connects infrastructure, applications, identities, and security signals into a living model of your digital environment. Discover hidden relationships, understand what changes, and evaluate what may happen next.

Primary CTA: Explore the System Graph
Secondary CTA: Discover Mirror Brain Ψ

Recommended sections:

Every enterprise is a constellation.

From signals to understanding.

Meet Mirror Brain Ψ.

Intelligence across your environment.

Intelligence with governance.

Closing CTA: “Your systems are changing. Understand the direction.”

Potential domains: Infrastructure, Applications, Identity, Security, Data, Operations, and Automation.

Do not invent customer testimonials, deployments, integrations, or performance claims.

Page 2: Interactive Intelligence Dashboard

Suggested navigation:

Overview

System Graph

Mirror Brain Ψ

Changes

Predictions

Evidence

Agents

Governance

Settings

Implement only pages with a defined purpose. Unfinished features must not falsely appear operational.

Dashboard header

Show:

STARDANCER identity

Selected environment

Model state

Last observation timestamp

Evidence freshness

Current risk summary

Distinguish simulated data from live connected data.

System Graph

The graph should be the largest and most important visualization.

Users should be able to:

Select a node.

Inspect its attributes.

Explore incoming and outgoing relationships.

Review changes over time.

Inspect the evidence behind a relationship.

Explore potential impact.

Navigate to related entities.

Use shapes or restrained markers to distinguish entity types. Use edge styles to distinguish observed relationships from inferred ones.

Entity detail panel

What

Entity type, name, identifier, current state

Why

Why the entity matters and why it was surfaced

Relationships

Connected entities, direction, provenance

Changes

Previous state, current state, observation timestamp

Impact

Potentially affected components, dependencies, risk implications

Prediction

Candidate future states, confidence, assumptions

Evidence

Source, observation timestamp, source identifier, validation status

Never represent unimplemented AI-generated explanations as real findings.

Dashboard metrics

Use metrics that can eventually be calculated from actual data:

Entities observed

Relationships mapped

Changes detected

Unresolved anomalies

Evidence freshness

Predictions awaiting review

Prototype metrics must be clearly labeled as demonstration data.

8. AI Agents

The initial interface should represent four operational agents.

APEX — Strategy and Architecture

Establish architectural context.

Analyze system structure.

Identify dependencies.

Propose system-model improvements.

LUMEN — Research and Intelligence

Investigate observations.

Correlate evidence.

Develop hypotheses.

Explain relationships.

OPERON — Execution and Build

Translate approved requirements into execution plans.

Coordinate permitted operations.

Track execution results.

Return evidence from completed operations.

VECTOR — Security and Compliance

Evaluate security implications.

Identify policy conflicts.

Review evidence requirements.

Recommend validation and remediation steps.

An optional fifth role, SENTINEL, may coordinate governance if it has distinct responsibilities. Do not create a separate agent merely to duplicate deterministic policy enforcement.

Agents are reasoning or execution roles. They do not replace evidence models, authorization, or validation logic.

9. Key User Workflows

Workflow A: Understand the environment

Open the dashboard.

Inspect the observed System Graph.

Select an entity.

Explore its dependencies.

Review related assets and events.

Inspect supporting evidence.

Success: Users can understand an entity and its relationships without manually correlating multiple tools.

Workflow B: Investigate a change

Receive a new observation.

Compare it with previous state.

Identify changed entities and relationships.

Evaluate potential consequences.

Inspect supporting evidence.

Record a traceable investigation result.

Success: Users can move from a detected change to its evidence and potential impact.

Workflow C: Evaluate a prediction

Analyze the current model.

Generate a candidate future state.

Record assumptions.

Evaluate available evidence.

Communicate confidence and uncertainty.

Present the hypothesis for review.

Success: Predictions remain explainable and distinguishable from observed facts.

Workflow D: Propose an improvement

An agent identifies an opportunity.

Record the proposed change.

Evaluate it against policies.

Run required validation.

Obtain human approval when required.

Execute only an authorized operation.

Record the outcome.

Success: Material changes have a traceable proposal, validation result, and authorization history.

10. Governance and Trust

Governance is a foundational product property, not a decorative dashboard section.

Distinguish observed facts from AI-generated hypotheses.

Associate material conclusions with supporting evidence.

Mark inferred relationships explicitly.

Expose prediction assumptions and uncertainty.

Enforce authorization independently of AI reasoning.

Prevent agents from bypassing mandatory gates.

Require appropriate approval for high-risk operations.

Do not silently adopt rejected proposals.

Record material changes in an auditable history.

Never treat missing evidence as proof of safety.

Principle: AI proposes. Governance validates. Evidence establishes what is known.

11. Technical Requirements

Claude Code must inspect the existing repository before selecting or changing the stack. Do not assume a new repository or framework is required.

Frontend requirements

Responsive desktop and mobile layouts

Interactive graph visualization

Entity selection and drill-down

State and relationship filters

Historical change inspection

Accessible controls and keyboard navigation

Coral-accented design tokens

Reduced-motion support

Loading, empty, and error states

Backend capabilities to plan for

Source ingestion

Data normalization

Entity identity resolution

Relationship management

Temporal state storage

Evidence provenance

Graph queries

Change detection

Prediction workflows

Governance and audit records

Authentication and authorization

Select the database, graph technology, and infrastructure after inspecting the repository and defining scale requirements. Avoid adding infrastructure without a demonstrated need.

Data model

Each observed entity should have:

Stable identifier

Entity type

Attributes

Source provenance

Observation timestamps

Each relationship should have:

Source and target entities

Relationship type

Provenance

Observation timestamp

Confidence or inference status where appropriate

Changes must preserve enough history to reconstruct previous states. Store predictions separately from observations.

Security requirements

Server-side authorization

Tenant isolation where applicable

Secure secrets management

Input validation

Audit logging

Least-privilege access

Secure handling of source credentials

Explicit approval before consequential operations

Never expose secrets or privileged credentials in frontend code.

12. Success Metrics

Area

Success criterion

Comprehension

A new user understands the product's purpose within 30 seconds

Visualization

Users can select an entity and inspect relationships

Explainability

Material findings expose supporting evidence

Temporal awareness

Users can distinguish current from historical state

Prediction

Hypotheses expose assumptions and uncertainty

Governance

Proposed actions cannot bypass required validation

Usability

Primary workflows work on desktop and mobile

Performance

Graph rendering stays responsive at the declared test scale

Accessibility

Keyboard navigation, focus, contrast, and reduced motion are supported

Reliability

Errors and missing data are surfaced explicitly

These are target criteria, not claims of achieved performance. Measure them before reporting success.

13. Non-Goals

The initial release is not intended to:

Replace every monitoring or security platform.

Replace an organization's SIEM.

Automatically remediate every finding.

Execute unapproved infrastructure changes.

Treat AI-generated hypotheses as facts.

Claim perfect prediction accuracy.

Require quantum-computing hardware.

Depend on a simulated AI entity to deliver real intelligence.

Display fabricated telemetry as live.

Introduce autonomous self-modification without validation.

Build every possible integration before the core System Graph works.

The first milestone is a credible interactive representation of system state, relationships, changes, and evidence.

14. Implementation Roadmap

Milestone 1: Brand and interface

Establish design tokens.

Implement the coral-accented dark interface.

Create main navigation.

Build the responsive landing page.

Build the dashboard shell.

Reuse supplied STARDANCER artwork where appropriate.

Milestone 2: Interactive System Graph

Create a defined demonstration dataset.

Render entity nodes and relationships.

Implement node selection.

Build the entity detail panel.

Add entity-type filters.

Clearly label demonstration data.

Milestone 3: Temporal intelligence

Represent previous and current entity states.

Show detected changes.

Add historical inspection.

Associate changes with supporting evidence.

Milestone 4: Mirror Brain Ψ

Define a concrete state representation.

Build observation and analysis workflows.

Record candidate interpretations.

Add validation status and model history.

Distinguish accepted and rejected states.

Milestone 5: Agent workflows

Implement APEX, LUMEN, OPERON, and VECTOR interfaces.

Define inputs and outputs for each role.

Add proposal records and execution status.

Keep external model calls disabled until configured.

Milestone 6: Governance

Add policy checks.

Implement authorization requirements.

Record validation outcomes.

Add human approval workflows where required.

Test attempts to bypass mandatory gates.

Milestone 7: Real integrations

Define source adapters.

Connect an initial telemetry source.

Normalize incoming observations.

Associate observations with graph entities.

Verify preservation of evidence and timestamps.

15. Claude Code Build Directive

Use this directive after loading the project context.

PROJECT: STARDANCER

MISSION
Build STARDANCER as a systems-intelligence platform that models
an organization's digital environment as an evidence-backed,
evolving System Graph.

POSITIONING
STARDANCER — Intelligence in Motion.

Primary headline:
Understand what your systems are becoming.

CORE CONCEPT
Every enterprise is a constellation.

Connect infrastructure, applications, identities, security signals,
configurations, and operational events into a unified temporal model.

Mirror Brain Ψ reasons over observed system state, evaluates changes,
and proposes candidate future states.

AI proposes.
Governance validates.
Evidence establishes what is known.

DESIGN
Use a premium dark-space interface with coral as the primary
interaction accent.

Primary coral: #FF806F
Background: #070A12
Elevated surface: #101626
Cyan: #62D9FF
Violet: #9D8CFF
Primary text: #F1F4FC

Use restrained cosmic visuals, interconnected constellation nodes,
subtle orbital geometry, and quantum-inspired elements.

The System Graph is the primary visualization.

A feminine synthetic AI motif may support the brand identity,
but must not replace the System Graph as the main product interface.

Avoid excessive neon, decorative clutter, generic dashboard layouts,
and unnecessary nested cards.

PRODUCT PRINCIPLES
1. Evidence over assertion.
2. Observed facts are separate from hypotheses.
3. Predictions expose assumptions and uncertainty.
4. System relationships preserve provenance.
5. Historical state is reconstructable.
6. Authorization is enforced independently of AI reasoning.
7. Mandatory gates cannot be bypassed.
8. High-risk operations require appropriate approval.
9. Proposed actions are not represented as completed actions.
10. Demonstration data is never misrepresented as live data.

INITIAL EXPERIENCE
Create a landing page and interactive dashboard.

The dashboard should include:
Overview, System Graph, Mirror Brain Ψ, Changes, Predictions,
Evidence, Agents, and Governance.

Implement only functionality that can be demonstrated honestly.

CORE WORKFLOWS
1. Inspect an entity and its relationships.
2. Investigate a system change.
3. Review supporting evidence.
4. Explore a candidate future state.
5. Inspect an agent's proposed improvement.
6. Validate a proposal before adoption.

AGENTS
APEX: Strategy and architecture.
LUMEN: Research and intelligence.
OPERON: Execution and build.
VECTOR: Security and compliance.

Keep governance deterministic and independent of agent recommendations.

BUILD PROCESS
First inspect the existing repository, framework, dependencies,
architecture, design assets, and current implementation.

Do not overwrite working functionality or invent a new architecture
before understanding the repository.

Report the existing architecture, relevant files, current capabilities,
gaps, and proposed implementation sequence.

Implement incrementally, beginning with the smallest complete
end-to-end workflow.

Run available tests, linting, type checks, and builds.

Report actual test results. Do not claim success without evidence.

Preserve existing security controls and test authorization boundaries.

Do not fabricate integrations, telemetry, customers, or performance
claims.

Do not activate external AI calls or consequential operations without
the necessary configuration and authorization.

DEFINITION OF DONE
A user can open the dashboard, inspect a System Graph, select an entity,
understand its relationships, examine a change, review its evidence,
and inspect a candidate hypothesis.

Demonstration data is clearly labeled.
The interface is responsive and accessible.
Validation and authorization boundaries are explicit.
Available relevant tests pass.

BEGIN by inspecting the repository. Then implement the first complete
milestone and report what changed, what was tested, and what remains.

16. Product Vision

The initial product should focus on three capabilities:

SEE — Visualize the organization's digital environment as an interactive constellation.

UNDERSTAND — Use Mirror Brain Ψ to explain relationships, changes, and their potential significance through traceable evidence.

ANTICIPATE — Develop candidate future states and identify which hypotheses need further validation.

STARDANCER can ultimately serve as a systems-intelligence layer above products such as MDLC.ai, ALPHA ZER0, Threat Watch, CyberOps, and Synapse/Pulse, while each retains its own operational responsibilities.

The System Graph becomes the shared model. Mirror Brain Ψ becomes the reasoning layer. Governance determines what can actually be accepted or executed.

STARDANCER — Understand what your systems are becoming.

# STARDANCER

## What We're Building

STARDANCER is an AI systems-intelligence platform for understanding, protecting, and governing complex digital environments.

Modern enterprises generate enormous amounts of telemetry from servers, endpoints, networks, cloud platforms, applications, identities, security systems, databases, infrastructure, and business processes. The problem is not a lack of data. The problem is that the data exists in disconnected systems that rarely understand how the pieces relate to one another.

STARDANCER creates that missing layer of intelligence.

It continuously observes the environment, builds a living System Graph, understands relationships between systems and entities, detects changes and anomalies, evaluates risk, reasons over evidence, predicts potential consequences, and produces validated recommendations or actions.

The objective is simple:

**Understand the system behind the signals.**

---

## The Core Idea

STARDANCER treats an organization's digital environment as a living system.

Servers are connected to applications.

Applications depend on databases.

Users authenticate to identities.

Identities access applications.

Applications communicate across networks.

Security controls protect those relationships.

Deployments change them.

Threats attempt to exploit them.

Business processes depend on them.

STARDANCER continuously models those relationships.

Instead of looking at isolated alerts, it asks:

**What changed?**

**What does it affect?**

**What does it depend on?**

**Why did it happen?**

**What could happen next?**

**What should be done?**

---

## The System Graph

At the center of STARDANCER is a continuously evolving System Graph.

The graph represents:

* Users
* Identities
* Devices
* Servers
* Applications
* APIs
* Databases
* Containers
* Cloud resources
* Networks
* Firewalls
* Data
* Secrets
* Certificates
* Cryptographic keys
* Policies
* Controls
* Vulnerabilities
* Threats
* Incidents
* Deployments
* Business processes
* AI agents

Relationships between those entities are modeled as well.

The graph is temporal.

STARDANCER does not only know what the environment looks like now. It maintains an understanding of how the environment changed over time.

This enables historical reconstruction, change detection, drift detection, dependency analysis, attack-path analysis, incident reconstruction, and predictive intelligence.

---

## The Mirror Brain

STARDANCER introduces the concept of the **Mirror Brain Ψ**.

Ψ is the intelligence layer that reasons over the observed system rather than operating from an isolated prompt.

The architecture is:

**Reality → Observation → System Graph → Ψ → Understanding → Prediction → Recommendation → Validation**

The AI does not become the source of truth.

Evidence is the source of truth.

The AI interprets evidence and proposes conclusions. Deterministic policy, governance, authorization, and validation determine what can actually happen.

This creates an important architectural boundary:

**AI proposes. Governance validates.**

---

## The Continuous Intelligence Loop

STARDANCER operates as a continuous feedback system:

**Observe → Model → Reason → Validate → Act → Observe Again**

Conceptually:

**Ψₙ → ψ(Ψₙ) → Sₙ → λψ → Validate(Sₙ) → Ψₙ₊₁**

The system continuously incorporates new evidence, changes its understanding of the environment, validates that understanding, and produces the next trusted system state.

This allows STARDANCER to evolve with the environment without becoming an uncontrolled self-modifying AI.

---

## Evidence-First Intelligence

Every significant conclusion should be traceable to evidence.

STARDANCER therefore maintains an evidence chain connecting:

**Observation → Source → Entity → Relationship → Change → Risk → Reasoning → Recommendation → Validation → Action → Outcome**

This allows security teams, operators, auditors, and executives to understand not only what STARDANCER believes, but why it believes it.

The system should never treat an LLM response as authoritative simply because an AI produced it.

---

## Integration With the MDLC Ecosystem

STARDANCER is designed to operate as the intelligence layer of the broader MDLC architecture.

### MDLC

MDLC governs how software is researched, designed, built, reviewed, secured, and released.

**MDLC governs the build.**

### STARDANCER

STARDANCER observes and understands the resulting living system.

**STARDANCER governs understanding.**

### ALPHA ZER0

ALPHA ZER0 provides cryptographic security, data protection, identity trust, and sovereign security controls.

**ALPHA ZER0 governs trust.**

### Threat Watch

Threat Watch ingests and analyzes enterprise security telemetry.

**Threat Watch governs threat intelligence.**

### CyberOps

CyberOps provides external attack-surface and exposure intelligence.

**CyberOps governs external visibility.**

### Synapse / Pulse

Synapse/Pulse provides infrastructure, availability, network, DNS, API, TLS, and operational telemetry.

**Synapse/Pulse governs operational visibility.**

Together they form a larger architecture rather than a collection of disconnected products.

---

## From Alerts to System Intelligence

Traditional security and monitoring systems often follow:

**Event → Alert**

STARDANCER extends that model:

**Event → Entity → Relationship → System State → Change → Context → Risk → Prediction → Recommendation → Validation → Action**

This is the fundamental difference.

STARDANCER is not intended to be another SIEM, CMDB, EDR, vulnerability scanner, or monitoring platform.

It is the intelligence layer that connects those systems.

---

## Autonomous AI With Governance

As organizations deploy increasingly capable AI agents, those agents need reliable context about the environments in which they operate.

STARDANCER can provide that context.

Before an AI agent takes an action, STARDANCER can establish:

* Who is requesting the action
* What system is affected
* What the system depends on
* What data is involved
* What policies apply
* What the business impact could be
* What the risk is
* What actions are authorized
* Whether human approval is required
* Whether the resulting state can be validated

This creates a governed architecture for autonomous systems.

The AI does not receive unrestricted authority.

It operates within a continuously observed and policy-controlled environment.

---

## Self-Improvement

STARDANCER can eventually support a governed Self-Improvement Agent.

The process is:

**Observe → Analyze → Propose Ψ' → Test → Kernel Gate → Accept or Reject**

Self-improvement is therefore constrained by the same principles used throughout MDLC:

* Mandatory validation
* Evidence
* Deterministic policy
* Risk classification
* Auditability
* Human approval for high-risk changes
* No governance bypass
* No uncontrolled recursive modification

The system can improve its understanding without being permitted to redefine the rules that govern it.

---

## Sovereign Deployment

STARDANCER is designed for enterprise and potentially fully on-premises deployment.

Sensitive organizations should be able to operate the platform inside their own environment with:

* Local telemetry
* Local System Graph
* Local AI models
* Local policy
* Local cryptographic keys
* Local evidence
* Local audit records
* Local processing

Cloud services can be optional rather than fundamental.

This is especially important for organizations that cannot or do not want to send security telemetry and operational intelligence to an external AI provider.

---

## Resilience

STARDANCER itself must not become a single point of failure.

The architecture therefore separates:

* Collection
* Message transport
* Processing
* Graph storage
* State storage
* AI workers
* Policy enforcement
* Cryptographic services
* Evidence storage

Failure of an AI worker should not destroy system state.

Failure of a collector should not destroy historical evidence.

Failure of a network path should not compromise governance.

The intelligence layer must be resilient enough to operate in the environments it is designed to protect.

---

## What STARDANCER Ultimately Becomes

STARDANCER is building toward a **living digital model of an enterprise**.

Not simply a dashboard.

Not simply an AI chatbot.

Not simply a SIEM.

Not simply a monitoring platform.

Not simply a graph database.

It is a continuously evolving intelligence system that understands:

**what exists, how it is connected, what changed, why it changed, what it affects, what threatens it, what depends on it, what could happen next, and what actions are safe to take.**

The ultimate architecture is:

**Observe → Model → Understand → Predict → Validate → Act → Learn**

with evidence, governance, cryptography, and human control underneath every consequential transition.

---

## The STARDANCER Vision

**Every enterprise is a constellation.**

STARDANCER makes the constellation visible.

It turns disconnected telemetry into relationships.

Relationships into system state.

System state into intelligence.

Intelligence into validated decisions.

And validated decisions into controlled action.

**STARDANCER — intelligence for systems in motion.**

STARDANCER — Comprehensive Build Context

STARDANCER should now be treated as the intelligence and systems-understanding layer above the entire MDLC ecosystem, rather than simply another application.

The architecture emerging from the builds is:

MDLC governs how systems are built.
STARDANCER understands how systems behave.
ALPHA ZER0 protects the systems and data.
Threat Watch detects threats.
CyberOps observes external attack surface and security posture.
Synapse/Pulse observes availability, infrastructure and operational health.

STARDANCER becomes the layer that can reason across all of them.

1. What We're Building

STARDANCER is an AI systems-intelligence platform that creates a continuously evolving model of an organization's digital environment.

It ingests signals from:

infrastructure
servers
endpoints
networks
firewalls
cloud environments
applications
databases
identities
security systems
logs
telemetry
configurations
vulnerabilities
incidents
deployments
business processes
cryptographic systems
availability monitors
human decisions
AI agents

It transforms those signals into a continuously updated System Graph.

The goal is not merely:

"What happened?"

It is:

"What is the state of the system, why is it in that state, what is changing, what depends on it, what could happen next, and what should be done?"

2. The STARDANCER Concept

The name fits because the system is intended to represent a constantly changing digital environment as a living constellation.

Every entity becomes a node.

Every relationship becomes an edge.

Every event changes the state.

Every change potentially changes the risk.

Conceptually:

                USERS
                  │
                  ▼
IDENTITY ─── APPLICATION ─── API
   │              │            │
   ▼              ▼            ▼
POLICY ─────── DATABASE ─── SERVICE
   │              │            │
   ▼              ▼            ▼
ALPHA ZER0 ─── DATA ───── INFRASTRUCTURE
   │                           │
   ▼                           ▼
CRYPTOGRAPHY              NETWORK
                               │
                               ▼
                         THREAT WATCH

STARDANCER sits above the graph and continuously interprets it.

3. The Core Architectural Relationship

The ecosystem should converge around this model:

                    STARDANCER
              Systems Intelligence
                       │
             ┌─────────┴─────────┐
             │                   │
          SYSTEM               MIRROR
           GRAPH                BRAIN Ψ
             │                   │
             └─────────┬─────────┘
                       │
                VALIDATED STATE
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      MDLC         ALPHA ZER0     THREAT WATCH
   Governance       Security       Detection
        │              │              │
        └──────────────┼──────────────┘
                       │
                CUSTOMER SYSTEMS

And beneath that:

MDLC
 ↓
Build
 ↓
System
 ↓
Telemetry
 ↓
STARDANCER
 ↓
Understanding
 ↓
Decision
 ↓
Validation
 ↓
Action
 ↓
New System State
 ↓
STARDANCER

That creates a closed feedback system.

4. The Problem

Modern organizations have enormous amounts of telemetry but very little unified system understanding.

Security tools know about security.

Monitoring tools know about uptime.

SIEMs know about events.

CMDBs know about assets.

Cloud platforms know about resources.

Identity systems know about users.

Vulnerability platforms know about vulnerabilities.

Application monitoring knows about applications.

But these systems generally do not maintain a unified, continuously validated representation of:

how everything relates to everything else.

STARDANCER addresses that gap.

5. Who It Affects
Security teams

Need to understand whether an event represents an isolated anomaly or part of a larger attack path.

Infrastructure teams

Need to understand dependencies and cascading failures.

Application teams

Need visibility into how deployments affect downstream systems.

Cloud teams

Need a continuously changing model of cloud resources and relationships.

GRC

Needs evidence that controls actually operate within the real environment.

Executives

Need a system-level view rather than thousands of disconnected alerts.

AI agents

Need a trustworthy representation of the environment before they are allowed to act.

That last category becomes particularly important.

6. STARDANCER as the AI-Agent Context Layer

One of the strongest opportunities from the previous builds is that STARDANCER can become the context layer for autonomous AI agents.

An agent should not simply receive:

"Investigate server X."

It should receive:

SYSTEM CONTEXT
    ↓
Asset
    ↓
Identity
    ↓
Application
    ↓
Dependencies
    ↓
Recent changes
    ↓
Security events
    ↓
Vulnerabilities
    ↓
Policies
    ↓
Business criticality
    ↓
Historical behavior
    ↓
Current risk
    ↓
Allowed actions

Then the agent can reason within a bounded environment.

This connects directly to the governance principles developed for MDLC and ALPHA ZER0.

7. Mirror Brain Ψ

The Mirror Brain Ψ becomes one of the central concepts.

Ψ is not simply another LLM.

It is the reasoning mechanism operating against the observed system state.

Conceptually:

REAL SYSTEM
     │
     ▼
OBSERVATION
     │
     ▼
SYSTEM GRAPH
     │
     ▼
Ψ
     │
     ▼
SYSTEM MODEL
     │
     ▼
PREDICTION / ANALYSIS

The critical distinction is:

The AI reasons about a governed representation of reality rather than being allowed to redefine reality itself.

This preserves the principle established in the MDLC kernel work:

AI proposes. Governance validates.

8. The Evolution Loop

The existing formula becomes highly relevant to STARDANCER:

Ψn → ψ(Ψn) → Sn → λψ → Validate(Sn) → Ψn+1

Interpretation:

Ψn = current system intelligence state
ψ(Ψn) = reasoning/observation transformation
Sn = proposed system state/model
λψ = transformation/improvement function
Validate(Sn) = deterministic validation
Ψn+1 = validated next intelligence state

This gives STARDANCER a controlled learning/evolution architecture.

It should not mean unconstrained self-modifying AI.

9. System-State Formula

The other equation developed for the kernel becomes useful:

Ψ(t) = ∂Σ[Si(t) ∩ Sj(t)] / ∂t → R(t)

For STARDANCER, this can represent the rate of change in relationships between system components.

For example:

Server
   +
Identity
   +
Application
   +
Database
   +
Network
   +
Threat

creates a relationship surface.

STARDANCER observes how that surface changes over time.

The output becomes:

R(t) = System Risk / Relationship Change

This is much more powerful than simply counting alerts.

10. The System Graph

The System Graph should become the primary data structure.

Nodes could include:

USER
IDENTITY
DEVICE
SERVER
CONTAINER
VM
APPLICATION
API
DATABASE
NETWORK
FIREWALL
ROUTER
CLOUD RESOURCE
DATASET
SECRET
CERTIFICATE
KEY
POLICY
CONTROL
VULNERABILITY
THREAT
INCIDENT
DEPLOYMENT
SERVICE
BUSINESS PROCESS
AI AGENT

Edges represent relationships:

OWNS
AUTHENTICATES
CONNECTS_TO
DEPENDS_ON
CALLS
HOSTS
CONTAINS
PROTECTS
ENCRYPTS
ACCESSes
TRUSTS
DEPLOYS_TO
GENERATES
AFFECTS
VIOLATES
MITIGATES
MONITORS
11. Time Becomes a First-Class Dimension

The graph should not simply answer:

What is connected?

It should answer:

What was connected, when, why, and what changed?

Therefore:

Graph(t0)
Graph(t1)
Graph(t2)
Graph(t3)
...
Graph(tn)

becomes a sequence of system states.

That gives STARDANCER the ability to perform:

change detection
drift detection
causal analysis
historical reconstruction
incident timelines
attack-path reconstruction
dependency analysis
predictive analysis
12. Evidence Before Intelligence

A major principle from the MDLC governance work should carry directly into STARDANCER:

Intelligence cannot outrank evidence.

Therefore:

Telemetry
   ↓
Normalization
   ↓
Evidence
   ↓
Correlation
   ↓
Graph
   ↓
Reasoning
   ↓
Recommendation

Not:

LLM
 ↓
Guess
 ↓
Action

Every important conclusion should be traceable to evidence.

13. Evidence Ledger

STARDANCER should maintain an evidence ledger.

A meaningful analytical result could contain:

OBSERVATION
EVIDENCE
TIMESTAMP
SOURCE
ENTITY
RELATIONSHIP
CHANGE
RISK
MODEL
REASONING
RECOMMENDATION
VALIDATION
ACTION
OUTCOME

This gives you something extremely important for enterprise deployment:

explainability without requiring the LLM itself to be trusted.

14. Deterministic + AI Architecture

This should remain a fundamental architectural principle.

Deterministic layer

Responsible for:

identity
authorization
graph construction
event normalization
policy
risk rules
cryptographic verification
evidence
gates
audit
state transitions
safety boundaries
AI layer

Responsible for:

correlation
interpretation
hypothesis generation
prediction
summarization
anomaly reasoning
natural-language interaction
proposed remediation

Therefore:

AI proposes
     ↓
Policy evaluates
     ↓
Deterministic gates validate
     ↓
Authorized action
15. ALPHA ZER0 Integration

ALPHA ZER0 should become the cryptographic trust/control plane within this architecture.

The relationship becomes:

STARDANCER
      │
      ▼
"What is happening?"
      │
      ▼
ALPHA ZER0
      │
      ▼
"Is this identity/data/action cryptographically trusted?"

ALPHA ZER0 can protect:

sensitive telemetry
graph data
credentials
secrets
evidence
keys
agent authorization
encrypted objects
communications
audit records

And its evolutionary model:

Ψ₀ → Ψ₁ → Ψ₂ → ... → Ψn

can represent validated cryptographic state transitions.

16. Threat Watch Integration

Threat Watch becomes the security-event intelligence source.

Security Logs
     ↓
Threat Watch
     ↓
Threat Intelligence
     ↓
STARDANCER
     ↓
System Context

Instead of Threat Watch saying:

"This endpoint generated a suspicious event."

STARDANCER can answer:

"This endpoint is connected to these identities, these applications, this database, and this business process. The activity is unusual relative to its historical behavior and creates a potential path toward these assets."

That is a much more valuable product.

17. CyberOps Integration

CyberOps becomes the external perspective:

INTERNET
   ↓
ATTACK SURFACE
   ↓
CYBEROPS
   ↓
STARDANCER

STARDANCER can correlate:

External exposure
+
Internal configuration
+
Identity
+
Vulnerability
+
Threat activity
+
Business criticality

This moves the system from:

asset discovery

to:

exposure intelligence.

18. Synapse / Pulse Integration

Synapse/Pulse provides operational signals:

DNS
TCP
UDP
HTTP
HTTPS
API
SSL
SNMP
INFRASTRUCTURE

Those become another stream into the System Graph.

Now STARDANCER can correlate:

Availability
+
Performance
+
Security
+
Configuration
+
Dependencies

For example:

Application unavailable
       ↓
API failure
       ↓
Load balancer
       ↓
Network change
       ↓
Firewall policy
       ↓
Recent deployment

Instead of merely saying:

"HTTP monitor failed."

STARDANCER can reason about the system-level failure.

19. MDLC Integration

This is perhaps the most important relationship.

MDLC governs the creation and evolution of software.

STARDANCER observes what that software becomes.

MDLC
 │
 ├── RESEARCH.md
 ├── ARCHITECTURE.md
 ├── SPEC.md
 ├── FLOW.md
 ├── COMPLIANCE.md
 ├── DECISIONS.md
 ├── SECURITY-AUDIT.md
 └── REPORT.md
          │
          ▼
       BUILD
          │
          ▼
       SYSTEM
          │
          ▼
     STARDANCER

This creates a potentially powerful feedback loop:

BUILD
 ↓
OBSERVE
 ↓
UNDERSTAND
 ↓
DETECT DRIFT
 ↓
ANALYZE
 ↓
PROPOSE
 ↓
MDLC GOVERNANCE
 ↓
BUILD
20. The Self-Improvement Architecture

The Self-Improvement Agent concept can live inside STARDANCER.

Its cycle:

OBSERVE
   ↓
ANALYZE
   ↓
PROPOSE Ψ'
   ↓
TEST
   ↓
KERNEL GATE
   ↓
ACCEPT / REJECT

But the existing governance rules remain critical:

No uncontrolled recursive improvement.
No AI modification of governance rules during a governed build.
No bypassing mandatory gates.
No autonomous escalation of privileges.
High-risk changes require human approval.
Every meaningful change generates an evidence record.
21. Risk Model

The previous kernel work suggested that risk should not be invented dynamically by the AI.

Instead:

Risk =
Scope × Path Category

with a fixed governance table.

The AI can propose:

"This appears high risk."

But MDLC determines the actual governance consequence.

That distinction should carry into STARDANCER.

22. Fault-Tolerant Architecture

The ALPHA ZER0 discussions also point toward a broader STARDANCER requirement:

The intelligence layer cannot become the organization's single point of failure.

Therefore:

Collectors
   ↓
Message Bus
   ↓
Processing
   ↓
Graph Store
   ↓
State Store
   ↓
AI Workers
   ↓
Policy/Governance

should be independently recoverable.

Failure of:

LLM
AI worker
collector
network path
graph node
storage node

should not destroy authoritative system state.

This is especially important for an on-prem deployment.

23. Local AI

The security-appliance work also fits naturally here.

STARDANCER should be capable of operating with:

LOCAL LLM
+
LOCAL VECTOR/GRAPH DATA
+
LOCAL TELEMETRY
+
LOCAL POLICY
+
LOCAL CRYPTOGRAPHY

This allows sensitive organizations to retain data sovereignty.

Cloud AI can potentially be an optional capability, but the architecture should not fundamentally depend upon it.

24. On-Prem Architecture

The commercial architecture emerging from the builds is particularly strong for enterprise environments:

CUSTOMER ENVIRONMENT
─────────────────────────────

          STARDANCER
              │
      ┌───────┼────────┐
      │       │        │
    Threat   ALPHA    Synapse
    Watch    ZER0      /Pulse
      │       │        │
      └───────┼────────┘
              │
        Local AI Layer
              │
        Customer Data

The customer retains control of:

telemetry
cryptographic keys
system graph
evidence
AI models
policies
sensitive data

That creates a compelling sovereign systems-intelligence appliance.

25. STARDANCER Should Not Become a SIEM

This is an important boundary.

STARDANCER should consume SIEM/SOC telemetry rather than attempt to recreate every SIEM feature.

Likewise it should not become:

another generic CMDB
another basic monitoring platform
another vulnerability scanner
another EDR
another ticketing system

Its differentiation is:

cross-domain system intelligence.

26. What Makes It Different

The fundamental differentiation becomes:

Traditional tools:

EVENT → ALERT

STARDANCER:

EVENT
 ↓
ENTITY
 ↓
RELATIONSHIP
 ↓
SYSTEM STATE
 ↓
CHANGE
 ↓
CONTEXT
 ↓
RISK
 ↓
PREDICTION
 ↓
RECOMMENDATION
 ↓
VALIDATED ACTION

That is a substantially different architecture.

27. Key Workflows
1. Security investigation
Alert
→ affected asset
→ identity
→ dependencies
→ historical behavior
→ attack path
→ business impact
→ recommended response
2. Outage investigation
Failure
→ dependency graph
→ recent changes
→ infrastructure
→ network
→ application
→ probable root cause
3. Change impact
Proposed change
→ affected nodes
→ dependent services
→ security implications
→ business impact
→ risk
4. AI-agent authorization
Agent request
→ identity
→ current system state
→ policy
→ permitted scope
→ risk
→ approval
→ execution
→ verification
5. Cryptographic event
Key/object event
→ ALPHA ZER0
→ identity
→ affected data
→ system dependencies
→ risk
→ rotation/revocation
→ validation
6. Continuous system discovery
Telemetry
→ discover
→ correlate
→ graph
→ validate
→ update
→ observe again
28. Success Metrics

STARDANCER should ultimately be measured on outcomes rather than AI benchmarks.

System understanding
% of known assets represented
% of known dependencies represented
graph freshness
relationship accuracy
Security
reduction in investigation time
reduction in alert triage time
attack-path discovery time
mean time to understand incident impact
Operations
MTTR reduction
root-cause identification time
change-impact prediction accuracy
outage correlation accuracy
AI
evidence-backed conclusion rate
hallucination rate
recommendation acceptance rate
false-positive rate
prediction accuracy
Governance
unauthorized action rate = 0
policy bypass rate = 0
unvalidated state transitions = 0
audit completeness = 100%
29. Non-Goals

STARDANCER should not:

replace humans for high-risk decisions
blindly execute AI recommendations
become an unconstrained self-modifying AI
treat LLM output as authoritative truth
replace every security product
replace every monitoring product
become a generic chatbot
expose sensitive telemetry unnecessarily
allow AI to modify governance boundaries
make cryptographic claims such as "mathematically unbreakable"

The architecture should aim for verifiable security, not marketing mythology.

30. Brand Voice

STARDANCER should feel:

Intelligent.

Not gimmicky.

Cosmic.

But technically grounded.

Precise.

Not mystical.

Quietly powerful.

Not loud enterprise marketing.

Observational.

It sees relationships others miss.

Potential language:

See the system behind the signals.

Every system is a constellation.

Understand what is connected.

Know what changed.

See what comes next.

Intelligence for systems in motion.

The name can carry the visual identity while the technical architecture establishes credibility.

31. Design Language

The visual language should naturally extend what you've already been developing:

VOID
   ↓
CONSTELLATION
   ↓
NODES
   ↓
RELATIONSHIPS
   ↓
FLOW
   ↓
STATE

Dark environment.

Sparse luminous elements.

System nodes.

Graph relationships.

Temporal movement.

Threat/risk represented through state rather than decorative dashboards.

The interface should feel less like:

"Here are 17 dashboards."

and more like:

"Here is your living system."

32. The Bigger Architecture

The strongest version of the entire ecosystem now looks like this:

                         STARDANCER
                   SYSTEMS INTELLIGENCE
                              │
                 ┌────────────┴────────────┐
                 │                         │
          SYSTEM GRAPH               MIRROR BRAIN Ψ
                 │                         │
                 └────────────┬────────────┘
                              │
                       GOVERNED STATE
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
      MDLC                ALPHA ZER0          OBSERVABILITY
   BUILD GOVERNANCE      TRUST/CRYPTO        SYNAPSE/PULSE
        │                     │                     │
        │                     │                     │
        └──────────────┐      │      ┌──────────────┘
                       │      │      │
                       ▼      ▼      ▼
                    THREAT WATCH / CYBEROPS
                           SECURITY
                              │
                              ▼
                    CUSTOMER ENVIRONMENT

And underneath everything:

                    EVIDENCE
                       │
                    POLICY
                       │
                  VALIDATION
                       │
                    AUDIT
                       │
                  HUMAN CONTROL
33. The Fundamental STARDANCER Formula

I would make this the conceptual foundation:

SYSTEM(t)
   ↓
OBSERVE
   ↓
GRAPH(t)
   ↓
Ψ(G(t))
   ↓
UNDERSTAND
   ↓
PREDICT
   ↓
PROPOSE
   ↓
VALIDATE
   ↓
ACT
   ↓
SYSTEM(t+1)

Or mathematically:

G(t+1) = Validate[ G(t) + ΔEvidence(t) + ΔRelationships(t) ]

with:

Ψ(t) = Intelligence(G(t), Evidence(t), Policy(t))

and:

Action(t) = Gate[ Ψ(t), Policy(t), Risk(t), Authorization(t) ]

That last equation is important because it prevents STARDANCER from becoming an autonomous black box.

34. The Strategic Position

The most interesting thing about what you've been building is that these projects no longer need to be marketed as unrelated products.

They can form a coherent architecture:

MDLC

Govern the creation of software.

STARDANCER

Understand the living system.

ALPHA ZER0

Establish cryptographic trust and sovereignty.

Threat Watch

Understand security threats.

CyberOps

Understand external exposure.

Synapse / Pulse

Understand operational health.

And the common foundation is:

Observe → Model → Reason → Validate → Act → Observe again.

That is the architecture I would use as the master context for the STARDANCER build going forward.


## Users & Outcomes
**Key Workflows:**
Workflow A: Understand the environment
The user opens the dashboard.
STARDANCER presents the observed system graph.
The user selects an entity.
The system shows its dependencies.
The user examines related assets and events.
The user reviews supporting evidence.

Success: The user can understand an entity and its relationships without manually correlating several separate tools.

Workflow B: Investigate a change
A new observation arrives.
The system compares it with the previous state.
The changed entity and affected relationships are identified.
Potential consequences are evaluated.
The user drills into the relevant evidence.
The investigation produces a traceable result.

Success: The user can move from a detected change to its evidence and potential impact.

Workflow C: Evaluate a prediction
The Mirror Brain evaluates the current model.
A candidate future state is generated.
The system records the assumptions behind the hypothesis.
Available evidence is evaluated.
Confidence and uncertainty are communicated.
The user reviews the result.

Success: Predictions are explainable and distinguishable from observed facts.

Workflow D: Propose an improvement
An agent identifies a potential improvement.
The proposed change is documented.
The change is evaluated against policies.
Required validation checks run.
Human approval is obtained when required.
An authorized operation may proceed.
The outcome is recorded.

Success: Every material change has a traceable proposal, validation outcome, and authorization history.

# Key Workflows

STARDANCER is built around a continuous intelligence loop rather than isolated dashboards or one-time investigations.

**Observe → Correlate → Understand → Predict → Validate → Act → Verify**

## 1. Continuous Environment Discovery

STARDANCER continuously discovers and models the organization's environment.

**Telemetry → Entity Discovery → Relationship Mapping → System Graph → Validation**

It identifies:

* Assets
* Identities
* Applications
* Services
* APIs
* Databases
* Networks
* Cloud resources
* Dependencies
* Security controls
* AI agents

The result is a continuously evolving representation of the actual environment.

---

## 2. Security Event Investigation

A security event enters STARDANCER from Threat Watch, EDR, SIEM, firewall, identity, cloud, or another security source.

**Event → Entity → Relationships → Historical Context → Risk → Attack Path → Recommendation**

STARDANCER determines:

* What generated the event
* Which identity is involved
* What asset is affected
* What the asset connects to
* Whether similar activity occurred previously
* What vulnerabilities exist
* What business systems depend on it
* Whether the event is part of a larger attack

The analyst receives system context rather than an isolated alert.

---

## 3. Attack-Path Analysis

STARDANCER evaluates relationships between exposures, identities, vulnerabilities, privileges, and critical assets.

**Exposure → Vulnerability → Identity → Access → Dependency → Critical Asset**

It identifies potential paths through the environment and prioritizes paths based on:

* Exploitability
* Asset criticality
* Identity privilege
* Exposure
* Business impact
* Existing controls

The objective is to understand **what an attacker could reach**, not simply how many vulnerabilities exist.

---

## 4. Root-Cause Analysis

When an outage or abnormal condition occurs:

**Failure → Recent Changes → Dependencies → Infrastructure → Network → Application → Probable Cause**

STARDANCER correlates operational telemetry with:

* Deployments
* Configuration changes
* Network changes
* Infrastructure state
* Application behavior
* Security events
* Historical incidents

The result is a ranked explanation of what most likely caused the condition.

---

## 5. Change-Impact Analysis

Before or immediately after a change:

**Change → Affected Entities → Dependencies → Security Impact → Business Impact → Risk**

STARDANCER determines what could be affected by:

* Application deployments
* Infrastructure changes
* Firewall rules
* Identity changes
* Configuration changes
* Database changes
* API changes
* Cloud changes

This provides a continuously updated impact model.

---

## 6. Drift Detection

STARDANCER compares intended and observed system states.

**Expected State → Observed State → Difference → Risk → Recommendation**

Drift can include:

* Configuration changes
* Unexpected services
* New network relationships
* Unauthorized assets
* Identity changes
* Security-control changes
* Architecture changes

This extends traditional configuration monitoring into system-level drift detection.

---

## 7. Incident Reconstruction

After or during an incident, STARDANCER reconstructs the timeline.

**State(t₀) → Event → Change → Relationship → Event → State(tₙ)**

It correlates:

* Authentication events
* Endpoint activity
* Network traffic
* Application events
* Cloud events
* Security alerts
* Configuration changes
* Data access
* Identity activity

The goal is to establish:

**What happened → When → Where → How → What was affected → What happened next**

---

## 8. Vulnerability Prioritization

Instead of treating every vulnerability equally:

**Vulnerability → Asset → Exposure → Dependency → Business Criticality → Attack Path → Risk**

STARDANCER can prioritize vulnerabilities according to their position in the actual system.

A severe vulnerability on an isolated asset may be less urgent than a moderate vulnerability that provides a path toward a critical business system.

---

## 9. AI-Agent Context and Authorization

When an AI agent requests an action:

**Agent → Identity → Requested Action → System Context → Policy → Risk → Authorization**

STARDANCER determines:

* Who the agent is
* What it is trying to do
* Which systems it will affect
* What dependencies exist
* What data is involved
* What policies apply
* What risk exists
* Whether approval is required

The action then passes through governance before execution.

**AI proposes. Governance validates.**

---

## 10. Automated Response

For authorized low-risk actions:

**Detection → Analysis → Recommendation → Policy Gate → Action → Verification**

Examples could include:

* Isolating a compromised endpoint
* Disabling a compromised credential
* Blocking malicious network activity
* Restarting a failed service
* Rotating a credential
* Opening an incident
* Notifying an operator

High-risk actions can require human approval.

---

## 11. Cryptographic Trust Workflow

ALPHA ZER0 provides the cryptographic trust layer.

**Object/Event → Identity → Cryptographic Verification → Trust Decision → Policy**

This can be used to validate:

* Identities
* Keys
* Certificates
* Data
* Evidence
* Agent actions
* Communications
* System state

STARDANCER uses that trust information when evaluating the larger system.

---

## 12. Operational Health Correlation

Synapse/Pulse telemetry enters the same system model.

**DNS + Network + TLS + API + Infrastructure + Application → System Health**

Instead of treating a failed monitor as an isolated event, STARDANCER examines the dependency graph to determine whether the failure is:

* Local
* Network-related
* Application-related
* Infrastructure-related
* Security-related
* Dependency-related

---

## 13. External Exposure Analysis

CyberOps provides external attack-surface information.

**Internet → Exposure → Asset → Service → Vulnerability → Internal Dependency → Risk**

STARDANCER correlates external exposure with internal system context.

This allows organizations to understand not merely:

> "This port is open."

but:

> **"This exposed service provides a potential path toward this critical internal capability."**

---

## 14. Continuous Compliance

STARDANCER continuously compares observed system behavior against defined controls.

**Control → Expected State → Observed State → Evidence → Exception → Risk**

This creates a move from periodic compliance assessments toward continuous evidence.

---

## 15. Self-Improvement

The intelligence layer can propose improvements to its own understanding.

**Observe → Analyze → Propose Ψ′ → Test → Kernel Gate → Accept / Reject**

The system may improve models, correlations, or analytical strategies, but governance remains outside the AI's authority.

No self-improvement process can:

* Bypass mandatory gates
* Modify governance rules during a governed build
* Escalate privileges autonomously
* Suppress evidence
* Redefine its own authorization boundaries

---

# The Master Workflow

All STARDANCER workflows ultimately converge on one continuous loop:

**OBSERVE**

↓

**BUILD SYSTEM STATE**

↓

**CORRELATE RELATIONSHIPS**

↓

**MIRROR BRAIN Ψ**

↓

**UNDERSTAND**

↓

**PREDICT**

↓

**PROPOSE**

↓

**VALIDATE**

↓

**ACT**

↓

**VERIFY**

↓

**UPDATE SYSTEM STATE**

↓

**OBSERVE AGAIN**

The result is not another collection of disconnected security and monitoring workflows.

It is a **continuous intelligence loop around the entire digital environment**.

**Success Metrics:**
The initial website and dashboard should be evaluated against these measurable outcomes.

Area	Success criterion
Comprehension	A new user understands the product's purpose within 30 seconds
Visualization	Users can select an entity and inspect its relationships
Explainability	Material findings expose their supporting evidence
Temporal awareness	Users can distinguish current state from historical state
Prediction	Hypotheses expose assumptions and uncertainty
Governance	Proposed actions cannot bypass required validation
Usability	Primary workflows work on desktop and mobile
Performance	Graph rendering remains responsive at the declared test scale
Accessibility	Keyboard navigation, focus states, contrast, and reduced motion are supported
Reliability	Errors and missing data are surfaced explicitly

Do not claim that these targets have been achieved until they have been measured.

# Success Metrics

STARDANCER succeeds when it turns fragmented enterprise telemetry into an accurate, continuously validated understanding of the environment and uses that understanding to improve security, operations, resilience, and decision-making.

The primary measurement is not how much data STARDANCER collects.

**It is how much better the organization understands and controls its systems.**

## 1. System Understanding

### System Graph Coverage

Measure the percentage of known enterprise entities represented in the System Graph.

**Target:** ≥ 95% of discoverable critical assets.

### Relationship Accuracy

Measure the percentage of modeled relationships confirmed as accurate.

**Target:** ≥ 98% for critical relationships.

### Graph Freshness

Measure the time between a material environmental change and its representation in STARDANCER.

**Target:** Near-real-time for critical telemetry — measured as p95 observation-accepted → graph-updated ≤ 60 s (PBR-3).

### Unknown Asset Rate

Measure previously unknown assets discovered by STARDANCER.

**Target:** Continuous reduction in unidentified assets.

---

# 2. Security Intelligence

### Mean Time to Understand — MTTU

Measure the time required to understand the scope and significance of a security event.

**Target:** 50–80% reduction.

### Alert Investigation Time

Measure analyst time required to investigate security alerts.

**Target:** ≥ 50% reduction.

### Attack-Path Discovery

Measure the time required to identify viable paths from exposure to critical assets.

**Target:** Minutes rather than hours or days.

### False-Positive Reduction

Measure the reduction in alerts requiring manual investigation but ultimately determined to be benign.

**Target:** Continuous reduction without increasing missed threats.

---

# 3. Incident Response

### Mean Time to Detect — MTTD

Measure the time between malicious activity and detection.

**Target:** Continuous reduction.

### Mean Time to Respond — MTTR

Measure the time between detection and validated response.

**Target:** ≥ 50% reduction.

### Incident Reconstruction Time

Measure how long responders require to reconstruct:

**Who → What → When → Where → How → Impact**

**Target:** Hours reduced to minutes for supported data sources.

### Incident Context Completeness

Measure the percentage of significant incidents for which STARDANCER can establish affected entities, relationships, timeline, and evidence.

**Target:** ≥ 95%.

---

# 4. Operational Intelligence

### Root-Cause Identification Time

Measure the time required to identify the probable cause of an outage or abnormal condition.

**Target:** ≥ 50% reduction.

### Root-Cause Accuracy

Measure how frequently STARDANCER's highest-ranked probable cause is subsequently confirmed.

**Target:** ≥ 90% for supported workflows.

### Change-Impact Accuracy

Measure how accurately STARDANCER predicts affected systems before or immediately after a change.

**Target:** ≥ 95% for critical dependencies.

### Mean Time to Recovery

Measure improvement in recovery time for incidents where STARDANCER provides system context.

**Target:** ≥ 30–50% reduction.

---

# 5. Vulnerability Intelligence

### Critical-Risk Prioritization Accuracy

Measure whether STARDANCER correctly identifies vulnerabilities that create meaningful organizational risk.

**Target:** High correlation between prioritized findings and validated attack paths or business impact.

### Time to Identify Exploitable Paths

Measure the time required to connect:

**Exposure → Vulnerability → Identity → Access → Critical Asset**

**Target:** Minutes.

### Vulnerability Noise Reduction

Measure the reduction in vulnerabilities requiring immediate human attention after contextual prioritization.

**Target:** Significant reduction without increasing missed critical risk.

---

# 6. AI Intelligence

STARDANCER must measure AI quality differently from conventional AI products.

### Evidence-Backed Conclusion Rate

Percentage of significant AI conclusions supported by identifiable evidence.

**Target:** ≥ 99%.

### Unsupported Conclusion Rate

Percentage of significant conclusions that cannot be adequately supported by system evidence.

**Target:** Approaches 0%.

### Recommendation Accuracy

Percentage of recommendations determined to be appropriate after human or deterministic validation.

**Target:** ≥ 95%.

### Context Completeness

Percentage of AI decisions that receive the relevant system, identity, dependency, policy, and risk context.

**Target:** ≥ 99% for governed actions.

---

# 7. Autonomous-Agent Safety

This is one of STARDANCER's most important metrics.

### Unauthorized Actions

**Target: 0**

### Governance Bypasses

**Target: 0**

### Unvalidated High-Risk Actions

**Target: 0**

### Incorrect Privilege Escalations

**Target: 0**

### Human-Approval Compliance

Percentage of actions correctly routed for required human approval.

**Target: 100%.**

The goal is not maximum autonomy.

The goal is **maximum useful autonomy within validated boundaries.**

---

# 8. System Resilience

### Availability

Measure STARDANCER platform availability.

**Target:** Enterprise-grade availability — 99.9 % monthly SLO on /api/health (PBR-4).

### Data Durability

Measure preservation of system state and evidence during component failures.

**Target:** No loss of authoritative evidence.

### Recovery Time Objective

Measure time required to restore operational intelligence after infrastructure failure.

**Target:** Defined and tested enterprise RTO — RTO ≤ 1 h, RPO ≤ 5 min (Neon PITR; local: pg_dump runbook) (PBR-4).

### AI Independence

Measure whether core governance and system state remain functional when AI services are unavailable.

**Target:** Core system remains operational without an LLM.

---

# 9. Evidence and Governance

### Evidence Completeness

Percentage of significant decisions containing sufficient evidence for reconstruction.

**Target:** 100%.

### Decision Traceability

Percentage of consequential actions traceable to:

**Evidence → Reasoning → Policy → Authorization → Action → Outcome**

**Target:** 100%.

### Audit Reconstruction

Time required to reconstruct why a consequential action occurred.

**Target:** Minutes.

### Policy Enforcement

Percentage of governed actions evaluated against applicable policy.

**Target:** 100%.

---

# 10. Compliance

### Continuous Control Coverage

Percentage of applicable controls continuously monitored.

**Target:** ≥ 95% of supported controls.

### Evidence Collection Automation

Percentage of audit evidence collected automatically.

**Target:** ≥ 80%.

### Control Drift Detection

Time between a control deviation and detection.

**Target:** Near-real-time where telemetry supports it.

---

# 11. Business Impact

Technical metrics ultimately need to translate into business outcomes.

Measure:

* Reduced downtime
* Reduced incident impact
* Reduced investigation labor
* Reduced security exposure
* Faster remediation
* Reduced operational risk
* Reduced audit effort
* Improved change success rate
* Reduced business disruption

The ultimate question is:

> **Does STARDANCER make the organization safer, more resilient, and easier to operate?**

---

# 12. Adoption Metrics

### Time to First Useful Intelligence

Time from deployment to the first meaningful system-level insight.

**Target:** Hours, not weeks.

### Time to Build System Graph

Time required to establish an initial representation of the environment.

**Target:** Hours to days depending on environment size.

### Analyst Adoption

Percentage of target analysts actively using STARDANCER during investigations.

**Target:** ≥ 80%.

### Investigation Dependency

Percentage of supported investigations where analysts use STARDANCER as a primary contextual source.

**Target:** ≥ 70%.

---

# 13. The North-Star Metrics

The entire platform can ultimately be reduced to several North-Star measurements:

### **1. System Understanding**

**How much of the real environment does STARDANCER accurately understand?**

### **2. Time to Understand**

**How quickly can humans understand what is happening?**

### **3. Prediction Accuracy**

**How accurately can STARDANCER predict impact and risk?**

### **4. Evidence Integrity**

**Can every consequential conclusion be traced back to evidence?**

### **5. Governed Autonomy**

**How much useful automation can occur without violating policy?**

### **6. Business Impact**

**Does that intelligence measurably reduce risk, downtime, and operational cost?**

---

# The Ultimate Success Metric

STARDANCER should ultimately move an organization from:

**"We have thousands of signals and alerts."**

to:

**"We understand our system."**

And from:

**"Something changed."**

to:

**"We know what changed, why it matters, what it affects, what is likely to happen next, and what we can safely do about it."**

That is the fundamental measure of STARDANCER's success.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
monitoring: "Prometheus + Grafana"

# Data & Storage
database_preference: "Neon"

# Security & Compliance
rate_limiting: true
audit_logging: true

# Frontend
frontend_framework: "Next.js"
state_management: "Redux"

# Backend
backend_framework: "Express"
api_style: "REST"
orm_preference: "Prisma"
realtime_needed: true
background_jobs: "BullMQ"

# Performance & Quality
performance_requirements: ["sustained load test at 50k+ concurrent users"]
logging_format: "structured JSON"

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
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
Composed and dense without clutter. Calm neutral surfaces, a disciplined grid, tables and forms done well, restrained accent use, subtle depth. Functional motion, not decoration.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2665ed | Buttons, links, active states |
| Secondary | #aa30e3 | Accents, badges, highlights |
| Accent | #deb151 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #16181d | Headings, body text |
| Text Secondary | #5c6270 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2665ed | Buttons, links, active states |
| Secondary | #ac35e4 | Accents, badges, highlights |
| Accent | #deb254 | Callouts, hover states |
| Background | #090a0c | Page background |
| Surface | #121417 | Cards, elevated containers |
| Text | #eaeaec | Headings, body text |
| Text Secondary | #838995 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

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
- Theme: Dark only — DESIGN-TEMPLATE.html palette is canonical (pre-build review PBR-1; was "Light only")

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#2665ed',
    secondary: '#aa30e3',
    accent: '#deb151',
    background: '#fafafa',
    surface: '#f4f5f5',
    foreground: '#16181d',
    muted: '#5c6270',
    success: '#1fad53',
    warning: '#ec9c13',
    destructive: '#df2020',
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
  --color-primary: #2665ed;
  --color-secondary: #aa30e3;
  --color-accent: #deb151;
  --color-background: #fafafa;
  --color-surface: #f4f5f5;
  --color-text: #16181d;
  --color-text-secondary: #5c6270;
  --color-success: #1fad53;
  --color-warning: #ec9c13;
  --color-error: #df2020;
  --font-heading: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 2px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #2665ed;
  --color-secondary: #ac35e4;
  --color-accent: #deb254;
  --color-background: #090a0c;
  --color-surface: #121417;
  --color-text: #eaeaec;
  --color-text-secondary: #838995;
  --color-success: #33cc6b;
  --color-warning: #e2a336;
  --color-error: #d74242;
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

# Research Sections (Stage 0 — filled by the build agent, 2026-10-12)

> Stage 0 research gate: §3 was absent from the wizard-generated RESEARCH.md, so research was run per Appendix A at `build_depth: comprehensive` (all categories, ≥2–3 analyzed entries each, threat model in §5). Package versions were confirmed with `npm view <pkg> version` on 2026-10-12, not from memory.

## 3. Source Categories

### 3.1 — Official Documentation & Vendor Sources

| Source | URL | What It Covers | Notes |
|--------|-----|----------------|-------|
| Next.js — Content Security Policy guide | https://nextjs.org/docs/15/app/guides/content-security-policy | Per-request nonce generated in middleware/proxy, `'strict-dynamic'`, `x-nonce` header, dynamic rendering requirement | Profile N CSP (WEB.md §6). Next 16.4.0 current; v16 uses `proxy.ts`. |
| Prisma ORM — Neon guide | https://www.prisma.io/docs/orm/v6/overview/databases/neon | Pooled (`-pooler`) URL for the client, direct URL for Migrate | `prisma`/`@prisma/client` 7.10.0 stable (8.0 is RC — not used). |
| Neon — Prisma DX / pooling | https://neon.com/blog/prisma-dx-improvements | PgBouncer transaction pooling up to 10k client connections; direct URL for migrations | Drives the large-tier pooling row. |
| BullMQ — Job Schedulers | https://docs.bullmq.io/guide/job-schedulers/manage-job-schedulers | `upsertJobScheduler` (replaces the deprecated repeatable API), cron/every patterns, `tz` | bullmq 6.3.12. Delivery is at-least-once → idempotent claim is the app's job. |
| Stripe — Idempotent requests | https://docs.stripe.com/api/idempotent_requests | `Idempotency-Key` header; first result cached per key | `stripe` 23.0.0. Webhooks: `constructEvent(rawBody, sig, secret)`; Stripe retries for up to 3 days. |
| Express 5 migration guide | https://expressjs.com/en/guide/migrating-5.html | Async error propagation, path-to-regexp v8 route syntax | express 5.3.0. |
| Redux Toolkit / RTK Query | https://redux-toolkit.js.org/rtk-query/overview | Cache, tag invalidation, typed hooks | @reduxjs/toolkit 2.13.0, react-redux 9.3.0 — honours `state_management: Redux`. |
| prom-client | https://github.com/siimots/prom-client | Prometheus exposition for Node (default + custom metrics) | 15.1.3 — honours `monitoring: Prometheus + Grafana`. |
| Caddy — `tls` directive | https://caddyserver.com/docs/caddyfile/directives/tls | `tls internal` local CA, automatic `:80`→`:443` redirect | The TLS-terminating edge required by `protocol_support: HTTPS only`. |

**Key Takeaways:**
- Nonce CSP in Next.js forces dynamic rendering of HTML routes; public pages must still stream fast with a small JS payload.
- Prisma needs a direct (non-pooled) URL for migrations; the app uses the pooled URL → `DATABASE_URL`, `DIRECT_URL`, plus optional `DATABASE_REPLICA_URL` for the large-tier read split.
- BullMQ job schedulers are at-least-once; every scheduled job needs a DB-level idempotent claim (BUILD.md `has_scheduled_work` #4).
- Stripe correctness = idempotency key derived from the business action + webhook as source of truth.

### 3.2 — GitHub & Open Source

| Repository | URL | Stars | Last Active | Relevance |
|------------|-----|-------|-------------|-----------|
| SpecterOps/BloodHound (CE) | https://github.com/SpecterOps/BloodHound | ~2k CE (legacy ~10k) | 2025–2026 (v8 OpenGraph, Jul 2025) | Attack-path graphs over identities; OpenGraph = generic node/edge ingestion schema. Apache-2.0 — reference only. |
| cartography-cncf/cartography | https://github.com/cartography-cncf/cartography | ~3.5k | Active (CNCF Sandbox) | Asset+relationship graph ingestion; entity/edge typing and sync-tag staleness model. Apache-2.0 — reference only. |
| ocsf/ocsf-schema | https://github.com/ocsf/ocsf-schema | ~800 | Active | Vendor-neutral event normalization (category → class → attributes); observation envelope borrows class ids. Apache-2.0. |
| cytoscape/cytoscape.js | https://github.com/cytoscape/cytoscape.js | ~10k | Active (3.34.3) | Candidate graph renderer; MIT. Rejected for first-load budget in favour of a bespoke SVG constellation (§4). |
| taskforcesh/bullmq | https://github.com/taskforcesh/bullmq | ~7k | Active | Queue/workers/schedulers; MIT; dependency. |
| prisma/prisma | https://github.com/prisma/prisma | ~42k | Active | ORM + migrations; Apache-2.0; dependency. |

**Evaluation Checklist:**
- [x] License compatible — all dependencies MIT/Apache-2.0/BSD.
- [x] Actively maintained (commits within 6 months).
- [x] Documentation / examples available for every dependency.
- [x] Community size large for every runtime dependency.
- [x] Dependency vs reference: BloodHound, Cartography, OCSF = reference only; BullMQ, Prisma, Express, Next, RTK = dependencies.

**Key Takeaways:**
- Prior art stores the graph in Neo4j; STARDANCER stores it relationally (Postgres adjacency tables + bounded BFS) to honour `database_preference: Neon` and keep one transactional store — depth is bounded (≤ 6 hops), so this is tractable.
- Typed nodes/edges with `firstSeen`/`lastSeen` and source provenance is the common schema shape; adopted.

### 3.3 — Video & Tutorial Sources

| Title | URL | Creator | Length | Why It's Useful |
|-------|-----|---------|--------|-----------------|
| How Graphs Changed The Way Hackers Attack | https://neo4j.com/videos/how-graphs-changed-the-way-hackers-attack-andy-robbins-bloodhound/ | Andy Robbins (SpecterOps), GraphConnect 2017 | ~40 min | Attack paths as traversal from low- to high-privilege nodes; motivates the attack-path workflow. |
| Cartography: Using Graphs to Improve and Scale Security Decision-Making | https://slideslive.com/38942582/cartography-using-graphs-to-improve-and-scale-security-decisionmaking | Lyft security team | ~30 min | Continuous asset/relationship discovery; staleness and sync-tagging. |
| From Side Project to Startup: Graph Lessons from the Cartography Journey | https://neo4j.com/nodes-2025/agenda/from-side-project-to-startup-graph-lessons-from-the-cartography-journey/ | NODES 2025 | ~30 min | Scaling an asset graph; schema evolution. |

**Key Takeaways:**
- Sync-tagging (stamp every node/edge with the ingestion run; mark stale what the last run did not see) is the proven drift primitive.
- Attack-path value comes from ranking paths to crown-jewel assets, not listing all paths.

### 3.4 — Articles, Blogs & Written Tutorials

| Title | URL | Author/Site | Date | Relevance |
|-------|-----|-------------|------|-----------|
| Wiz Security Graph — architecture overview | https://www.puppygraph.com/blog/wiz-security-graph | PuppyGraph | 2026-04-08 | Graph as platform foundation; contextual risk; blast radius. |
| Alert fatigue in cybersecurity | https://www.wiz.io/academy/detection-and-response/alert-fatigue-in-cybersecurity | Wiz Academy | 2025 | Correlating findings into one path instead of N alerts. |
| Tamper-evident audit trails in PostgreSQL | https://appmaster.io/blog/tamper-evident-audit-trails-postgresql | AppMaster | 2025 | Hash-chain rows; no UPDATE/DELETE; periodic verification. |
| Building a tamper-evident audit log with hash chains | https://dev.to/elwin_ernst/building-a-tamper-evident-audit-log-in-nestjs-with-hash-chains-4l76 | DEV | 2025 | App-layer hashing + verification walk. |
| How Reddit uses a graph-based CMDB | https://www.jupiterone.com/blog/how-reddit-uses-a-graph-based-cmdb | JupiterOne | — | Graph CMDB for alert triage and incident analysis. |
| Socket.IO with multiple servers | https://oneuptime.com/blog/post/2026-01-24-socketio-multiple-servers/markdown | OneUptime | 2026-01-24 | Sticky sessions + Redis adapter requirement; informs the SSE choice. |

**Key Takeaways:**
- A hash chain in the same DB is only tamper-*evident* if the head is anchored elsewhere → HMAC checkpoints keyed outside the DB and a verification job that actually runs.
- Context-layered correlation is the core value claim; every conclusion must cite evidence (RESEARCH "Evidence-First Intelligence").

### 3.5 — Standards, RFCs & Specifications

| Standard | Reference | URL | Applicability |
|----------|-----------|-----|---------------|
| Problem Details for HTTP APIs | RFC 9457 (obsoletes 7807) | https://www.rfc-editor.org/rfc/rfc9457 | SPEC §7 error envelope (`application/problem+json`). |
| OCSF | schema 1.x | https://schema.ocsf.io | Observation category/class ids for ingested events. |
| MITRE ATT&CK Enterprise | current matrix | https://attack.mitre.org/matrices/enterprise/ | Technique tags on attack-path edges and investigations. |
| NIST CSF 2.0 | CSWP 29 (Feb 2024) | https://www.nist.gov/cyberframework | Control catalogue for continuous compliance (Govern/Identify/Protect/Detect/Respond/Recover). |
| NIST SP 800-63B | password verifiers | https://pages.nist.gov/800-63-4/sp800-63b.html | Length-based password policy, breached-list screen, no composition rules. |
| OWASP Top 10 | 2021 | https://owasp.org/Top10/ | Security floor (no `security_baseline` regime named). |
| WCAG 2.2 AA | W3C Rec | https://www.w3.org/TR/WCAG22/ | UI accessibility floor. |
| Gartner CTEM (via XM Cyber) | 5 stages | https://xmcyber.com/blog/xm-cyber-named-a-challenger-in-the-2025-gartner-magic-quadrant-for-exposure-assessment-platforms-eaps/ | Scoping → Discovery → Prioritization → Validation → Mobilization maps onto OBSERVE → UNDERSTAND → PREDICT → VALIDATE → ACT. |

**Key Takeaways:**
- `security_baseline` is blank → OWASP Top 10 floor (ADR); the 800-63B verifier is adopted because auth is password-based.
- Compliance controls are seeded from NIST CSF 2.0 (public), not licensed catalogues.

### 3.6 — Competing / Adjacent Products

| Product | URL | Pricing | Strengths | Weaknesses | Our Differentiation |
|---------|-----|---------|-----------|------------|---------------------|
| Wiz | https://wiz.io/lp/wiz-security-graph | Enterprise, quote | Agentless cloud graph, attack paths, blast radius | Cloud-centric; SaaS-hosted | Hybrid/on-prem/AI-agent entities; sovereign deployment; predictions with explicit assumptions. |
| JupiterOne | https://www.jupiterone.com | Free tier + enterprise | Graph CMDB, J1QL, compliance | Query-centric; no governed action loop | Closed OBSERVE → ACT loop with governed agent actions. |
| XM Cyber | https://xmcyber.com | Enterprise | Attack-path simulation, CTEM | Security-only | Security + operations + change in one graph. |
| Axonius | https://devstack.axonius.com/blog/why-axonius-is-the-backbone-of-a-successful-ctem-strategy-moving-from-reactive-to-proactive-security | Enterprise | Asset inventory aggregation | Weak relationship reasoning | Relationship-first model, evidence ledger. |
| BloodHound CE | https://specterops.io/blog/2025/07/29/bloodhound-community-edition-v8-launches-with-opengraph/ | Free / Enterprise | Identity attack paths | Identity-only scope | Whole-system scope. |

**Key Takeaways:**
- The market validates graph + attack-path + CTEM; differentiation is (1) evidence-backed predictions explicitly distinguished from facts, (2) governed autonomy (policy → validation → approval → recorded outcome), (3) tamper-evident evidence chain, (4) sovereign self-hosted deployment.

### 3.7 — Community & Forums

| Thread/Post | URL | Platform | Key Insight |
|-------------|-----|----------|-------------|
| ServiceNow false-positive change-risk alerts: CMDB root cause | https://virima.com/blog/servicenow-false-positive-change-risk-alerts-cmdb | Practitioner blog | Stale relationships make change-risk scoring wrong → freshness must be first-class. |
| Alert fatigue in cybersecurity | https://www.n-able.com/de/blog/alert-fatigue-cybersecurity | MSP community blog | Analysts dismiss rather than investigate under volume → group by root cause. |
| BloodHound discussion | https://wxzte24279.lithium.com/t5/neo4j-graph-platform/bloodhound/m-p/38642 | Neo4j community | Practitioners value path ranking and query explainability. |

**Key Takeaways:**
- Practitioners distrust confidently-wrong tools; freshness and confidence are shown next to every fact and prediction.
- Unknown/unmanaged assets drive incidents (Trend Micro study cited by Virima: 74 %) → an Unknown Asset Rate KPI.

### 3.8 — APIs & Integrations

| API/Service | URL | Auth Method | Rate Limits | Docs Quality | SDK Available? |
|-------------|-----|-------------|-------------|--------------|----------------|
| Stripe (Checkout, Subscriptions, Refunds, Webhooks) | https://docs.stripe.com/api | Secret key; webhook HMAC (`Stripe-Signature`) | 100 r/s live, 25 r/s test | Excellent | `stripe` 23.0.0 |
| stripe-mock | https://github.com/stripe/stripe-mock | none | n/a | Good | Docker image |
| Neon Postgres | https://neon.com/docs | Connection string over TLS | pooler 10k conns | Excellent | via Prisma |
| Observation ingest (inbound, ours) | SPEC §8 | Per-source API key (hashed) | per-key token bucket | — | — |
| Prometheus / Grafana | https://prometheus.io/docs | private-network scrape | n/a | Excellent | prom-client |
| Map tiles | — | — | — | — | **None** — geography rendered as self-drawn SVG projection; no billable map provider (has_geo cost-abuse vector removed by design). |

**Key Takeaways:**
- Without Stripe keys the local stack uses signed webhook fixtures and a mock checkout; live Stripe is a QUICKSTART provisioning step.
- No external map provider; coordinate bounds still enforced (DB CHECK + Zod).

### 3.9 — Architecture & Design Patterns

| Pattern | Source | Why It Applies |
|---------|--------|----------------|
| Property graph on relational adjacency + bounded BFS | Cartography schema; Postgres docs | One ACID store (Neon); bounded traversal for dependencies, blast radius, attack paths. |
| Sync-tag / snapshot diff for drift | Cartography | Each discovery run stamps `lastSeenRunId`; diff → change events. |
| Transactional outbox + idempotent claim | BUILD.md has_dual_write / has_scheduled_work | DB → realtime bus / queue fan-out without lost events. |
| Hash-chained evidence ledger + HMAC checkpoint | AppMaster; DEV article | Tamper-evident evidence; verification job. |
| Policy-as-data + four-eyes approval | RESEARCH "Autonomous AI With Governance" | Agent proposals evaluated, validated, human-approved. |
| CQRS-lite read models | — | Dashboard counters/health computed by jobs into summary rows (cheap reads at scale). |
| Local HTTPS edge | https://dev.to/hugo__df/docker-compose-local-https-with-nginx-or-caddy-5emk | Caddy `tls internal` in compose. |

**Key Takeaways:**
- Derived intelligence (paths, impact, predictions) is recorded as evidence-linked artifacts, never silently recomputed, so every conclusion is reproducible.

---

## 4. Technology Stack Candidates

| Layer | Option A | Option B | Recommendation | Rationale |
|-------|----------|----------|----------------|-----------|
| Language | TypeScript 5.x on Node 22 LTS | Go | **TypeScript** | Next.js + Express both Node; shared Zod schemas. |
| Framework (web) | Next.js 16.4 App Router + Redux Toolkit 2.13 | Vite SPA | **Next.js 16.4** (hard constraint) | SSR public surfaces, nonce CSP; RTK Query for server state. |
| Framework (API) | Express 5.3 | Fastify | **Express 5.3** (hard constraint) | REST `/api/v1`. |
| ORM | Prisma 7.10 | Drizzle | **Prisma 7.10** (hard constraint) | Migrations + typed client. |
| Database | Neon | Self-hosted Postgres 17 | **Neon in production; postgres:17 container in the local stack** | Neon is wire-compatible Postgres; no Neon credentials locally (ADR). |
| Cache / bus / queue | Redis 7 + BullMQ 6 | RabbitMQ | **Redis 7 + BullMQ 6.3** (hard constraint) | Also rate limits and SSE fan-out (pub/sub). |
| Realtime | SSE + Redis pub/sub | Socket.IO + redis adapter | **SSE** | One-way streams; no sticky sessions; cookie auth; CSP-simple. |
| Graph viz | Bespoke SVG constellation | cytoscape.js | **Bespoke** | Template-faithful Ψ-core look; first-load JS < 400 KB. |
| Hosting/Infra | Docker Compose | Kubernetes | **Docker Compose** — Caddy edge, 2× API, worker, web, Redis, Postgres, Prometheus, Grafana | Local-stack deploy (Outcome C); K8s out of scope (ADR). |
| Auth | Email/password + DB sessions (argon2id) | OAuth/SSO | **Email/password + DB sessions**; agents/sources use hashed API keys | `auth_model` blank → safest self-contained default (ADR). |
| CI/CD | GitHub Actions | — | **GitHub Actions** | Service Floor. |
| Observability | prom-client + pino JSON | OpenTelemetry | **prom-client + pino** | Hard constraints. |
| Testing | Vitest 5 + Supertest + Playwright 1.64 + autocannon 8 | Jest | **Vitest/Playwright/autocannon** | Verification Gate + load test. |

---

## 5. Risk & Unknowns Register (with Threat Model)

| ID | Unknown / Risk | Severity | Mitigation | Status |
|----|----------------|----------|------------|--------|
| R1 | `scale: large — 50k+ concurrent` cannot be load-tested on one developer machine | high | Build to the large tier; load test at the highest stable concurrency the host supports per the BUILD.md methodology; report the gap as a red §9 verdict + residual, never retune targets. | open (residual) |
| R2 | Neon unreachable from the local stack | med | Postgres 17 container; env schema accepts Neon pooled + direct URLs; QUICKSTART documents switching. | resolved (ADR) |
| R3 | Stripe keys absent | med | Signed webhook fixtures + mock checkout path; live keys are a provisioning step. | resolved (ADR) |
| R4 | Graph traversal cost at large entity counts | med | Depth ≤ 6, indexed adjacency, per-request node budget, precomputed summaries; SPEC §9 p95 targets. | open |
| R5 | No LLM configured for the "Mirror Brain" | med | Deterministic explainable inference (graph statistics + rules) with explicit assumptions and confidence; pluggable provider interface; no data leaves the environment. | resolved (ADR) |
| R6 | HTTPS-only locally → self-signed CA trust | med | Caddy `tls internal`; CA trust install or documented per-browser trust in QUICKSTART. | open |
| R7 | Template `--sans` is Inter (anti-slop risk) | low | Template tokens copied verbatim; Manrope added as display face per RESEARCH typography; ADR. | resolved (ADR) |
| R8 | RESEARCH design said "Light only"; template and prose are dark | low | Precedence: template > RESEARCH → single dark theme from the template palette (PBR-1). ADR. | resolved (ADR) |

### Threat model (STRIDE over trust boundaries)

**Assets:** tenant system graph (infrastructure topology — high value to attackers), evidence ledger integrity, agent action authority, credentials/API keys, billing state.

**Actors:** anonymous internet user; authenticated tenant user (viewer / analyst / admin / owner); AI agent identity (API key); telemetry source (ingest key); Stripe (webhook); malicious user in another tenant; operator with DB access.

**Trust boundaries:** (B1) browser → Caddy edge (TLS); (B2) edge → web/api; (B3) api → Postgres/Redis; (B4) ingest source → `/api/v1/ingest`; (B5) agent → proposal APIs; (B6) Stripe → webhook; (B7) worker → DB (no user present).

| Threat (STRIDE) | Boundary | Control |
|---|---|---|
| S — credential stuffing / brute force | B1/B2 | argon2id; (ip,email) limiter before verify; breached-password screen; generic errors; session rotation on login. |
| S — forged webhook | B6 | Signature verified on raw body before any DB read; event-id UNIQUE. |
| S — stolen ingest/agent key | B4/B5 | Keys stored as SHA-256 hashes, prefix-identified, org + capability scoped, revocable, per-key rate limit. |
| T — evidence tampering | B3 | Per-org hash chain, append-only by app, HMAC checkpoints with key outside DB, scheduled verification surfaces breaks. |
| T — CSRF on cookie session | B1 | SameSite=Lax + double-submit CSRF token on mutations; Origin check. |
| R — agent/user denies action | B5 | Every proposal/approval/execution recorded in the evidence ledger with actor identity. |
| I — cross-tenant read (IDOR) | B2/B3 | Every query org-scoped in the service layer (invariant); 404 on foreign ids; same-tenant role checks on sub-resources; tests for both. |
| I — topology leakage via errors/logs | B2 | RFC 9457 errors without internals; pino redaction of secrets/cookies/keys. |
| D — floods / expensive graph queries | B1/B2 | Edge + per-user/per-IP Redis token buckets; cost-weighted limits on traversal; depth/node budgets; body limits. |
| D — job storms | B7 | BullMQ concurrency caps, DLQ, idempotent claims. |
| E — agent privilege escalation | B5 | Agents can only propose; execution needs policy pass + validation + human approval for risk ≥ high; approver ≠ proposer. |
| E — role escalation | B2 | Role changes org-scoped; owner-only grants; last owner cannot be demoted. |
| Injection via ingested data | B4 | Ingested text never executed or interpolated into policies; inference runs on typed fields only; output escaped by React. |

---

## 6. Research Gaps

- [x] Gap 1: Neon production credentials — unavailable; local Postgres stand-in (ADR), documented in QUICKSTART.
- [x] Gap 2: Stripe keys — unavailable; signed fixtures; provisioning documented.
- [x] Gap 3: Real telemetry connectors — out of scope for v1; generic authenticated ingest API + seeded demo environment (ADR).
- [x] Gap 4: LLM provider for the Mirror Brain — none specified; deterministic explainable engine (ADR).
- [x] Gap 5: 50k concurrent verification — hardware-bound; residual with measured headroom.
- [x] Gap 6: What is charged (`has_payments`) — resolved as tenant subscription plans (Explorer / Constellation / Sovereign) via Stripe Checkout (ADR).
- [x] Gap 7: Geographic use (`has_geo`) — resolved as asset site/region coordinates on a constellation geo view (ADR).

**Blocker?** No — every gap has a documented resolution that keeps the build self-contained.

---

## 7. Summary & Recommendation

STARDANCER is a multi-tenant system-intelligence platform: it ingests observations, builds a live relationship graph of the environment, detects change and drift, analyses attack paths / root cause / impact, issues explainable predictions with explicit assumptions, and routes AI-agent proposals through policy, validation and human approval, recording everything in a tamper-evident evidence ledger. The market (Wiz, JupiterOne, XM Cyber, Axonius, BloodHound) validates graph-centric exposure management; differentiation is the governed, evidence-first closed loop and sovereign deployment. All hard constraints are mutually compatible. The main honest gap is verifying 50k concurrent users on one machine — the architecture targets the large tier and the load-test shortfall will be reported, not hidden.

**Go / No-Go Decision:** `GO`

**Next Step:** → [ARCHITECTURE.md](./ARCHITECTURE.md)

---

## Pre-Build Review (Step 3b — 2026-10-12)

| ID | Field | Conflict / gap | Resolution applied |
|----|-------|----------------|--------------------|
| PBR-1 | Design Language › Theme / palette | "Light only" + blue `#2665ed` primary contradicts the prose ("coral-accented dark interface") and the dark DESIGN-TEMPLATE.html | Theme set to dark only; template `:root` tokens canonical; RESEARCH light/dark Tailwind palette superseded. |
| PBR-2 | `domain` | "Productivity" contradicts a security/operations intelligence product | domain → "Security & Operations Intelligence"; archetype stays `saas`. |
| PBR-3 | Success Metrics › Graph Freshness | "Near-real-time" has no threshold | p95 observation-accepted → graph-updated ≤ 60 s, exported as a Prometheus histogram. |
| PBR-4 | Success Metrics › Availability, RTO | "Enterprise-grade" / "Defined" are not measurable | 99.9 % monthly SLO; RTO ≤ 1 h, RPO ≤ 5 min. |
| PBR-5 | Integrations | Threat Watch, CyberOps, Synapse/Pulse, ALPHA ZER0, EDR/SIEM named in workflows but no integration list | v1 ships one authenticated generic ingest API whose `sourceKind` enumerates these producers; dedicated connectors are deferred (Non-Goal: "build every possible integration before the core System Graph works"). |
| PBR-6 | domain_signals `has_payments`, `has_geo` | No workflow uses them | `has_payments` → tenant subscription billing (Explorer / Constellation / Sovereign) via Stripe; `has_geo` → site/region coordinates on entities with a geo view of the constellation. |
| PBR-7 | Non-Goal "Display fabricated telemetry as live" vs demo data | A fresh tenant has no telemetry | Demo environment data is generated only on explicit request, stored with `sourceKind: simulated`, and every surface that shows it carries a visible SIMULATED badge. |
| PBR-8 | Workflow 11 (Cryptographic Trust via ALPHA ZER0) | External system unavailable | v1 provides in-platform trust: SHA-256 hash-chained evidence, HMAC-signed checkpoints and signed agent action records; an ALPHA ZER0 verification adapter is deferred. |
