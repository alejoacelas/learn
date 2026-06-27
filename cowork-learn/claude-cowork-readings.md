# Claude Cowork Technical Reading Packet

Last researched: 2026-06-24

Focus: gears-level understanding of Claude Cowork, not a help-center tour. Support pages are included only when they encode hard product boundaries, admin controls, or runtime behavior.

Note on excerpts: you originally asked for 280-character verbatim abstracts. I kept verbatim quotes shorter for copyright safety and used my own technical abstracts.

## Reading Order

### 1. How we contain Claude across products

Source: https://www.anthropic.com/engineering/how-we-contain-claude

Why this is first: this is the best official technical framing. It compares claude.ai containers, Claude Code's human-in-the-loop sandbox, and Cowork's local VM model.

Quote: "The VM has its own Linux kernel, its own filesystem, and its own process table."

Technical abstract: Anthropic's containment model treats Cowork as the "local VM" pattern. The original Cowork architecture ran the whole agent loop inside a full VM using Apple Virtualization.framework on macOS and HCS on Windows. The user's selected workspace and `.claude` folder were mounted; host credentials stayed in the host keychain. Anthropic later moved the agent loop outside the VM for reliability while keeping code execution inside. Local MCP servers also moved outside the VM because some need host-local resources.

What to extract:

- The hard security boundary is the mounted workspace plus VM/hypervisor boundary, not a user reviewing shell commands.
- Cowork was designed for knowledge workers who should not be expected to audit bash.
- The architecture deliberately reduces host filesystem reachability.
- Isolation creates an observability tradeoff: EDR cannot see inside the guest VM.
- Remote tools are not just code supply-chain risk; they are prompt-injection channels whose behavior can change after approval.

### 2. Inside Claude Cowork: How Anthropic Runs Claude Code in a Local VM on Your Mac

Source: https://pvieito.com/2026/01/inside-claude-cowork

Why read it: independent reverse engineering of the early macOS implementation. This gives the concrete mechanics: VM bundle, session isolation, VirtioFS, path rewriting, local proxy allowlist, and MCP pass-through.

Quote: "Inside this VM, it executes Claude Code CLI within a multi-layered sandbox."

Technical abstract: Cowork is effectively Claude Desktop orchestrating Claude Code inside a local Linux VM. The investigated build shared one VM across multiple conversations, but each session had its own isolated user/session directory. User folders were mounted with VirtioFS and exposed under session-specific `/sessions/.../mnt/...` paths. Claude Desktop rewrote VM paths in the UI back into user-friendly host paths. Network traffic from the VM went through a local allowlist proxy; arbitrary direct internet access was blocked.

What to extract:

- Cowork reuses Claude Code as the execution harness.
- The VM is not per prompt; sessions can share VM infrastructure while preserving per-session boundaries.
- Path translation is a product layer: Claude sees VM paths, users see host paths.
- Host/guest communication uses multiple channels: filesystem mounts, pipes, sockets, and RPC-like control messages.
- MCP can be proxied through Desktop rather than running wholly inside the VM.

### 3. Inside Claude Cowork: How Anthropic's Autonomous Agent Actually Works

Source: https://blog.pluto.security/p/inside-claude-cowork-how-anthropics

Why read it: security reverse engineering from a different angle. Useful for understanding where the VM stops helping: Chrome/browser control, computer use, feature flags, and host-side bridges.

Quote: "Chrome browser control runs outside the VM sandbox"

Technical abstract: Pluto's teardown emphasizes the split between VM-contained execution and host-side capabilities. The VM boundary may be strong for shell/code execution, but browser automation via Claude in Chrome runs in the user's real browser profile. That means host cookies, real network context, same Chrome profile, and visual tab-group separation rather than true profile isolation. The post also claims remote feature flags shape Cowork behavior, and that network controls observed in the VM were implemented above ordinary in-guest firewall rules.

What to extract:

- The VM is not a universal sandbox for every Cowork capability.
- Browser automation is a distinct path with its own trust boundary.
- Same-profile Chrome operation matters because ambient cookies become agent authority.
- Tab-group isolation is visibility, not credential isolation.
- A "local agent" threat model has to model host-side bridges, extensions, native messaging, and feature flags.

