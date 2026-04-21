# Claude Code Codebase Analysis Plan

**Goal**: Comprehensive analysis of the Claude Code CLI codebase to understand:
- Launch sequence and initialization
- Main REPL loop architecture and execution
- Background processes and lifecycle management
- Alternative execution modes

## Execution Strategy

### Agent Team Architecture

**Team-Based Coordination**: Use Claude Code's agent team system for distributed codebase analysis.

**Structure**:
- **Orchestrator**: Team lead that creates and manages the shared task list
- **Analysis Teammates**: 17+ specialized agents that claim and complete analysis tasks
- **Tmux Sessions**: Each teammate runs in a separate tmux session for isolation and monitoring
- **Shared Task List**: Central coordination mechanism at `~/.claude/tasks/codebase-analysis/`

**Workflow**:
1. **Team Setup**: Orchestrator creates team and initializes shared task list
2. **Wave 1 Tasks**: Orchestrator creates 17 analysis tasks (one per subsystem)
3. **Teammate Spawn**: Orchestrator launches analysis teammates in tmux sessions
4. **Self-Coordination**: Teammates claim available tasks, complete work, mark done
5. **Wave 2+ Tasks**: Based on recommendations in completed reports, orchestrator adds follow-up tasks
6. **Synthesis**: After all waves, orchestrator compiles findings into final report

### Team Coordination Protocol

**Orchestrator Responsibilities**:
- Create and manage the `codebase-analysis` team
- Populate task list with analysis work (Wave 1, Wave 2+, synthesis)
- Spawn teammates with appropriate configurations
- **ACTIVELY MONITOR task progress with continuous loop** (every 3-5 minutes)
  - Check TaskList for in_progress tasks
  - Check file timestamps for output files
  - Send status checks to teammates working >5 minutes
  - Reassign tasks stuck >10 minutes with no output
  - Shutdown and replace teammates that fail repeatedly
- Add follow-up tasks based on completed report recommendations
- Synthesize final report when all waves complete

**Critical**: Orchestrator must run active monitoring loop, not passively wait for notifications

**Teammate Responsibilities**:
- Check TaskList for available work (status: pending, no owner, not blocked)
- Claim tasks in ID order (prefer lower IDs first for context flow)
- Complete analysis and write report to specified location
- Mark task as completed with TaskUpdate
- Include follow-up recommendations in report
- Repeat until no available tasks remain

**Active Monitoring Protocol (CRITICAL)**:

The orchestrator MUST implement a continuous monitoring loop. DO NOT passively wait for notifications.

**Monitoring Loop (every 3-5 minutes)**:
1. Run `TaskList()` to get all in_progress tasks
2. For each in_progress task:
   - Check task start time (when status changed to in_progress)
   - Check if output file exists and has recent modifications
   - Calculate elapsed time
3. **Check for idle teammates with pending work**:
   - If pending tasks exist AND teammates are idle >5 minutes
   - Send direct assignment messages (not just broadcasts)
   - Example: "Task #N is available. Please claim it immediately."
   - If no response in 2 minutes, shutdown idle teammate and spawn replacement
4. Take action based on elapsed time:
   - **0-5 minutes**: Normal, no action
   - **5-10 minutes**: Send status check via SendMessage
   - **10+ minutes with no output file**: Reassign task immediately
   - **15+ minutes with partial output**: Send warning, consider reassignment
   - **20+ minutes**: Force reassignment, mark teammate as problematic

**Status Check Message Template**:
```
Status check: Task #N has been in progress for X minutes. 
What's your current status? Are you still working or stuck? 
Please provide a brief update on your progress.
```

**Reassignment Procedure**:
1. Use `TaskUpdate({taskId: "N", owner: "different-analyst", status: "pending"})`
2. Send message to new owner: "Task #N reassigned to you due to stuck teammate"
3. Log the incident (which teammate, which task, elapsed time)
4. If same teammate gets stuck 3+ times, send shutdown request and spawn replacement

**File Monitoring**:
- Check output file existence: `ls -lh <report_path>`
- Check file modification time to detect stalls
- If file exists but hasn't been modified in 5+ minutes, investigate

**Signs of Stuck Teammate**:
- Task in_progress for >10 minutes with no output file created
- Output file exists but no modifications for >5 minutes
- Teammate not responding to status check within 2 minutes
- Repeated failures across multiple tasks

**Recovery Actions**:
1. **First timeout**: Reassign task to different teammate
2. **Second timeout (same teammate)**: Send warning + reassign
3. **Third timeout (same teammate)**: Shutdown teammate, spawn replacement
4. **Always**: Document the failure for debugging

