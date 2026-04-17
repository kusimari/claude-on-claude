# Claude Code Codebase Analysis Plan

**Goal**: Comprehensive analysis of the Claude Code CLI codebase to understand:
- Launch sequence and initialization
- Main REPL loop architecture and execution
- Background processes and lifecycle management
- Alternative execution modes

## Execution Strategy

### Multi-Wave Analysis

**Wave 1**: 17 parallel agents investigate core subsystems  
**Wave 2+**: Follow-up agents based on Wave 1 recommendations  
**Synthesis**: Compile findings into comprehensive report

### Report Requirements

Each analysis agent produces a report containing:

1. **Analysis Report**
   - Detailed findings with code references (file:line format)
   - Key data structures and their purposes
   - Control flow diagrams or descriptions
   - Architectural patterns observed

2. **Reference Material**
   - Function signatures for important functions
   - Critical constants and their meanings
   - Code snippets that illustrate key behaviors
   - Data flow examples

3. **Integration Points**
   - How this subsystem connects to others
   - Input/output interfaces
   - Dependencies (what it needs, what needs it)
   - State shared or modified

4. **Recommendations**
   - Specific areas needing deeper analysis
   - Proposed follow-up agents with clear scope
   - Open questions or ambiguities
   - Complexity hotspots

## Wave 1: Initial Investigation (17 Agents)

### Phase 1: Pre-REPL Initialization (5 agents)

**Agent 1: setup-environment**
- Scope: Analyze `setup()` function and related initialization
- Focus: cwd resolution, hooks loading, worktree creation, analytics sinks
- Key files: `main.tsx`, setup-related utilities
- Deliverable: How environment is configured before REPL starts
- Should flag: If hooks/worktree systems need dedicated deep-dive

**Agent 2: trust-and-auth**
- Scope: Analyze `showSetupScreens()` and authentication flows
- Focus: Trust dialogs, OAuth flows, onboarding sequences
- Key files: Setup screens, auth utilities
- Deliverable: How user identity and trust are established
- Should flag: If auth flows are complex enough for separate agent

**Agent 3: tool-system-init**
- Scope: Analyze tool loading, registration, and permission setup
- Focus: Tool discovery, permission context building, tool registry
- Key files: `Tool.ts`, `tools/` directory, permission system
- Deliverable: How tools become available to the agent loop
- Should flag: If specific tool categories (Bash, MCP, etc.) need analysis

**Agent 4: mcp-bootstrap**
- Scope: Analyze MCP (Model Context Protocol) server initialization
- Focus: MCP discovery, connection establishment, protocol setup
- Key files: MCP-related utilities, server configurations
- Deliverable: How external tools are integrated via MCP
- Should flag: If MCP protocol implementation needs deep-dive

**Agent 5: state-and-config**
- Scope: Deep dive on STATE, config, and settings initialization
- Focus: What gets loaded when, merge precedence, critical fields
- Key files: `bootstrap/state.ts`, `utils/config.ts`, `utils/settings/`
- Deliverable: Complete picture of available state at REPL start
- Should flag: Which STATE fields are most critical for agent loop

### Phase 2: Main REPL Loop (6 agents)

**Agent 6: repl-ui-input**
- Scope: Analyze REPL.tsx and user input handling
- Focus: Input capture, message formatting, UI rendering
- Key files: `screens/REPL.tsx`, input components
- Deliverable: How user input becomes a query
- Should flag: If UI framework (Ink) patterns need explanation

**Agent 7: query-orchestration**
- Scope: Analyze query.ts and orchestration logic
- Focus: Async generator pattern, streaming, loop control
- Key files: `query.ts` (69KB)
- Deliverable: How the query engine drives the agent loop
- Should flag: Complex control flow that needs separate analysis

**Agent 8: prompt-construction**
- Scope: Analyze how prompts are built for the API
- Focus: System prompt, tool definitions, context, user input assembly
- Key files: Prompt building utilities, `constants/prompts.js`
- Deliverable: What gets sent to Claude API and why
- Should flag: If prompt caching or optimization needs analysis

**Agent 9: api-interaction**
- Scope: Analyze Claude API communication
- Focus: Request format, streaming, tokens, caching, error handling
- Key files: API client code, Anthropic SDK usage
- Deliverable: Technical details of API interaction
- Should flag: If retry/error recovery logic is complex

**Agent 10: response-parsing**
- Scope: Analyze API response handling
- Focus: Streaming chunks, tool call extraction, thinking blocks, stop reasons
- Key files: Response parsing utilities
- Deliverable: How API responses become actions
- Should flag: If streaming or parsing logic is intricate

**Agent 11: inner-agent-loop**
- Scope: Analyze the think-act-observe cycle
- Focus: Tool execution, result integration, continuation logic, stop conditions
- Key files: Tool execution engine, loop control
- Deliverable: How the agent decides what to do next
- Should flag: If tool execution or stop logic needs deeper analysis

### Phase 3: Background & Lifecycle (4 agents)

**Agent 12: background-processes**
- Scope: Analyze concurrent processes running alongside REPL
- Focus: Daemon mode, background tasks, MCP servers, analytics
- Key files: `daemon/`, background task managers
- Deliverable: What runs in parallel with main loop
- Should flag: If daemon architecture needs dedicated analysis

**Agent 13: interruption-handling**
- Scope: Analyze user interrupts and error recovery
- Focus: Ctrl+C handling, cancellation, graceful degradation
- Key files: Signal handlers, error recovery code
- Deliverable: How the system handles disruptions
- Should flag: If error recovery patterns are sophisticated

