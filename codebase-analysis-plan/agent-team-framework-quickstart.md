# Agent Team Framework: Dynamic Multi-Wave Analysis with Autonomous Orchestration

**Purpose**: A reusable pattern for large-scale analysis or work that requires:
- Multiple waves where each wave's output determines the next
- Hierarchical synthesis to manage information overload
- Autonomous orchestration with active monitoring and recovery
- No human intervention required after initial setup

---

## Core Pattern: Dynamic Wave Execution

### Overview

```
Wave 1 (Analysis) → Review → Wave 2 (Follow-up) → Review → ... → Synthesis (Hierarchical) → Final Output
     ↓                ↓            ↓                  ↓                    ↓
  Outputs      Recommendations  Outputs        Convergence?         Bite-size → Narrative
```

**Key Principle**: Each wave's **outputs** determine whether and what comes next. The orchestrator is autonomous and adaptive.

---

## Phase 1: Wave-Based Analysis

### 1.1 Wave Structure

Each wave consists of:

**Input**: 
- Tasks (what needs to be done)
- Success criteria (when is this wave complete?)
- Deliverables (what artifacts to produce)

**Execution**:
- Teammates claim and complete tasks autonomously
- Orchestrator monitors progress actively
- Recovery happens automatically

**Output**:
- Artifacts (reports, data, analysis)
- Recommendations (what needs follow-up)
- Metadata (complexity, gaps, interactions)

**Review Decision**:
- Convergence achieved? → Move to Synthesis
- Gaps identified? → Design Wave N+1
- Complexity flags? → Create deep-dive tasks

### 1.2 Wave Design Principles

**Wave 1** (Initial Coverage):
- Broad scope, parallel independent tasks
- Goal: Cover the entire problem space
- Each task produces: findings + recommendations
- Granularity: Medium (1-2 hours per task max)

**Wave 2+** (Targeted Deep-Dives):
- Narrow scope, focused on gaps/complexity from Wave N-1
- Goal: Address specific recommendations
- Created dynamically based on Wave N-1 outputs
- Granularity: Fine (30 min - 1 hour per task)

**Convergence Check** (after each wave):
```
if (all_areas_covered AND no_critical_gaps AND recommendations_manageable):
    proceed_to_synthesis()
else:
    design_next_wave()
```

---

## Phase 2: Hierarchical Synthesis

### 2.1 The Information Overload Problem

After multiple waves, you have:
- 10-50+ detailed reports
- Too much to process in one pass
- Risk of missing patterns or duplicating work

**Solution**: Progressive summarization in 3 stages

### 2.2 Synthesis Stage 1: Summarization

**Goal**: Extract key findings from detailed outputs into lightweight summaries.

**Process**:
- One summary task per logical grouping (e.g., per phase, per category)
- Each summary extracts:
  - Key findings (3-5 bullets)
  - Critical locations (file:line, API endpoints, etc.)
  - Integration points
  - Complexity assessment
- Output: Bite-size summaries (1-2 pages each)

**Why**: Makes next stage tractable - read 5 summaries instead of 50 reports

### 2.3 Synthesis Stage 2: Grouping & Pattern Detection

**Goal**: Identify which pieces are independent vs. which interact.

**Process**:
- Read all summaries (now manageable)
- Identify:
  - **Distinct items**: Can be documented independently
  - **Interaction clusters**: Groups that work together
  - **Cross-cutting concerns**: Patterns across multiple clusters
- Output: Grouping plan / synthesis strategy

**Why**: Determines how to organize the final narrative

### 2.4 Synthesis Stage 3: Compilation

**Goal**: Build final output with clear narrative structure.

**Process**:
- **NOT a monolithic task** (learned this the hard way)
- Break into section-level tasks:
  - One task per major section
  - Each task: read relevant summaries + original reports → write section
  - Sequential or parallel depending on dependencies
- Each section is verifiable (grep for headers, check line count)

**Output**: Final comprehensive document