**Implementation Note**: 
The orchestrator cannot "sleep" or "wait" - instead, use ScheduleWakeup or check on every user interaction. Active monitoring is the orchestrator's primary job during execution.

**Tmux Session Management**:
- Each teammate runs in named tmux session: `codebase-analysis-{agent-name}`
- Orchestrator can attach to sessions for debugging: `tmux attach -t codebase-analysis-{name}`
- Sessions persist until teammate completes or is shut down
- Use `tmux list-sessions | grep codebase-analysis` to view active teammates

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
   - Proposed follow-up tasks with clear scope
   - Open questions or ambiguities
   - Complexity hotspots

## Wave 1: Initial Investigation (17 Tasks)

**Task Creation Pattern**: Orchestrator creates all 17 tasks at team initialization. Each task includes:
- Clear subject and description
- Target report path in metadata
- Subsystem scope and focus areas
- Files/directories to investigate
- Guidance on what to flag for follow-up

### Phase 1: Pre-REPL Initialization (5 tasks)

**Task 1: setup-environment**
- Subject: Analyze setup() function and environment initialization
- Description: Analyze `setup()` function and related initialization including cwd resolution, hooks loading, worktree creation, and analytics sinks
- Key files: `main.tsx`, setup-related utilities
- Deliverable: How environment is configured before REPL starts
- Should flag: If hooks/worktree systems need dedicated deep-dive
- Report: `sub-analysis/wave-1/phase-1/01-setup-environment.md`

**Task 2: trust-and-auth**
- Subject: Analyze showSetupScreens() and authentication flows
- Description: Analyze `showSetupScreens()` and authentication flows including trust dialogs, OAuth flows, and onboarding sequences
- Key files: Setup screens, auth utilities
- Deliverable: How user identity and trust are established
- Should flag: If auth flows are complex enough for separate task
- Report: `sub-analysis/wave-1/phase-1/02-trust-and-auth.md`

**Task 3: tool-system-init**
- Subject: Analyze tool loading, registration, and permissions
- Description: Analyze tool loading, registration, and permission setup including tool discovery, permission context building, and tool registry
- Key files: `Tool.ts`, `tools/` directory, permission system
- Deliverable: How tools become available to the agent loop
- Should flag: If specific tool categories (Bash, MCP, etc.) need analysis
- Report: `sub-analysis/wave-1/phase-1/03-tool-system-init.md`

**Task 4: mcp-bootstrap**
- Subject: Analyze MCP server initialization and protocol setup
- Description: Analyze MCP (Model Context Protocol) server initialization including MCP discovery, connection establishment, and protocol setup
- Key files: MCP-related utilities, server configurations
- Deliverable: How external tools are integrated via MCP
- Should flag: If MCP protocol implementation needs deep-dive
- Report: `sub-analysis/wave-1/phase-1/04-mcp-bootstrap.md`

**Task 5: state-and-config**
- Subject: Deep dive on STATE, config, and settings initialization
- Description: Deep dive on STATE, config, and settings initialization including what gets loaded when, merge precedence, and critical fields
- Key files: `bootstrap/state.ts`, `utils/config.ts`, `utils/settings/`
- Deliverable: Complete picture of available state at REPL start
- Should flag: Which STATE fields are most critical for agent loop
- Report: `sub-analysis/wave-1/phase-1/05-state-and-config.md`

### Phase 2: Main REPL Loop (6 tasks)

**Task 6: repl-ui-input**
- Subject: Analyze REPL.tsx and user input handling
- Description: Analyze REPL.tsx and user input handling including input capture, message formatting, and UI rendering
- Key files: `screens/REPL.tsx`, input components
- Deliverable: How user input becomes a query
- Should flag: If UI framework (Ink) patterns need explanation
- Report: `sub-analysis/wave-1/phase-2/06-repl-ui-input.md`

**Task 7: query-orchestration**
- Subject: Analyze query.ts orchestration and async patterns
- Description: Analyze query.ts and orchestration logic including async generator pattern, streaming, and loop control
- Key files: `query.ts` (69KB)
- Deliverable: How the query engine drives the agent loop
- Should flag: Complex control flow that needs separate analysis
- Report: `sub-analysis/wave-1/phase-2/07-query-orchestration.md`