**Agent 14: session-persistence**
- Scope: Analyze session save/restore and history
- Focus: Conversation persistence, state serialization, resume capability
- Key files: Session management, persistence utilities
- Deliverable: How sessions survive across runs
- Should flag: If persistence strategy is complex

**Agent 15: cleanup-and-exit**
- Scope: Analyze shutdown sequence
- Focus: Cleanup, resource disposal, final saves, exit codes
- Key files: Shutdown handlers, cleanup utilities
- Deliverable: What happens when user exits
- Should flag: If cleanup has critical ordering requirements

### Phase 4: Non-REPL Modes (2 agents)

**Agent 16: daemon-mode**
- Scope: Analyze daemon architecture and operation
- Focus: Worker processes, job queue, IPC, daemon lifecycle
- Key files: `daemon/main.js`, `daemon/workerRegistry.js`
- Deliverable: How daemon mode differs from interactive
- Should flag: If worker management is complex

**Agent 17: bridge-and-templates**
- Scope: Analyze bridge/remote-control and template jobs
- Focus: Bridge authentication, remote control, template system
- Key files: `bridge/bridgeMain.js`, `cli/handlers/templateJobs.js`
- Deliverable: How alternative modes work
- Should flag: If bridge protocol needs analysis

## Wave 2+: Follow-up Analysis

Launch additional agents based on Wave 1 recommendations for:
- Complex subsystems requiring deeper investigation
- Integration patterns spanning multiple subsystems
- Performance-critical paths
- Security-sensitive areas
- Any gaps or ambiguities identified

## Final Report Structure

**Target**: `codebase-report.md`

1. **Executive Summary**
   - Architecture overview
   - Key design patterns
   - Critical subsystems

2. **Launch & Bootstrap**
   - Fast-path routing (from CODEBASE.md)
   - Environment setup (Phase 1 synthesis)
   - State initialization

3. **Main REPL Loop**
   - User input → API → response → action cycle
   - Prompt construction strategy
   - Tool execution engine
   - Stop conditions and loop control

4. **Concurrent Systems**
   - Background processes
   - MCP servers
   - Session persistence
   - Interruption handling

5. **Alternative Modes**
   - Daemon mode
   - Bridge/remote-control
   - Template jobs

6. **Cross-Cutting Concerns**
   - State management patterns
   - Error handling strategy
   - Configuration precedence
   - Performance optimizations

7. **Reference Index**
   - Key files and their roles
   - Important functions by category
   - Data structures glossary

## Report Organization

**Location**: `sub-analysis/`

**Naming**:
- Wave 1: `wave-1/phase-N/NN-name.md`
- Wave 2+: `wave-N/topic-name.md`

## Success Criteria

- Human-readable narrative that explains "how it works"
- Technical reference for future investigations
- Code locations for every major concept
- Clear tracing from user action → system behavior
- Identification of all major subsystems and their interactions

---

## Execution Tracking

### Current Status

**Wave**: Not started  
**Progress**: 0/17 agents complete

#### Phase 1: Pre-REPL Initialization (0/5)

- [ ] Agent 1: setup-environment
  - Agent Name: setup-env-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-1/01-setup-environment.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 2: trust-and-auth
  - Agent Name: trust-auth-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-1/02-trust-and-auth.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 3: tool-system-init
  - Agent Name: tool-init-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-1/03-tool-system-init.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 4: mcp-bootstrap
  - Agent Name: mcp-boot-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-1/04-mcp-bootstrap.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 5: state-and-config
  - Agent Name: state-config-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-1/05-state-and-config.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

#### Phase 2: Main REPL Loop (0/6)

- [ ] Agent 6: repl-ui-input
  - Agent Name: repl-ui-agent
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/06-repl-ui-input.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 7: query-orchestration
  - Agent Name: query-orch
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/07-query-orchestration.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 8: prompt-construction
  - Agent Name: prompt-build
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/08-prompt-construction.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 9: api-interaction
  - Agent Name: api-call
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/09-api-interaction.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 10: response-parsing
  - Agent Name: response-parse
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/10-response-parsing.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 11: inner-agent-loop
  - Agent Name: agent-loop
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-2/11-inner-agent-loop.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

#### Phase 3: Background & Lifecycle (0/4)

- [ ] Agent 12: background-processes
  - Agent Name: bg-proc
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-3/12-background-processes.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 13: interruption-handling
  - Agent Name: interrupt
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-3/13-interruption-handling.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 14: session-persistence
  - Agent Name: session
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-3/14-session-persistence.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 15: cleanup-and-exit
  - Agent Name: cleanup
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-3/15-cleanup-and-exit.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

#### Phase 4: Non-REPL Modes (0/2)

- [ ] Agent 16: daemon-mode
  - Agent Name: daemon
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-4/16-daemon-mode.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

- [ ] Agent 17: bridge-and-templates
  - Agent Name: bridge
  - Status: Not started
  - Report: `sub-analysis/wave-1/phase-4/17-bridge-and-templates.md`
  - Started: -
  - Completed: -
  - Follow-ups identified: -

### Wave 2+ Status

**Pending**: Wave 1 must complete first to identify follow-up investigations.

### Report Synthesis Status

- [ ] Phase 1 synthesis → codebase-report.md Section 1.2-1.3
- [ ] Phase 2 synthesis → codebase-report.md Section 2
- [ ] Phase 3 synthesis → codebase-report.md Section 3
- [ ] Phase 4 synthesis → codebase-report.md Section 4
- [ ] Cross-cutting concerns → codebase-report.md Section 5
- [ ] Reference index → codebase-report.md Section 6
- [ ] Executive summary → codebase-report.md Section 0

### Notes

(Track observations, unexpected findings, or plan adjustments here)