**Why**: Granular tasks enable:
- Progress monitoring (which sections done?)
- Parallel work (multiple teammates on different sections)
- Easy recovery (stuck on section 4? others continue)

---

## Phase 3: Autonomous Orchestrator Pattern

### 3.1 The Orchestrator's Job

**Not**: Passively wait for notifications
**Is**: Actively monitor, adapt, and recover

**Core Responsibilities**:
1. **Setup**: Create team, design initial tasks
2. **Spawn**: Launch teammates
3. **Monitor**: Active loop checking progress (every 3-5 min)
4. **Adapt**: Add tasks, reassign work, respawn teammates
5. **Decide**: Determine when to move to next wave/phase
6. **Shutdown**: Clean termination

### 3.2 Active Monitoring Loop

**Every 3-5 minutes**, the orchestrator must:

```typescript
// 1. Check in-progress tasks
tasks = TaskList()
for (task in tasks.in_progress) {
  elapsed = now() - task.start_time
  
  // 2. Verify progress
  output_file = task.metadata.output_path
  file_exists = check_file(output_file)
  last_modified = get_mtime(output_file)
  
  // 3. Take action based on elapsed time + file status
  if (elapsed > 10min AND !file_exists) {
    // Stuck - no output at all
    reassign_task(task)
  } else if (elapsed > 15min AND stale_file(last_modified)) {
    // Stalled - partial output, not progressing
    send_status_check(task.owner)
  } else if (elapsed > 5min AND no_recent_progress) {
    // Early warning
    send_status_check(task.owner)
  }
}

// 4. Check for idle teammates with pending work
idle_teammates = get_idle_teammates()
pending_tasks = tasks.pending
if (idle_teammates.length > 0 AND pending_tasks.length > 0) {
  // Idle teammate not claiming work - direct assignment needed
  for (teammate in idle_teammates) {
    send_direct_assignment(teammate, pending_tasks[0])
  }
}

// 5. Schedule next monitoring check
schedule_wakeup(3_minutes)
```

**Key Insight**: Don't wait for notifications - actively check.

### 3.3 Task Granularity Rules

**Principle**: Every task must have verifiable progress within 10 minutes.

**Good Task Granularity**:
- ✅ "Analyze setup() function and write report" (10-15 min)
- ✅ "Write Section 3: Bootstrap" (10-15 min)
- ✅ "Summarize Phase 1 reports" (8-12 min)

**Bad Task Granularity**:
- ❌ "Compile entire final report" (unknown duration, no progress visibility)
- ❌ "Analyze all of main.tsx" (too broad, 804KB file)
- ❌ "Complete Wave 2 follow-ups" (multiple separate tasks, no clear scope)

**Rule of Thumb**:
- If task duration estimate > 20 minutes → break into smaller tasks
- If can't verify progress within 10 minutes → too coarse
- Each task should produce **one** verifiable artifact

### 3.4 Recovery Procedures

**Stuck Task** (in_progress >10min, no output):
```
1. Send status check message
2. Wait 2 minutes for response
3. If no response: reassign to different teammate
4. Log the incident
```

**Stalled Task** (in_progress >15min, partial output but stale):
```
1. Send urgent status check
2. Wait 2 minutes
3. If no progress: reassign
4. Consider breaking task into smaller pieces
```

**Idle Teammate** (pending tasks exist, teammate idle >5min):
```
1. Send direct assignment (not broadcast)
2. If no response in 2 minutes: shutdown and spawn replacement
3. Pattern: idle teammates often stuck in broken state
```

**Repeated Failures** (same teammate stuck 3+ times):
```
1. Shutdown problematic teammate
2. Spawn replacement
3. Reassign failed tasks to new teammate
4. Log for post-mortem
```

### 3.5 Progress Verification

**File-Based Verification** (for write tasks):
```bash
# Check if output exists
ls -lh <output_path>

# Check file size (should be growing)
wc -l <output_path>

# Check modification time (should be recent)
stat -c %Y <output_path>

# Check for expected structure (e.g., section headers)
grep "^## [0-9]" <output_path>
```