**Task 8: prompt-construction**
- Subject: Analyze prompt building for Claude API
- Description: Analyze how prompts are built for the API including system prompt, tool definitions, context, and user input assembly
- Key files: Prompt building utilities, `constants/prompts.js`
- Deliverable: What gets sent to Claude API and why
- Should flag: If prompt caching or optimization needs analysis
- Report: `sub-analysis/wave-1/phase-2/08-prompt-construction.md`

**Task 9: api-interaction**
- Subject: Analyze Claude API communication patterns
- Description: Analyze Claude API communication including request format, streaming, tokens, caching, and error handling
- Key files: API client code, Anthropic SDK usage
- Deliverable: Technical details of API interaction
- Should flag: If retry/error recovery logic is complex
- Report: `sub-analysis/wave-1/phase-2/09-api-interaction.md`

**Task 10: response-parsing**
- Subject: Analyze API response handling and parsing
- Description: Analyze API response handling including streaming chunks, tool call extraction, thinking blocks, and stop reasons
- Key files: Response parsing utilities
- Deliverable: How API responses become actions
- Should flag: If streaming or parsing logic is intricate
- Report: `sub-analysis/wave-1/phase-2/10-response-parsing.md`

**Task 11: inner-agent-loop**
- Subject: Analyze think-act-observe cycle implementation
- Description: Analyze the think-act-observe cycle including tool execution, result integration, continuation logic, and stop conditions
- Key files: Tool execution engine, loop control
- Deliverable: How the agent decides what to do next
- Should flag: If tool execution or stop logic needs deeper analysis
- Report: `sub-analysis/wave-1/phase-2/11-inner-agent-loop.md`

### Phase 3: Background & Lifecycle (4 tasks)

**Task 12: background-processes**
- Subject: Analyze concurrent processes running alongside REPL
- Description: Analyze concurrent processes running alongside REPL including daemon mode, background tasks, MCP servers, and analytics
- Key files: `daemon/`, background task managers
- Deliverable: What runs in parallel with main loop
- Should flag: If daemon architecture needs dedicated analysis
- Report: `sub-analysis/wave-1/phase-3/12-background-processes.md`

**Task 13: interruption-handling**
- Subject: Analyze user interrupts and error recovery
- Description: Analyze user interrupts and error recovery including Ctrl+C handling, cancellation, and graceful degradation
- Key files: Signal handlers, error recovery code
- Deliverable: How the system handles disruptions
- Should flag: If error recovery patterns are sophisticated
- Report: `sub-analysis/wave-1/phase-3/13-interruption-handling.md`

**Task 14: session-persistence**
- Subject: Analyze session save/restore and history
- Description: Analyze session save/restore and history including conversation persistence, state serialization, and resume capability
- Key files: Session management, persistence utilities
- Deliverable: How sessions survive across runs
- Should flag: If persistence strategy is complex
- Report: `sub-analysis/wave-1/phase-3/14-session-persistence.md`

**Task 15: cleanup-and-exit**
- Subject: Analyze shutdown sequence and cleanup
- Description: Analyze shutdown sequence including cleanup, resource disposal, final saves, and exit codes
- Key files: Shutdown handlers, cleanup utilities
- Deliverable: What happens when user exits
- Should flag: If cleanup has critical ordering requirements
- Report: `sub-analysis/wave-1/phase-3/15-cleanup-and-exit.md`

### Phase 4: Non-REPL Modes (2 tasks)

**Task 16: daemon-mode**
- Subject: Analyze daemon architecture and operation
- Description: Analyze daemon architecture and operation including worker processes, job queue, IPC, and daemon lifecycle
- Key files: `daemon/main.js`, `daemon/workerRegistry.js`
- Deliverable: How daemon mode differs from interactive
- Should flag: If worker management is complex
- Report: `sub-analysis/wave-1/phase-4/16-daemon-mode.md`

**Task 17: bridge-and-templates**
- Subject: Analyze bridge/remote-control and template jobs
- Description: Analyze bridge/remote-control and template jobs including bridge authentication, remote control, and template system
- Key files: `bridge/bridgeMain.js`, `cli/handlers/templateJobs.js`
- Deliverable: How alternative modes work
- Should flag: If bridge protocol needs analysis
- Report: `sub-analysis/wave-1/phase-4/17-bridge-and-templates.md`

## Wave 2+: Follow-up Analysis

Orchestrator reviews completed Wave 1 reports and creates additional tasks based on recommendations for:
- Complex subsystems requiring deeper investigation
- Integration patterns spanning multiple subsystems
- Performance-critical paths
- Security-sensitive areas
- Any gaps or ambiguities identified

