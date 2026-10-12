# Domain Entities Redesign: Method

Oct 12, 2026 · @Andrés

## 1. Why redesign the domain entities

Our entity model mixes industry-standard concepts with company-specific ones, and that blocks a shared, searchable catalog.

- Technical entities (component, pipeline, release) are common across the industry
- Management entities (project, team, **application**) are specific to our company and hard for newcomers to read
- Many entities have no clear lifecycle or way to decommission them, so obsolete entities pile up
- Goal: a clear, governed model that adopts standards where they fit, stays proprietary only where it adds value, and gives every entity a defined lifecycle

## 2. Process in eight steps

The redesign runs as a straight sequence, each step feeding the next.

1. Frame the purpose: competency questions the model must answer
2. Inventory the current entities (as-is), including lifecycle and ownership
3. Layer them: technical, delivery, organization
4. Deconstruct "application"
5. Map to industry standards and decide standard vs proprietary
6. Define the lifecycle and decommissioning path of every entity
7. Design the target model and pilot it on real data
8. Migrate and govern: old-to-new mapping, versioning, stewards

## 3. Frame the purpose and take inventory

Start from the questions the model must answer, then document what exists today.

- Write 15-20 competency questions, e.g. "which components break if this database changes?"
- For each of the 30-50 main entities record: one-sentence definition, users, source systems and IDs, synonyms and homonyms, current lifecycle and any decommissioning path, owner, relationships, known quality problems

## 4. Layer the entities

Standards are strong in the technical and delivery layers and weak in the organizational one, so each layer gets its own approach.

| Layer | Examples | Standards coverage | Default stance |
| --- | --- | --- | --- |
| Technical artifacts | component, method, database, commit, pull request | Strong | Adopt the standard |
| Delivery and runtime | pipeline, stage, release, change, deployment | Good | Standard core plus extensions |
| Organization and management | team, responsible, project, application | Weak | Mostly proprietary, with a documented mapping |

## 5. The "application" case

"Application" is an overloaded management concept, so it is deconstructed before it is mapped.

- Ask: what questions does it answer, who uses it, what decisions depend on it, what does it contain?
- Likely finding: it bundles two or three ideas, such as an ownership or funding unit, a business capability, and a grouping of components
- If so, split it, then compare each part with its closest standard counterpart
- Candidates: ServiceNow CSDM *Business Application*, ArchiMate *Application Component* and *Business Capability*, Backstage *System* and *Domain*
- Possible outcomes: rename, keep with a clear glossary entry, or split into separate entities

## 6. Map to industry standards

Each of our entities is placed against the closest concept in the reference models, with an honest match type.

| Area | Reference models |
| --- | --- |
| Developer catalogs | Backstage (Component, API, Resource, System, Domain, Group, User) |
| Architecture and IT management | ArchiMate/TOGAF, ServiceNow CSDM, ITIL |
| Delivery and supply chain | CDEvents, SLSA, SPDX, CycloneDX |
| ALM and source control | OSLC, Git/GitHub/GitLab object models |
| Runtime and interfaces | OpenTelemetry resource conventions, OpenAPI, AsyncAPI |

Crosswalk per entity: closest standard concept, match type (exact, close, broader, narrower, related, none), and semantic differences.

## 7. Standard or proprietary: the scorecard

Every entity is scored on six criteria and the decision is recorded in a short decision record.

| Criterion | Favors standard | Favors proprietary |
| --- | --- | --- |
| Semantic fit | Match is exact or close | Standard distorts our meaning |
| Ecosystem leverage | Tools and vendors speak it natively | No tooling benefit |
| Differentiation | Commodity concept (commit, PR) | Reflects how our company is organized |
| Standard stability | Mature, widely adopted | Moving or fragmented |
| Comprehension | Familiar to new hires | Standard term would confuse our staff |
| Migration cost | Low | High with little gain |

## 8. Lifecycle for every entity

Every entity must declare how it is created, changed, deprecated and retired, so obsolescence is designed out rather than cleaned up later.

- Entity contract: an accountable owner, defined states (proposed, active, deprecated, retired, archived), transition rules and a decommissioning path
- Entities with no natural end, such as a commit, declare that explicitly
- Enforce with signals: last-seen activity, automatic staleness flags, periodic owner attestation, dependency checks before retirement
- A missing owner or lifecycle state lowers the entity's quality score
- Start with the three or four entities where obsolescence hurts most

## 9. Expected outcome: the hybrid path

Most entities will not be fully standard or fully proprietary, so the target is a standard core with our own extensions.

- Technical artifacts: adopt the standard as is
- Delivery: adopt standard events and vocabulary, add extension attributes
- Management layer: keep our concepts where they add value, but give them a clearer name, a glossary definition and a published mapping to the nearest standard
- Across all layers: stable IDs, a source-ID crosswalk, and governed core schemas with extension points, plus an owner, lifecycle states and a decommissioning path for each entity

## 10. Timeline and next steps

&#91;embedded content: redesign plan · 6 workstreams over 7 weeks\]

The standards and application decisions land in weeks 3-5, so the pilot in weeks 5-7 tests them on real data.

Next steps:

- Agree the 15-20 competency questions
- Set up the inventory and crosswalk templates
- Name an owner for each entity