**Task Status Verification** (for all tasks):
```typescript
# Get task details
task = TaskGet(taskId)

# Check state transitions
if (task.status == "in_progress" AND task.owner != null) {
  // Task claimed and active
} else if (task.status == "pending" AND task.blockedBy.length == 0) {
  // Available for claiming
}
```

### 3.6 Wave Transition Decision

After each wave completes, orchestrator reviews outputs:

```typescript
function decide_next_phase(wave_outputs) {
  // 1. Read all outputs from completed wave
  outputs = read_all_outputs(wave_outputs)
  
  // 2. Extract recommendations
  recommendations = extract_recommendations(outputs)
  
  // 3. Categorize by priority
  high_priority = recommendations.filter(r => r.priority == "high")
  medium_priority = recommendations.filter(r => r.priority == "medium")
  
  // 4. Check convergence criteria
  coverage_complete = all_areas_covered(outputs)
  critical_gaps = high_priority.length
  
  // 5. Decision logic
  if (coverage_complete AND critical_gaps == 0) {
    return "proceed_to_synthesis"
  } else if (critical_gaps > 0) {
    return design_wave_n_plus_1(high_priority)
  } else if (medium_priority.length > threshold) {
    return design_wave_n_plus_1(medium_priority.top(5))
  } else {
    return "proceed_to_synthesis"
  }
}
```

---

## Phase 4: Team Lifecycle Management

### 4.1 Team Creation

```typescript
// 1. Create team
TeamCreate({
  team_name: "project-name",
  description: "High-level goal"
})

// 2. Design initial tasks (Wave 1)
tasks = design_wave_1_tasks(problem_space)

// 3. Create all tasks
for (task in tasks) {
  TaskCreate({
    subject: task.subject,
    description: task.description,
    metadata: task.metadata  // Include output_path, phase, wave
  })
}
```

### 4.2 Teammate Spawning

**Strategy**: Start with small team, scale if needed

```typescript
// Initial team size: 3-5 teammates for most tasks
num_teammates = min(5, ceil(num_tasks / 3))

for (i in 1..num_teammates) {
  Agent({
    name: `analyst-${i}`,
    team_name: "project-name",
    run_in_background: true,
    prompt: teammate_instructions()
  })
}
```

**Teammate Instructions Template**:
```
You are {name} on the {team_name} team.

Your job: Autonomously claim and complete tasks.

Instructions:
1. Check TaskList for available work (status: pending, no owner, not blocked)
2. Claim lowest-ID available task (TaskUpdate: set owner to "{name}")
3. Mark in_progress (TaskUpdate: status = "in_progress")
4. Complete the work as specified in task description
5. Write output to path specified in task.metadata.output_path
6. Mark completed (TaskUpdate: status = "completed")
7. Repeat until no available tasks

Working directory: {cwd}

Start now.
```

### 4.3 Monitoring Implementation

**Option 1: User-Prompted Checks** (current limitation):
```
User prompts every 5 minutes to check progress.
Orchestrator runs monitoring loop on each prompt.
```

**Option 2: ScheduleWakeup** (requires /loop mode):
```typescript
// In /loop mode only
ScheduleWakeup({
  delaySeconds: 180,  // 3 minutes (stay in cache)
  reason: "Monitor team progress",
  prompt: "Run monitoring loop: TaskList, check files, recover stuck tasks"
})
```

**Option 3: Autonomous Loop** (ideal, NOT CURRENTLY AVAILABLE):
```typescript
// Hypothetical - orchestrator runs continuously
// THIS DOES NOT EXIST - requires Claude Code feature enhancement
while (tasks_remaining()) {
  run_monitoring_loop()
  sleep(180)
}

// Would require home-level settings:
// ~/.claude/settings.json:
// {
//   "agent_teams": {
//     "enable_autonomous_orchestration": true,
//     "monitoring_interval_seconds": 180
//   }
// }
```