**Task Naming**: `wave-2/NN-descriptive-name.md` (or wave-3, wave-4, etc.)

## Synthesis Phase: Hierarchical Analysis & Compilation

After all analysis waves complete, execute a **3-phase hierarchical synthesis** to build the final report.

### Synthesis Design Philosophy

The synthesis team architecture will be designed and executed as a **separate planning phase** after analysis completes. The orchestrator will:
1. Review all completed analysis outputs
2. Design an appropriate synthesis team structure
3. Execute synthesis without requiring additional inputs

This approach allows the synthesis strategy to adapt to the actual findings rather than pre-committing to a fixed structure.

### Synthesis Phase 1: Summarization

**Goal**: Extract key findings from each detailed analysis report into lightweight summaries.

For each completed analysis report, create a summary task that produces:
- **Key Findings**: 3-5 bullet points of main discoveries
- **Critical Code Locations**: File paths and line numbers of important functions/structures
- **Integration Points**: What this subsystem connects to and how
- **Recommendations**: Follow-up needs identified
- **Complexity Assessment**: Simple/Medium/Complex rating

**Output**: `summaries/NN-subsystem-name.md` (one per analysis report)

### Synthesis Phase 2: Grouping & Interaction Analysis

**Goal**: Identify which subsystems are distinct vs which interact heavily and need joint analysis.

**Task: Analyze Summaries for Patterns**
- Read all summaries from Phase 1
- Identify:
  - **Distinct Subsystems**: Can be documented independently
  - **Interaction Clusters**: Groups of subsystems that work together
  - **Shared Concerns**: Cross-cutting patterns (state management, error handling, etc.)
- Create grouping document showing:
  - Independent subsystems (list)
  - Interaction clusters with members and relationships
  - Cross-cutting concerns and which subsystems they span

**Output**: `synthesis-plan.md` with grouping decisions

**Task: Deep-Dive Interaction Analysis** (one per cluster)
- For each interaction cluster identified:
  - Read detailed reports for all cluster members
  - Trace data flow between components
  - Document integration mechanisms
  - Create narrative explaining how components work together
  - Identify critical interaction points

**Output**: `interactions/cluster-name.md` (one per cluster)

### Synthesis Phase 3: Final Compilation (Granular Tasks)

**CRITICAL LESSON**: Do NOT create monolithic "compile entire report" tasks. Break into section-level tasks.

**Goal**: Build `codebase-report.md` incrementally, section by section, with verifiable progress.

**Task Breakdown Strategy**:
1. **Read synthesis-plan.md** to understand proposed structure
2. **Create one task per major section** (not one task for entire report)
3. **Each task**:
   - Has clear scope (e.g., "Write Section 3: Bootstrap")
   - Specifies input sources (which reports/summaries to use)
   - Specifies output mode (create file vs append)
   - Has realistic time estimate (5-10 min per section)
   - Produces verifiable output (section appears in file)

**Example Task Granularity**:
- ✅ Good: "Write Section 3: Bootstrap (Cluster A)" → 1 task, ~10 min
- ❌ Bad: "Compile final codebase-report.md" → 1 task, unknown duration, no progress visibility

**Section-Level Tasks** (9 total):
1. Write Section 1: Executive Summary (creates file)
2. Write Section 2: Runtime Topology Diagram (append)
3. Write Section 3: Bootstrap / Cluster A (append)
4. Write Section 4: Request Cycle / Cluster B (append)
5. Write Section 5: Lifecycle / Cluster C (append)
6. Write Section 6: Background & Alt Modes / Cluster D (append)
7. Write Section 7: Cross-Cutting Concerns (append)
8. Write Section 8: Appendices (append)
9. Write Section 9: Recommendations (append)

**Benefits of Granular Tasks**:
- ✅ Progress visibility (can check which sections exist)
- ✅ Parallel work possible (multiple teammates on different sections)
- ✅ Easy recovery (if stuck on section 4, others continue)
- ✅ Clear time estimates (5-10 min per section vs ??? for whole report)
- ✅ Monitoring is simple (check file size, grep for section headers)

**Monitoring Granular Tasks**:
- Check if section header exists in file: `grep "^## N\." codebase-report.md`
- Check file size growth: `wc -l codebase-report.md`
- Stuck detection: Section task in_progress >15 min → reassign
- Each section = ~500-1500 lines, manageable scope

**Output**: Complete `codebase-report.md` built incrementally

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
- Wave 2+: `wave-N/NN-topic-name.md`