### 4. patrickjaja/claude-cowork-service

Source: https://github.com/patrickjaja/claude-cowork-service

Why read it: unofficial Linux-compatible reimplementation of the Cowork backend. This is useful because protocol-compatible projects reveal what Claude Desktop expects from a Cowork service.

Quote: "length-prefixed JSON-over-Unix-socket protocol"

Technical abstract: The project implements a length-prefixed JSON-over-Unix-socket protocol that Claude Desktop can speak to. It describes a service layer between Claude Desktop and execution backends. On macOS/Windows the model is Desktop -> cowork service -> Apple VM/Hyper-V VM -> SDK daemon over vsock. The Linux project can run native host execution or experimental QEMU/KVM for parity. It also documents path remapping from `/sessions/<name>/mnt/...` into a local session store with symlinked mount points.

What to extract:

- Cowork has a daemon/service boundary, not just an Electron UI calling shell commands.
- The Desktop app expects structured session lifecycle operations over a socket-like protocol.
- The VM backend can be thought of as an execution provider behind that protocol.
- Claude Code streaming output and MCP control request/response messages are proxied back to Desktop.
- Path remapping is a compatibility contract between Desktop assumptions and backend filesystem layout.

### 5. Claude Cowork desktop architecture overview

Source: https://support.claude.com/en/articles/14479288-claude-cowork-desktop-architecture-overview

Why this support page stays: it is the concise official state of the current architecture after Anthropic moved parts out of the VM.

Quote: "Claude Cowork uses two execution environments"

Technical abstract: Current official docs describe two environments: the agent loop runs natively on the device, while code/shell execution runs in an isolated Linux VM. Native agent-loop responsibilities include conversation handling, file reads/writes in connected folders, web fetches, and local plugin MCP servers. The VM enforces network egress filtering, syscall restrictions, and per-session user isolation. Device-level controls can disable local MCP servers and desktop extensions.

What to extract:

- The current architecture is host-mode agent loop plus VM code execution.
- File operations and local plugin MCPs may be host-side, permission-gated application-layer actions.
- If the VM fails, Cowork can continue file/web work but shell/code execution reports workspace unavailable.
- Audit logs, Compliance API, and data exports do not currently capture Cowork activity.
- EDR cannot inspect activity inside the VM by design.

### 6. Enterprise configuration for Claude Desktop

Source: https://support.claude.com/en/articles/12622667-enterprise-configuration-for-claude-desktop

Why this support page stays: it encodes the device-policy keys that define the enforceable knobs.

Quote: "Administrators on Team or Enterprise plans can control Claude Desktop through system policies."

Technical abstract: Claude Desktop can be controlled via MDM profiles on macOS and Group Policy/Intune on Windows. Important keys include `allowedWorkspaceFolders`, `forceLoginOrgUUID`, `isDesktopExtensionEnabled`, `isDesktopExtensionDirectoryEnabled`, `isLocalDevMcpEnabled`, `isClaudeCodeForDesktopEnabled`, and `secureVmFeaturesEnabled`. These are device policy controls, separate from in-app organization settings.

What to extract:

- Organization settings are not the only control plane; managed-device policy is crucial.
- `allowedWorkspaceFolders` scopes what users can mount to Cowork.
- `forceLoginOrgUUID` prevents accidental/personal-account use.
- Local MCP and desktop extension controls are separate switches.
- `secureVmFeaturesEnabled` is effectively a Cowork/VM feature gate in Desktop policy.

### 7. Monitor Claude Cowork activity with OpenTelemetry

Source: https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry

Why this support page stays: it explains the telemetry stream that partially compensates for missing audit/Compliance API coverage.

Quote: "A shared `prompt.id` attribute links every event"

Technical abstract: Cowork can stream structured events to an OTLP collector. Event classes include user prompt text, tool/MCP invocations, file paths touched, skills/plugins used, human approval decisions, API request metadata, token counts, estimated cost, errors, and timings. This is not enabled by default and may leak sensitive content into SIEM unless redacted downstream.

What to extract:

- OTel is the real monitoring interface for Cowork today.
- The event graph can be reconstructed around `prompt.id`.
- It can show tool parameters and file paths, but not necessarily full guest VM internal process details.
- This is observability, not prevention.
- Prompt and tool-parameter logging need their own retention/redaction policy.

### 8. How to Secure Claude Cowork: Enterprise Deployment Guide

Source: https://generalanalysis.com/guides/how-to-secure-claude-cowork

Why read it: a useful technical control-plane decomposition, especially the four-path model.

Quote: "The most important distinction is between four paths"

Technical abstract: General Analysis breaks Cowork into local files, code execution, browser use, and computer use. That decomposition is more precise than "Cowork has access." Each path has a different enforcement layer and failure mode: folder grants/deletion prompts for files, VM/network egress for code, Chrome permissions/profile/network controls for browser, and per-app approval/app blocklists for computer use.

What to extract:

- Security analysis should be path-specific, not product-wide.
- Code execution and browser use have different network surfaces.
- Computer use operates on the real desktop and should be treated separately from VM code execution.
- A proxy, browser policy, and MCP governance may matter more than chat policy.
- Cowork UX hides the task graph; controls need to observe downstream actions, not only initial prompts.

### 9. Security Guidance for Claude Cowork and Risks

Source: https://generalanalysis.com/guides/security-guidance-for-claude-cowork-and-risks

Why read it: a concrete risk taxonomy for rollout and monitoring.

Quote: "Cowork should be treated as a privileged desktop agent."

Technical abstract: This guide frames Cowork as endpoint agent infrastructure. Risks include broad folder grants, browser prompt injection, computer-use actions in approved apps, plugin/MCP supply chain, connector scopes, network/web egress gaps, scheduled/delegated work, and incomplete audit coverage. The useful part is less the vendor pitch and more the surface inventory.

What to extract:

- Treat Cowork like an endpoint-resident automation agent.
- Browser and local file prompt injection are first-class risks.
- Scheduled tasks are unattended agent runs, not reminders.
- Connector tenant drift and account scope matter.
- High-assurance posture requires managed devices, scoped folders, reviewed plugins, and telemetry.

### 10. Reverse-Engineering Claude Code: A Deep Dive into Anthropic's AI-Powered CLI

Source: https://sathwick.xyz/blog/claude-code

Why read it: Cowork inherits or parallels parts of Claude Code's agent harness, so Claude Code internals help explain Cowork's execution core.

Quote: "a multi-layered permission system"

Technical abstract: This is an unofficial deep dive into Claude Code's TypeScript/Bun/React terminal architecture, tool system, permission layers, slash commands, MCP integration, context management, session persistence, subagents, hooks, and telemetry. Treat the details as reverse-engineered, but it is useful for understanding what a Claude Code-style harness brings into Cowork: structured tools, permissions, streaming, agent loops, task decomposition, and extensibility.

What to extract:

- Claude Code is not a thin terminal wrapper; it is a stateful agent runtime.
- It has tool schemas, permission gates, context compaction, persistent sessions, hooks, plugins, skills, and MCP.
- If Cowork executes Claude Code-like workflows, the hidden task graph can be substantial.
- Claude Code's developer affordances are explicit in the terminal; Cowork abstracts many of them behind Desktop UX.
- Comparing the two clarifies why Cowork needs stronger default containment for non-developer users.

## Gears-Level Model

### 1. Product shell

- Claude Desktop is the local application shell. It owns the Cowork UI, task list, progress display, permission prompts, local settings, organization/device-policy consumption, and bridges to local services.
- Cowork tasks are not just chat messages. A user prompt becomes a task graph: plan, choose tools, request permissions, read/write files, possibly run code, possibly call connectors/MCPs, possibly use browser/computer control, then produce artifacts or messages.
- The UI smooths over several substrates: host-native file operations, local service/daemon calls, VM code execution, cloud model calls, remote MCP/connectors, local MCP servers, and browser/computer control.

### 2. Agent loop placement

- Early architecture: evidence from Anthropic and reverse engineering indicates the whole agent loop originally ran inside the VM.
- Current architecture: Anthropic says the agent loop now runs natively on the device while shell/code execution remains inside the isolated Linux VM.
- Practical effect: Cowork can still converse, read/write connected folders, use web fetches, and use local plugin MCP servers even if VM startup fails.
- Security effect: code execution remains strongly isolated, but some agent actions are application-layer permission checks on the host rather than guest-VM operations.