**Reality**: User must prompt "check progress" every 3-5 minutes for monitoring to happen.

### 4.4 Shutdown Protocol

**Graceful Shutdown**:
```typescript
// 1. Wait for all tasks to complete
while (TaskList().filter(t => t.status != "completed").length > 0) {
  monitor_and_wait()
}

// 2. Send shutdown requests to all teammates
teammates = read_team_config().members
for (teammate in teammates) {
  SendMessage({
    to: teammate.name,
    message: {
      type: "shutdown_request",
      reason: "All work complete. Thank you."
    }
  })
}

// 3. Wait for acknowledgments
wait_for_shutdown_confirmations()

// 4. Clean up (optional)
// - Archive outputs
// - Delete team config
// - Summary report
```

---

## Template: Applying the Framework

### Step 1: Define Your Problem

```typescript
problem = {
  name: "Your analysis/work name",
  goal: "What you want to achieve",
  inputs: ["What data/code/resources you're analyzing"],
  success_criteria: "How you know it's complete"
}
```

### Step 2: Design Wave 1

**Decompose the problem space**:
```typescript
wave_1_tasks = decompose_problem_space(problem)

// Example: Codebase analysis
// Decompose by: subsystems, phases, or functional areas
// Each task = 1 subsystem/area

// Example: Research project  
// Decompose by: questions, topics, or sources
// Each task = 1 research question

// Principle: Cover entire space, independent tasks
```

**Task Template**:
```typescript
{
  subject: "Clear, actionable title",
  description: `
    What to analyze: <scope>
    Focus areas: <specific points>
    Key files/sources: <where to look>
    Deliverable: <what to produce>
    Should flag: <what needs follow-up>
  `,
  metadata: {
    output_path: "outputs/wave-1/NN-name.md",
    wave: "1",
    category: "<grouping>",
    estimated_duration: "10-20min"
  }
}
```

### Step 3: Execute Wave 1

```typescript
// 1. Create team
TeamCreate({team_name: problem.name})

// 2. Create Wave 1 tasks
for (task in wave_1_tasks) {
  TaskCreate(task)
}

// 3. Spawn teammates
spawn_teammates(count: 5, team_name: problem.name)

// 4. Monitor actively
while (!wave_1_complete()) {
  run_monitoring_loop()
  wait(3_minutes)
}
```

### Step 4: Review & Decide Next Wave

```typescript
// 1. Review all Wave 1 outputs
outputs = read_all_wave_outputs("wave-1")

// 2. Extract recommendations
recommendations = []
for (output in outputs) {
  recs = extract_section(output, "Recommendations")
  recommendations.push(...recs)
}

// 3. Categorize
high_priority = filter_by_priority(recommendations, "high")
medium_priority = filter_by_priority(recommendations, "medium")

// 4. Decide
if (high_priority.length > 0) {
  // Need Wave 2
  wave_2_tasks = design_tasks_from_recommendations(high_priority)
  execute_wave_2(wave_2_tasks)
} else {
  // Proceed to synthesis
  proceed_to_synthesis()
}
```

### Step 5: Hierarchical Synthesis

```typescript
// Stage 1: Summarization
summary_tasks = []
for (group in logical_groupings(all_outputs)) {
  summary_tasks.push({
    subject: `Summarize ${group.name} outputs`,
    description: `Read ${group.outputs.length} reports and extract key findings`,
    metadata: {output_path: `summaries/${group.name}.md`}
  })
}
execute_tasks(summary_tasks)

// Stage 2: Grouping
clustering_task = {
  subject: "Identify interaction clusters",
  description: "Read all summaries and identify independent vs interacting items",
  metadata: {output_path: "synthesis-plan.md"}
}
execute_task(clustering_task)

// Stage 3: Compilation
synthesis_plan = read("synthesis-plan.md")
section_tasks = []
for (section in synthesis_plan.sections) {
  section_tasks.push({
    subject: `Write ${section.name}`,
    description: `Use summaries + relevant reports to write this section`,
    metadata: {
      output_path: "final-report.md",
      mode: section.id == 1 ? "create" : "append"
    }
  })
}
execute_tasks(section_tasks)
```