## Success Criteria

- Human-readable narrative that explains "how it works"
- Technical reference for future investigations
- Code locations for every major concept
- Clear tracing from user action → system behavior
- Identification of all major subsystems and their interactions

---

## Orchestrator Execution Guide

### Step 1: Initialize Team

```typescript
// Create team
TeamCreate({
  team_name: "codebase-analysis",
  description: "Distributed analysis of Claude Code CLI codebase"
})
```

### Step 2: Create Wave 1 Tasks

For each of the 17 tasks defined above, use TaskCreate:

```typescript
TaskCreate({
  subject: "<task subject>",
  description: "<full task description with scope, focus, key files, deliverables, and flagging guidance>",
  metadata: {
    report_path: "sub-analysis/wave-1/phase-N/NN-name.md",
    phase: "N",
    wave: "1"
  }
})
```

### Step 3: Launch Analysis Teammates

Spawn general-purpose analysis agents in tmux sessions. Example for spawning 5 concurrent teammates:

```bash
# Teammate 1
tmux new-session -d -s codebase-analysis-analyst-1 "claude --team codebase-analysis --name analyst-1"

# Teammate 2
tmux new-session -d -s codebase-analysis-analyst-2 "claude --team codebase-analysis --name analyst-2"

# ... (repeat for 3-5 teammates total)
```

Teammates will automatically:
- Join the team
- Check TaskList for available work
- Claim and complete tasks
- Mark tasks as completed
- Repeat until no work remains

### Step 4: Monitor Progress

Periodically check task completion:

```typescript
TaskList()  // Review completed vs pending tasks
```

View active tmux sessions:
```bash
tmux list-sessions | grep codebase-analysis
```

Attach to a teammate's session for debugging:
```bash
tmux attach -t codebase-analysis-analyst-1
```

### Step 5: Wave 2+ Tasks

After Wave 1 completes:
1. Read all Wave 1 reports
2. Extract recommendations for follow-up analysis
3. Create Wave 2 tasks based on high-priority recommendations
4. Teammates will automatically claim and complete new tasks

### Step 6: Synthesis Phase

After all analysis waves complete, execute **hierarchical synthesis**:

**6.1: Review Outputs**
- Read all completed analysis reports
- Assess complexity and interconnections

**6.2: Design Synthesis Team**
- Determine synthesis team structure based on findings
- This is done as a separate planning exercise
- May involve summarizers, interaction analysts, and compilers

**6.3: Execute Synthesis**
- Launch synthesis team (separate from analysis team)
- Synthesis team follows 3-phase process:
  1. Summarization: Create lightweight summaries of each report
  2. Grouping: Identify distinct vs interacting subsystems
  3. Compilation: Build final narrative report

### Step 7: Completion

When all tasks are completed:
1. Review final `codebase-report.md`
2. Send shutdown messages to teammates
3. Clean up tmux sessions

```typescript
SendMessage({
  to: "analyst-1",
  message: {type: "shutdown_request"}
})

// After all teammates acknowledge shutdown:
// tmux kill-session -t codebase-analysis-analyst-1
```

---

## Teammate Instructions

**You are an analysis teammate on the codebase-analysis team.**

### Startup

When you join the team:
1. Read team config: `~/.claude/teams/codebase-analysis/config.json`
2. Note your name and teammate names
3. Check TaskList for available work

### Main Loop

Repeat until no available tasks:

1. **Find Work**:
   ```typescript
   TaskList()
   ```
   Look for: status='pending', no owner, empty blockedBy
   
2. **Claim Task** (prefer lowest ID):
   ```typescript
   TaskUpdate({taskId: "N", owner: "<your-name>"})
   ```

3. **Start Work**:
   ```typescript
   TaskUpdate({taskId: "N", status: "in_progress"})
   ```

4. **Complete Analysis**:
   - Read relevant source files
   - Analyze code patterns, data structures, control flow
   - Document findings with code references (file:line)
   - Identify integration points
   - Provide follow-up recommendations
   - Write report to path specified in task metadata

5. **Mark Complete**:
   ```typescript
   TaskUpdate({taskId: "N", status: "completed"})
   ```

6. **Repeat**: Go back to step 1

### When No Work Available

If TaskList shows no available tasks (all pending tasks are blocked or owned):
- Wait for new tasks to be created
- Or notify team lead that you're idle
- Or help unblock blocked tasks if you can

### Shutdown

When you receive a shutdown request from the orchestrator:
- Acknowledge the request
- Exit gracefully
