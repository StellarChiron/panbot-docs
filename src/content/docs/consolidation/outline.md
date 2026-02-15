# Architecture Documentation Consolidation Outline (Issue #10)

## Goal
Consolidate fragmented architecture documentation into a single, cohesive source of truth within `panbot-docs`.

## 1. Unified System Overview
- **High-Level Diagram**: Mermaid.js diagram of all components (Backend, Web, Desktop, Telephony, Printer).
- **Core Principles**: Multi-tenancy, Real-time first, Privacy-safe.
- **Component Roles**: Define the responsibility of each repo.

## 2. Shared Data Models & Contracts
- **OpenAPI Strategy**: How we use shared clients and avoid shape mismatches (#5, #82).
- **Real-time Event Schema**: Centrifugo channel patterns and JSON payloads.
- **Auth Contract**: Logto (Web/Owner) vs. Device/PIN (Staff/Desktop).

## 3. Deployment & CI/CD Patterns
- **Staging/Per-PR namespaces**: E2E infrastructure design (#87, #88).
- **Forensics/Observability**: RBAC for kubectl/helm forensics (#110).

## 4. Integration Guides
- **Payment Flow**: Stripe/Payment intents.
- **Telephony Flow**: LiveKit Agents + VAD + TTS.
- **Printer System**: Cloud-to-Local bridge.

## Proposed Action Plan
1. **Inventory**: Gather `ARCHITECTURE.md` from `panbot`, `panbot-web`, and `panbot-docs/plans`.
2. **Standardization**: Use consistent Mermaid diagrams and markdown headers.
3. **Migration**: Move technical details from `plans/` into `src/content/docs/architecture/`.
4. **Maintenance**: Establish a "docs-first" rule for new architecture changes.