### Step 6: Shutdown & Deliver

```typescript
// 1. Verify completion
final_output = read("final-report.md")
verify(final_output.meets(problem.success_criteria))

// 2. Shutdown team
shutdown_all_teammates()

// 3. Deliver
return {
  status: "complete",
  outputs: {
    detailed: "outputs/wave-*/",
    summaries: "summaries/",
    synthesis_plan: "synthesis-plan.md",
    final: "final-report.md"
  },
  stats: {
    waves: num_waves,
    tasks: num_tasks,
    duration: elapsed_time
  }
}
```

---

## Key Design Principles

### 1. Granularity Over Monoliths

**Bad**: One massive task that does everything
**Good**: Many small tasks, each verifiable

**Why**: Small tasks enable:
- Progress visibility
- Parallel work
- Easy recovery
- Clear time estimates

### 2. Active Over Passive

**Bad**: Wait for notifications, react when things go wrong
**Good**: Continuously monitor, detect problems early

**Why**: Proactive monitoring enables:
- Early detection of stuck teammates
- Automatic recovery
- Resource optimization (reassign idle teammates)

### 3. Dynamic Over Static

**Bad**: Pre-plan all waves upfront
**Good**: Each wave's outputs determine the next

**Why**: Dynamic adaptation enables:
- Responding to actual findings
- Avoiding unnecessary work
- Focusing on real complexity

### 4. Hierarchical Over Flat

**Bad**: Try to process 50 reports in one pass
**Good**: Progressive summarization (details → summaries → narrative)

**Why**: Hierarchical synthesis enables:
- Managing information overload
- Identifying patterns across reports
- Maintaining both detail and high-level view

### 5. Autonomous Over Supervised

**Bad**: Require human intervention at each step
**Good**: Orchestrator makes decisions based on criteria

**Why**: Autonomy enables:
- Scaling to large problems
- Running overnight or asynchronously
- Consistent decision-making

---

## Success Criteria for Framework Application

A successful application should achieve:

✅ **No human intervention** needed after initial setup
✅ **Automatic recovery** from stuck/failed teammates
✅ **Visible progress** at any point in time
✅ **Adaptive execution** based on findings
✅ **Hierarchical output** that manages complexity
✅ **Complete coverage** of the problem space
✅ **Clear termination** when convergence achieved

---

## Limitations & Future Work

**Current Limitations**:
1. **No true autonomous loop** - relies on user prompts or /loop mode
   - **Critical**: There is NO automatic background process that runs monitoring
   - Monitoring happens only when: user prompts, teammate messages, or /loop mode
   - Framework documents the ideal (continuous monitoring), reality requires triggers
   - **Workaround**: User must prompt "check progress" every 3-5 minutes
2. **File-based monitoring only** - can't inspect teammate internal state
3. **Manual synthesis design** - orchestrator must design synthesis structure

**Future Enhancements**:
1. **Self-monitoring orchestrator** that runs without prompts
2. **Richer progress signals** from teammates (% complete, checkpoint messages)
3. **Automated synthesis planning** based on output analysis
4. **Cost/time optimization** - predict task durations, optimize team size
5. **Learning from failures** - adjust granularity based on past stuck tasks

---

## Summary

This framework provides a **reusable pattern** for large-scale work:

1. **Wave-based analysis** with dynamic adaptation
2. **Hierarchical synthesis** to manage complexity  
3. **Autonomous orchestration** with active monitoring
4. **Granular task design** for progress visibility
5. **Automatic recovery** from failures

**Key Insight**: The orchestrator is not a passive manager but an **active agent** that continuously monitors, adapts, and recovers - enabling truly autonomous execution of complex, multi-phase work.


---

# Quick Start Guide


This guide shows how to apply the [Agent Team Framework](./agent-team-framework.md) to your own problems.