### 3. VM execution path

- Shell commands and Claude-written code run inside a dedicated Linux VM.
- macOS uses Apple Virtualization.framework; Windows uses Hyper-V/HCS-style virtualization.
- The guest has its own kernel, filesystem, process table, and per-session users.
- Selected workspace folders are mounted into the guest; the rest of the host filesystem is not visible by default.
- Host credentials should stay in host keychain/credential stores rather than entering the guest.
- VM network egress is filtered. The observed early implementation allowed Anthropic API and package registries while blocking arbitrary domains.
- VM isolation reduces blast radius but also blocks host EDR visibility into guest internals.

### 4. Session and filesystem model

- Cowork sessions are represented with session-specific paths such as `/sessions/<session>/mnt/<folder>`.
- User-selected host folders are mounted into the VM, observed via VirtioFS on macOS.
- Desktop translates guest paths into host-friendly paths in UI so the user sees `~/Downloads/...` rather than internal `/sessions/...` paths.
- Multiple conversations may share a VM instance but have isolated session users/directories.
- Inactive sessions may be depersonalized or ownership-shifted to reduce cross-session visibility.
- Folder grants are the main filesystem capability boundary. Broad grants are broad authority.

### 5. Service/protocol boundary

- Claude Desktop talks to a Cowork backend/service rather than embedding every execution primitive directly in the renderer.
- The unofficial Linux implementation describes length-prefixed JSON over a Unix socket and backends for host-native or VM execution.
- macOS/Windows shape appears to be: Desktop -> cowork service -> VM -> SDK daemon, with vsock-like communication inside VM-backed mode.
- Claude Code streaming JSON/output can be proxied back to Desktop so UI can show progress incrementally.
- MCP control requests/responses can ride through these streams or be proxied by Desktop's session manager.

### 6. MCP, plugins, and connectors

- Local MCP servers and desktop extensions are host-side software. They run with the permissions of ordinary local programs unless blocked by policy.
- Remote MCP/custom connectors are cloud-reached. Even in Desktop/Cowork, they are reached from Anthropic cloud infrastructure, not from the local machine.
- Plugins can bundle skills, connectors, hooks, sub-agents, and local MCPs. Hooks and sub-agents are Cowork-specific affordances in the Claude app ecosystem.
- Organization plugin marketplaces centralize distribution, but plugin code and MCP tools still expand the action surface.
- Local MCP is auditable and pinnable, but can access local resources. Remote MCP may change behavior after approval and is a continuing prompt-injection/tool-output risk.

### 7. Browser and computer-use paths

- Browser use is not the same as VM code execution. Claude in Chrome operates through the host browser and extension/native messaging bridge.
- If the browser path uses the user's real Chrome profile, it inherits cookies, sessions, identity, extensions, and network context.
- Tab-group separation is visibility and workflow separation, not a credential boundary.
- Computer use can click, type, take screenshots, open apps, and interact with the real desktop after per-app approval.
- Anthropic explicitly says computer use has no sandbox between Claude and approved desktop apps. This is the sharpest distinction from VM-contained code execution.

### 8. Network and egress surfaces

- VM code execution egress can be constrained by organization network settings and VM/local proxy enforcement.
- Web fetch/search, remote MCP/connectors, Claude in Chrome, and computer use do not all share the same egress path.
- Therefore "network is disabled" does not mean "the agent cannot receive untrusted web/tool content" or "no data can leave through another approved channel."
- An exfiltration can occur through an allowed service/API if the agent has both read access and an approved write path.
- Controls need to be attached to each egress path: VM network, browser profile/network, connector scopes, MCP tools, upload/send actions, and OTel visibility.

### 9. Approvals and safety gates

- Cowork exposes approval modes like ask-before-acting and act-without-asking.
- Permanent file deletion still requires explicit approval.
- Per-app approval gates computer use, but once an app is approved, visible data and actions in that app become part of the work surface.
- Model and classifier defenses are useful, but Anthropic's own engineering post argues hard environmental boundaries are the robust control when prompt injection or direct malicious instructions are possible.
- The central risk condition is: read untrusted content plus permission to take consequential actions.

