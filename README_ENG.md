# **Project AgentWhite — Bug Bounty Research Environment for an AI Team**

[На русском](./README.md)

## **Overview**

`AgentWhite` is a multi-language monorepo for authorized bug bounty research, where AI operates not as a chat assistant but as a managed team of specialized agents. The central technical piece is a Rust library and CLI (`agentwhite-core` / `agentwhite-cli`) implementing a **policy gateway**: mandatory enforcement of program rules at the network level. The research log (`research-log.md`) is the persistent chronological memory of the entire campaign — a single source of truth about what has been tested, what has been ruled out, and which hypotheses are active.

*The codebase is proprietary. Specific operator runbooks, prompt routing, internal heuristics, and program-specific workflows are intentionally not disclosed.*

*Github Langbar is simulated using [gh-lang-mock](https://github.com/Artemonim/gh-lang-mock) based on the language distribution of the source repository.*

## **Technology Stack**

*   **Rust core**: workspace `crates/*`, `resolver = "3"`, edition 2024, Rust 1.85+, `unsafe_code = "forbid"`; `agentwhite-core` (policy engine, HTTP proxy, audit) + `agentwhite-cli` (`clap`-based CLI: `preflight`, `request`, `proxy`, `tool`)
*   **TypeScript tooling**: `pnpm` workspace, Node `>=24.14`, ESLint + `typescript-eslint`; strict ESM, ES2024, `verbatimModuleSyntax`; typed web and tooling integrations
*   **Python subsystem**: Python 3.13+, Ruff, MyPy (strict), pytest; analytics and supporting automation
*   **PowerShell orchestration**: `run.ps1` → `tools/ci/build.ps1`; CI stage orchestrator, recon smoke, and JSON report
*   **Go utilities**: module `agentwhite.local/go`; point utilities
*   **External security tooling** (`tools/third_party/`):
    *   recon / OSINT: `subfinder`, `gau`, `asnmap`
    *   target-side / HTTP / fuzzing: `httpx`, `katana`, `ffuf`, `nuclei`, `naabu`
    *   orchestration-friendly runtime: `BBOT`, Docker-based flows; `nuclei-templates` submodule
*   **Cursor Browser MCP**: passive surface observation — the agent sees pages as a user would, collects navigation artifacts without sending security-specific requests
*   **Operator rules**: `AGENTS.md`, `Techlead.mdc`, `SubagentRules.md` — based on [AgentCompass](https://github.com/Artemonim/AgentCompass)
*   **CI/CD**: local PowerShell orchestrator `run.ps1`, stage scripts in `tools/ci/`, machine-readable JSON report

## **Key Challenges and Solutions**

*   **Enforcing program rules under automation:** Automated recon easily crosses scope boundaries, violates rate limits, or exposes operator identity to the target. The solution is a Rust policy gateway: every request passes through a per-program `policy.toml` (scope CIDR/domains, rate limits, required operator headers, VPN markers); proxy-capable tools are routed through it unconditionally. A policy mismatch is a rejection before the packet leaves the machine.
*   **Two-mode access to the research surface:** Not every research step requires a controlled outbound request. The solution is a clear split: **Cursor Browser MCP** for surface observation (the agent sees what a user sees, collects navigation artifacts without generating additional traffic) and the **agentwhite gateway** for specific security requests that pass through policy enforcement. The browser is the team's passive eyes; the gateway is its controlled hands.
*   **Accumulating research memory across sessions:** Agents start with zero memory, and a campaign can span weeks. The solution is `research-log.md` as persistent chronological memory: each cycle records tested hypotheses, killed vectors, active priors, and key decisions. New subagents receive log excerpts in their prompt; log saturation is the signal to design the next `TODO.md`.
*   **Artifact discipline under sensitive-data conditions:** Raw scan results and intermediate findings must not end up in git or the public layer. The solution is strict separation: `.temporary/<program>/` (gitignored, ephemeral), `evidence/` (sanitized, git-safe), `reports/` (disclosure templates only), `.enforcer/` (gateway audit log, gitignored).

### **Architectural and Technical Features**

1.  **Policy Gateway (Rust):**
    *   `agentwhite-core` loads a per-program `policy.toml`: scope CIDR/domains, rate-limit thresholds, required operator identification headers, VPN markers.
    *   Rate limiting is backed by a disk-based ledger — limit state survives CLI restarts.
    *   HTTP proxy mode: httpx, nuclei, katana, and ffuf are routed through the gateway with header injection and scope enforcement; passthrough tools (gau, asnmap) receive environment variables without direct proxying.
    *   `agentwhite-cli` exposes subcommands `preflight` (dry-run policy and scope check), `request` (single HTTP request through the gateway), `proxy` (local proxy server), and `tool <name>` (launcher that injects proxy flags and redirects output to `.temporary/<program>/`).
    *   Audit logs are written to `.enforcer/gateway/<program>/` — outside the public layer of the repository.

2.  **Research Log and Campaign State Management:**
    *   `programs/<name>/research-log.md` — chronological journal of the entire campaign: tested hypotheses, killed vectors, active priors, key operator decisions, and phase-level artifacts.
    *   Log excerpts are included in the prompt of every new subagent to compensate for zero starting memory.
    *   Log saturation serves as the signal to design a new `TODO.md`; the next TODO is built from documented state, not from the operator's memory.
    *   `research-graph` (`graph.yaml` + Mermaid views) is a machine-readable supplemental artifact under development for structured representation of assets, vectors, and findings.

3.  **Agent Operating Contract:**
    *   `AGENTS.md` — formal contract for agents: repository mission, explicit prohibitions, tool map, and CI contract.
    *   `Techlead.mdc` — main agent process: pre-production → task clusters → subagent calls → Verifier → Verifier Pro. When blockers arise or new decisions are needed, Techlead escalates to the operator.
    *   `SubagentRules.md` — subagent contract: zero memory at start, scope-first priority, handoff report structure, outcome classification (`finding`, `no finding`, `needs-human-repro`).
    *   Rules are based on the public framework [AgentCompass](https://github.com/Artemonim/AgentCompass).
    *   Agent roster: **OSINT Scout**, **OSINT Pro**, **Scope Sentinel**, **Surface Mapper**, **Vector Ranker**, **Field Operator**, **Traffic Triage** — each with a specific role and position in the pipeline.
    *   Working cycle: Architect designs `TODO.md` → Techlead distributes into clusters and calls subagents → every step is written to `research-log` → when the TODO is saturated, the next one is designed.

4.  **Multi-language Monorepo:**
    *   **Rust** — policy enforcement, core CLI, systems primitives; chosen for a reliable network layer without runtime overhead.
    *   **TypeScript** — typed web integrations and tool contracts; strict ESM, `verbatimModuleSyntax`, ES2024.
    *   **Python** — analytics and supporting automation with fast ecosystem integration.
    *   **PowerShell** — CI orchestrator and operator layer in a Windows environment.
    *   **Go** — point utilities where compact, statically linked delivery is the right fit.

5.  **Recon Smoke CI and Local Orchestrator:**
    *   `run.ps1` — single entrypoint with flags `-Fast` (smoke only), `-SkipTests`, `-SkipReconSmoke`.
    *   `Invoke-ReconToolsSmoke.ps1` starts a local HTTP server (`sandboxes/recon-smoke/www`) and runs each tool through a control scenario: httpx, katana, ffuf, nuclei (including template validation from the submodule), subfinder/gau/asnmap, optionally naabu and Docker/BBOT.
    *   Language tests are combined in a single run: `cargo test --workspace`, `pnpm test`, pytest.
    *   The final JSON report (`.ci_cache/report.json`) records stage statuses and durations for repeat analysis.

## **Demonstrated Skills**

*   Designing a network enforcement layer in `Rust`: proxy, policy engine, disk-based rate limiter, audit trail
*   Multi-language monorepo: `Rust` + `TypeScript` + `Python` + `PowerShell` + `Go`
*   Designing a multi-tier AI team: Techlead, specialized subagents, handoff contract, agent roster
*   Research memory accumulation via research-log and per-program TODO-driven process
*   Policy-aware automation for authorized bug bounty: scope, rate limits, audit, VPN markers
*   Two-mode surface access: Cursor Browser MCP (passive observation) + agentwhite gateway (controlled requests)
*   Curated external security tool catalog with smoke verification against a local server
*   Artifact discipline and data hygiene under sensitive-context conditions
*   PowerShell CI/CD orchestration with machine-readable reporting