---

## When to Use This Pattern

Use agent teams when your work has these characteristics:

✅ **Large scope** - too much for one agent or one session  
✅ **Decomposable** - can be broken into parallel independent tasks  
✅ **Iterative** - each phase's output informs what comes next  
✅ **Needs synthesis** - many outputs need to be combined into one  
✅ **Long-running** - takes hours, not minutes  

**Examples**:
- Analyze a large codebase across multiple subsystems
- Research project with many sources/questions
- Data analysis with multiple datasets
- Documentation generation from many components
- Code review across many files/PRs

---

## Quick Start: 5 Steps

### Step 1: Define Your Problem

Answer these questions:

```markdown
## Problem Definition

**Name**: What are you calling this project?
**Goal**: What do you want to achieve? (1-2 sentences)
**Inputs**: What are you analyzing/working with?
**Success Criteria**: How do you know when you're done?
**Estimated Scope**: How many tasks/outputs do you expect?
```

**Example** (from our codebase analysis):
```markdown
**Name**: Claude Code CLI Codebase Analysis
**Goal**: Understand architecture: launch, REPL loop, background processes, alt modes
**Inputs**: Extracted source code (~800KB main.tsx, 17 subsystems)
**Success Criteria**: Comprehensive report with code references, flow diagrams, integration points
**Estimated Scope**: 15-20 subsystem analyses
```

### Step 2: Design Wave 1 Tasks

Break your problem into **parallel, independent tasks** that cover the entire space.

**Template**:
```markdown
## Wave 1 Tasks

Task N: <Short Title>
- Subject: <Clear action + scope>
- Scope: <What to analyze/investigate>
- Focus: <Specific points to address>
- Key Sources: <Where to look>
- Deliverable: <What artifact to produce>
- Should Flag: <What needs follow-up>
- Output: <File path>
- Duration: <Estimate: 10-20 min>
```

**Decomposition Strategies**:

| Problem Type | Decompose By | Example Tasks |
|---|---|---|
| Codebase | Subsystems, phases, features | "Analyze setup() function", "Analyze REPL loop" |
| Research | Questions, topics, sources | "Research authentication patterns", "Survey rate limiting" |
| Data Analysis | Datasets, dimensions, periods | "Analyze Q1 sales", "Analyze user cohorts" |
| Documentation | Components, APIs, flows | "Document Auth API", "Document deployment flow" |

**Rules**:
- Each task = 10-20 minutes max
- Tasks must be **independent** (no blockedBy in Wave 1)
- Tasks must produce **verifiable output** (file exists, specific content)
- Include recommendations section in each task

### Step 3: Create Orchestrator Prompt

Use this template to start your orchestrator session:

```markdown
I need you to orchestrate an agent team analysis using the pattern in agent-team-framework.md.

## Problem
<paste your problem definition from Step 1>

## Wave 1 Tasks
<paste your task list from Step 2>

## Your Job as Orchestrator

1. Create team: `<team-name>`
2. Create all Wave 1 tasks using TaskCreate
3. Spawn 3-5 analyst teammates
4. Monitor actively every 3-5 minutes:
   - Check TaskList for progress
   - Check output files exist and are growing
   - Recover stuck teammates (reassign after 10 min with no output)
   - Direct-assign idle teammates if pending work exists
5. After Wave 1 completes:
   - Review outputs and recommendations
   - Decide: Wave 2 needed OR proceed to synthesis?
6. Execute hierarchical synthesis (3 stages):
   - Stage 1: Create summaries (grouped by category)
   - Stage 2: Identify clusters and patterns
   - Stage 3: Compile final report (9 section-level tasks, NOT one monolithic task)
7. Shutdown all teammates when complete

## Key Principles
- Active monitoring, not passive waiting
- Granular tasks (10-20 min each, verifiable output)
- Recover automatically (stuck >10min → reassign)
- Break synthesis into section-level tasks

Start now: Create the team and Wave 1 tasks.
```

### Step 4: Monitor & Adapt