### 10. Scheduling and Dispatch

- Scheduled Cowork tasks are unattended Cowork sessions triggered by cadence or on demand.
- They only run while Claude Desktop is open and the computer is awake.
- Dispatch/persistent-thread flows let users delegate from mobile while the desktop machine does the work.
- Dispatch can route development work to Claude Code and knowledge work to Cowork.
- Mobile becomes a remote control for desktop resources: local files, connectors, plugins, browser/computer permissions already configured on Desktop.

### 11. Admin/control planes

- Organization settings: enable/disable Cowork, configure capabilities/network settings, manage plugins/marketplaces, configure OTel endpoint, manage connectors, apply role/group controls on Enterprise.
- Device policy: restrict allowed workspace folders, force org login, disable local MCP, disable desktop extensions, disable extension directory, control auto-updates, gate Desktop Claude Code/Cowork features.
- Browser policy: needed if Claude in Chrome or browser use is allowed; controls profile isolation, site allow/block lists, extension availability, upload/DLP behavior.
- Connector/admin auth: can make connectors available org-wide, but user-level permissions and third-party scopes still define what data is reachable.
- Telemetry: OTel is currently the detailed Cowork event stream; standard audit logs/Compliance API/data exports do not cover Cowork activity according to current docs.

## Cowork vs Claude Code: Technical Difference

- Claude Code is primarily a terminal/IDE agent runtime. It assumes a developer can inspect commands, understand repository context, and manage local shell authority.
- Cowork is a Desktop task agent for knowledge workers. It assumes users may not understand shell-level consequences, so code execution is pushed into a local VM boundary.
- Claude Code's native surface is explicit: terminal output, command approvals, project files, repo settings, hooks, MCP config, `CLAUDE.md`, and managed settings.
- Cowork's native surface is abstracted: tasks, progress indicators, connected folders, permissions, plugins, projects, scheduled tasks, Dispatch, and generated deliverables.
- Claude Code has a granular layered settings model: managed, CLI args, local, project, user. Managed settings are hard policy.
- Cowork has several control planes, but fewer repo-like local policy layers: org toggle, Enterprise roles, plugin marketplaces, connector controls, OTel, Desktop MDM keys, browser controls, and per-user folder/app grants.
- Claude Code mostly operates in the developer's existing local environment unless sandboxing/policies are configured. Cowork's shell/code path is VM-contained by default.
- Cowork has more non-code authority surfaces: local document folders, Office-style deliverables, browser sessions, desktop apps, connectors, plugins, and scheduled recurring knowledge-work tasks.
- Claude Code is usually foreground developer work. Cowork is designed for delegation and can be background/unattended through scheduled tasks or Dispatch, increasing the need for after-the-fact telemetry and external guardrails.

## The Short Version

- Cowork is best understood as a local desktop agent orchestrator with multiple execution paths, not as "Claude Code but with a GUI."
- The VM is real and important, but it mainly contains shell/code execution. It does not automatically contain browser control, computer use, local MCP servers, remote connectors, or host-side file operations.
- The real security model is path-specific: files, VM code, browser, computer use, MCP, connectors, plugins, scheduling, and mobile Dispatch each have different authority and controls.
- The most important design choice is environmental containment over user shell-command judgment.
- The most important residual weakness is that useful agents need to read untrusted content and take useful actions, which creates prompt-injection-to-action paths.
- The most important admin reality is split control: organization settings, managed-device policy, browser policy, connector governance, plugin marketplace governance, and OTel all matter.

## Open Questions To Track

- How much of the early reverse-engineered VM behavior still applies after the host-mode agent-loop shift?
- Whether Anthropic will expose richer first-party live enforcement beyond OTel monitoring.
- Whether Cowork activity enters standard audit logs, Compliance API, or data exports.
- Whether Chrome/browser use gets stronger profile-level isolation by default.
- Whether Team plans get granular Cowork controls comparable to Enterprise roles/groups.
- Whether local MCP/plugin supply-chain controls become more declarative and centrally enforced.