As orchestrator works, you can:

**Check Progress**:
```bash
# See task status
ls -la outputs/wave-1/

# Check task list
# (orchestrator will show this periodically)

# Verify outputs
wc -l outputs/wave-1/*.md
```

**Intervene if Needed**:
- If orchestrator isn't monitoring: remind it to check progress
- If multiple teammates stuck: suggest respawning fresh teammates
- If synthesis task is monolithic: remind to break into sections

**Trust the Process**:
- Teammates will claim and complete tasks autonomously
- Orchestrator will recover stuck teammates automatically
- Let it run - don't micromanage

### Step 5: Review & Iterate

After first wave completes, orchestrator will ask:

**"Should we proceed to Wave 2 or Synthesis?"**

Review the recommendation extraction to decide:
- High-priority gaps → Wave 2
- Sufficient coverage → Synthesis

Synthesis produces:
- Summaries (bite-size extracts)
- Synthesis plan (clustering strategy)
- Final report (comprehensive narrative)

---

## Common Patterns

### Pattern 1: Codebase Analysis

```markdown
Wave 1: Analyze subsystems (one task per subsystem)
Review: Extract complexity flags and missing pieces
Wave 2: Deep-dive on flagged subsystems (optional)
Synthesis:
  - Summarize by phase/category
  - Identify interaction clusters
  - Compile report with architecture flow + sections
```

### Pattern 2: Research Project

```markdown
Wave 1: Research questions (one task per question)
Review: Identify gaps in coverage
Wave 2: Fill gaps with targeted research
Synthesis:
  - Summarize by topic
  - Identify cross-cutting themes
  - Compile report with findings + recommendations
```

### Pattern 3: Multi-Dataset Analysis

```markdown
Wave 1: Analyze datasets (one task per dataset)
Review: Check for anomalies, patterns
Wave 2: Cross-dataset comparisons
Synthesis:
  - Summarize per dataset
  - Identify correlations
  - Compile report with insights + visualizations
```

### Pattern 4: Documentation Generation

```markdown
Wave 1: Document components (one task per component)
Review: Check coverage, identify integration points
Wave 2: Document integrations between components
Synthesis:
  - Summarize by system/layer
  - Create navigation structure
  - Compile docs with overview + detailed sections
```

---

## Troubleshooting

### Problem: Teammates Not Claiming Tasks

**Symptoms**: Tasks sitting pending, teammates idle

**Solution**:
```
Orchestrator should:
1. Detect idle teammates with pending work
2. Send direct assignment (not broadcast)
3. If no response in 2 min, shutdown & respawn
```

### Problem: Tasks Taking Too Long

**Symptoms**: Task in_progress >20 min

**Causes**:
- Task too large (wrong granularity)
- Teammate stuck in loop
- Task ill-defined

**Solution**:
```
1. Reassign to fresh teammate
2. If happens repeatedly, break task into smaller pieces
3. Clarify task description
```

### Problem: Synthesis Task is Monolithic

**Symptoms**: One "compile final report" task running >30 min

**Solution**:
```
DELETE the monolithic task, replace with:
- Task 1: Write Section 1
- Task 2: Write Section 2
- ...
- Task N: Write Section N

Each section = one task (10-15 min)
```

### Problem: No Progress Visibility

**Symptoms**: Don't know if teammates are working

**Solution**:
```
Check output files:
ls -lh outputs/wave-1/
wc -l outputs/wave-1/*.md
grep "^## " outputs/wave-1/*.md

If no files created after 10 min → reassign tasks
```

### Problem: Information Overload in Synthesis

**Symptoms**: Synthesis trying to read 50+ reports at once

**Solution**:
```
Use hierarchical synthesis:
1. Create summaries first (bite-size)
2. Read summaries (manageable)
3. Compile final report section-by-section
```

---

## Best Practices

### ✅ Do

- **Active monitoring** - check every 3-5 minutes
- **Granular tasks** - 10-20 min each, verifiable output
- **Direct assignment** - send tasks to specific teammates, not broadcasts
- **Break monoliths** - synthesis = many section tasks, not one big task
- **Verify progress** - check files exist and are growing
- **Recover early** - reassign after 10 min with no output
- **Document in task metadata** - output_path, wave, category

### ❌ Don't

- **Passive waiting** - don't just wait for notifications
- **Monolithic tasks** - avoid "do everything" tasks
- **Ignore idle teammates** - detect and assign work
- **Trust without verification** - always check output files
- **Batch reassignments** - one task at a time
- **Skip synthesis hierarchy** - always summarize before compiling

---

## Advanced: Customizing the Pattern

### Custom Wave Logic

You can customize the wave transition logic:

```typescript
// Example: Always do exactly 2 waves
function decide_next_phase(wave_num, outputs) {
  if (wave_num == 1) {
    return design_wave_2(outputs)
  } else {
    return proceed_to_synthesis()
  }
}

// Example: Continue until no recommendations
function decide_next_phase(wave_num, outputs) {
  recs = extract_recommendations(outputs)
  if (recs.length > 0) {
    return design_next_wave(recs)
  } else {
    return proceed_to_synthesis()
  }
}

// Example: Max 3 waves regardless
function decide_next_phase(wave_num, outputs) {
  if (wave_num >= 3) {
    return proceed_to_synthesis()
  }
  recs = extract_high_priority(outputs)
  if (recs.length > 0) {
    return design_next_wave(recs)
  } else {
    return proceed_to_synthesis()
  }
}
```

### Custom Synthesis Structure

You can customize how synthesis is structured:

```markdown
# Option 1: By Category (default)
Section 1: Category A findings
Section 2: Category B findings
...

# Option 2: By Priority
Section 1: Critical findings
Section 2: Important findings
Section 3: Recommendations

# Option 3: By Timeline
Section 1: Historical context
Section 2: Current state
Section 3: Future roadmap

# Option 4: By Audience
Section 1: Executive summary
Section 2: Technical details
Section 3: Implementation guide
```

### Custom Monitoring Intervals

Adjust based on task duration:

```typescript
// Short tasks (5-10 min) - check frequently
monitor_interval = 2 minutes

// Medium tasks (10-20 min) - standard
monitor_interval = 3-5 minutes

// Long tasks (20-30 min) - check less often
monitor_interval = 5-10 minutes

// Rule: check_interval = task_duration / 3
```

---

## Success Checklist

Before you start, verify:

- [ ] Problem clearly defined with success criteria
- [ ] Wave 1 tasks are independent and parallel
- [ ] Each task has estimated duration 10-20 min
- [ ] Each task specifies output file path
- [ ] Orchestrator understands active monitoring
- [ ] Recovery procedures documented

During execution, verify:

- [ ] Teammates are claiming tasks autonomously
- [ ] Output files being created and growing
- [ ] Orchestrator checking progress every 3-5 min
- [ ] Stuck teammates recovered within 10-15 min
- [ ] Idle teammates assigned work directly

After completion, verify:

- [ ] All Wave 1 tasks completed
- [ ] Recommendations reviewed and addressed
- [ ] Synthesis completed hierarchically
- [ ] Final output meets success criteria
- [ ] All teammates shut down gracefully

---

## Resources

- [Full Framework Documentation](./agent-team-framework.md)
- [Example: Codebase Analysis Plan](./codebase-analysis-plan.md)
- [Example: Synthesis Plan](./synthesis-plan.md)
- [Example: Final Report](./codebase-report.md)

---

## Getting Help

If you encounter issues:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review the [Framework Documentation](./agent-team-framework.md)
3. Examine the example execution in this repo
4. File an issue describing the problem pattern

**Common Issues**:
- Teammates not claiming work → Check idle detection
- Tasks stuck → Check monitoring interval and reassignment
- Synthesis too slow → Check task granularity
- No progress visibility → Check file verification

Good luck orchestrating your agent team! 🚀
