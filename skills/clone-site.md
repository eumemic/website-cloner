---
name: clone-site
description: |
  Orchestrate iterative discovery of a target website, generating Ralph specs
  that can be used to build a clone. Handles setup, discovery waves, refinement,
  and transition to building.
triggers:
  - clone
  - reverse-engineer
  - copy this site
  - clone-site
---

# Clone Site Skill

This skill orchestrates the reverse-engineering of a target website into Ralph specs that can be used to build a clone.

## State Detection

When invoked, determine the current workflow state by examining project files:

### Step 1: Check Config Existence

```
Glob for: .ralph/discovery-config.yaml
```

If config does NOT exist → **Setup** state

### Step 2: Count Specs

```
Glob for: specs/*.md and specs/components/*.md
Count total spec files found
```

### Step 3: Read Spec Statuses

For each spec file found:
```
Read YAML frontmatter
Extract 'status' field (draft | complete)
```

### State Matrix

| Config Exists | Specs Exist | Status Distribution | State |
|--------------|-------------|---------------------|-------|
| No           | -           | -                   | **Setup** |
| Yes          | No or ≤1    | -                   | **Initial Discovery** |
| Yes          | Yes (>1)    | Has any draft       | **Iterating** |
| Yes          | Yes (>1)    | All complete        | **Ready to Build** |

### Determining State

1. **Setup**: No `.ralph/discovery-config.yaml` exists
   - User needs to configure target URL, scope, and fidelity

2. **Initial Discovery**: Config exists but no specs (or only the initial entry point spec)
   - Ready to dispatch first discovery wave

3. **Iterating**: Config exists and specs exist, with at least one having `status: draft`
   - Discovery is ongoing; present progress and offer options

4. **Ready to Build**: Config exists and all specs have `status: complete`
   - All discovery complete; ready to transition to Ralph plan/build

Once state is determined, proceed to the corresponding phase instructions below.

---

## Setup Phase

When in **Setup** state (no config exists), execute the following workflow:

### Step 1: Greet and Explain

Introduce the cloning workflow to the user:

```
I'll help you clone a website by reverse-engineering it into specs. Here's how this works:

1. **Setup** - We'll configure what to clone and how detailed to be
2. **Discovery** - I'll explore the target site and generate specs
3. **Refinement** - You review specs and answer any questions
4. **Build** - When specs are ready, we'll use Ralph to build the clone

Let's get started with setup.
```

### Step 2: Gather Target URL

If the user's initial message included a URL, extract it. Otherwise, use AskUserQuestion:

```yaml
questions:
  - question: "What's the URL of the website you want to clone?"
    header: "Target URL"
    options:
      - label: "I'll type it"
        description: "Enter the full URL (e.g., https://example.com)"
    multiSelect: false
```

The user will typically select "Other" and provide the URL directly.

### Step 3: Gather Scope

Ask about scope using natural language:

```yaml
questions:
  - question: "What parts of the site should I focus on?"
    header: "Scope"
    options:
      - label: "Entire site"
        description: "Clone all pages and features"
      - label: "Specific section"
        description: "Focus on particular pages or features (I'll describe)"
      - label: "Single page"
        description: "Clone just one page"
    multiSelect: false
```

If user selects "Specific section" or "Other", ask them to describe what to include/exclude.

### Step 4: Gather Fidelity Level

```yaml
questions:
  - question: "How closely should the clone match the original?"
    header: "Fidelity"
    options:
      - label: "Functional (Recommended)"
        description: "Same features and flows, visuals can differ"
      - label: "Pixel-perfect"
        description: "Exact visual match: colors, spacing, fonts"
      - label: "Structural"
        description: "Same information architecture, use your own design"
    multiSelect: false
```

Default to "functional" if user doesn't have strong preference.

### Step 5: Gather Backend Scope

```yaml
questions:
  - question: "Should I also infer backend behavior from the site?"
    header: "Backend"
    options:
      - label: "Frontend only (Recommended)"
        description: "Capture visuals and client-side interactions"
      - label: "Full-stack"
        description: "Also infer API patterns, data models, server behavior"
    multiSelect: false
```

Default to "frontend-only" if user doesn't have strong preference.

### Step 6: Create Configuration File

Write `.ralph/discovery-config.yaml` with collected values:

```yaml
target:
  url: <collected_url>
  scope: |
    <user's scope description>
  url_patterns:  # optional, add if user provided specific patterns
    include: []
    exclude: []

fidelity: <functional|pixel-perfect|structural>  # default: functional

backend_scope: <frontend-only|full-stack>  # default: frontend-only

workers: 1  # default, can be increased later
```

Ensure the `.ralph/` directory exists before writing.

### Step 7: Create Initial Entry Point Spec

Create a draft spec for the target URL's entry point. Derive the page name from the URL path (or use "homepage" for root).

Write to `specs/<page-name>.md`:

```markdown
---
status: draft
source_urls:
  - <target_url>
---

# <Page Name>

Entry point for discovery. This spec will be populated when the discovery agent explores the target site.

## Open Questions

- What is the overall structure and purpose of this page?
```

Ensure the `specs/` directory exists before writing.

### Step 8: Transition

After setup is complete, inform the user and transition to Initial Discovery:

```
Setup complete! I've created:
- `.ralph/discovery-config.yaml` with your preferences
- `specs/<page-name>.md` as the starting point for exploration

Ready to start exploring the target site?
```

Then proceed to **Initial Discovery** state.

---

## Authentication Handling

If the target site requires authentication (login walls, protected content), handle it as follows:

### Detecting Authentication Requirements

Authentication may be needed when:
- The target URL redirects to a login page
- User explicitly mentions the site requires login
- Discovery agent reports encountering auth walls

### Procedure

1. **Inform the User**

   When authentication is required, explain the situation:

   ```
   The target site appears to require authentication. To explore protected content:

   1. Open your browser and navigate to the target site
   2. Log in with your credentials
   3. Let me know when you're logged in

   The discovery agent will then explore as your authenticated session.
   ```

2. **Wait for Confirmation**

   Use AskUserQuestion to confirm:

   ```yaml
   questions:
     - question: "Have you logged in to the target site in your browser?"
       header: "Auth Status"
       options:
         - label: "Yes, I'm logged in"
           description: "Continue with discovery using my authenticated session"
         - label: "Skip protected content"
           description: "Only explore publicly accessible pages"
         - label: "Not yet"
           description: "Give me a moment to log in"
       multiSelect: false
   ```

3. **Proceed Based on Response**

   - **"Yes, I'm logged in"**: Continue with discovery. The chrome MCP will use the authenticated browser session.
   - **"Skip protected content"**: Update the scope to note that authenticated content is excluded, then proceed with public pages only.
   - **"Not yet"**: Wait and ask again when the user is ready.

### Notes

- Authentication state persists in the browser session across discovery waves
- If the session expires during long discovery, the user may need to re-authenticate
- Never ask for or store user credentials - always rely on manual browser login

---

## Initial Discovery Phase

When in **Initial Discovery** state (config exists but no specs or only the entry point spec), execute the following workflow:

### Step 1: Read Configuration

Load the discovery configuration:

```
Read: .ralph/discovery-config.yaml
Extract: target.url, target.scope, fidelity, backend_scope, workers
```

### Step 2: Confirm with User Before First Wave

Before dispatching the first discovery wave, confirm the configuration with the user:

```
I'm ready to start exploring the target site. Here's what I have configured:

**Target:** <target.url>
**Scope:** <target.scope>
**Fidelity:** <fidelity>
**Backend:** <backend_scope>

Does this look correct?
```

Use AskUserQuestion for confirmation:

```yaml
questions:
  - question: "Ready to start exploring the target site?"
    header: "Confirm"
    options:
      - label: "Yes, start exploring"
        description: "Begin discovery with the current configuration"
      - label: "Adjust settings"
        description: "Let me make changes first"
    multiSelect: false
```

If user selects "Adjust settings", return to the relevant setup step to collect new values and update the config file.

### Step 3: Check Authentication Requirements

Before dispatching agents, ensure the user is authenticated if the target site requires login:

```
Does the target site require you to be logged in to access the content you want to clone?
```

Use AskUserQuestion:

```yaml
questions:
  - question: "Does the target site require authentication?"
    header: "Auth Check"
    options:
      - label: "No, it's public"
        description: "The site content is publicly accessible"
      - label: "Yes, I'm already logged in"
        description: "I've logged into the site in my browser"
      - label: "Yes, let me log in first"
        description: "I need to authenticate before we proceed"
    multiSelect: false
```

- **"No, it's public"**: Proceed directly to agent dispatch
- **"Yes, I'm already logged in"**: Proceed to agent dispatch (chrome MCP will use the authenticated session)
- **"Yes, let me log in first"**: Wait, then ask again when user confirms they've logged in

### Step 4: Dispatch Discovery Agent to Entry Point

Dispatch the discovery agent to explore the entry point URL using the Task tool:

```
Use Task tool with:
  subagent_type: discovery-agent
  prompt: |
    Explore the target website starting from the entry point.

    ## Configuration
    - Target URL: <target.url>
    - Scope: <target.scope>
    - Fidelity: <fidelity>
    - Backend scope: <backend_scope>

    ## Starting Spec
    Read and explore: specs/<entry-point-spec>.md

    ## Instructions
    Follow the discovery agent methodology to:
    1. Navigate to the source URL
    2. Observe and document the page
    3. Discover linked pages within scope
    4. Create draft specs for discovered pages
    5. Return an exploration summary
```

### Step 5: Present Exploration Summary

After the agent completes, present the results to the user:

```
## First Exploration Complete

**Visited:** <count> pages
**Specs Updated:** <count>
**New Specs Created:** <count>
**Components Identified:** <count>

### Discovered Pages
- <list of new draft specs created>

### Open Questions
- <list of questions from exploration>

Ready to continue exploring, or would you like to review what we've found?
```

### Step 6: Transition to Iterating

After presenting the summary, transition to the **Iterating** state:

- If there are draft specs to explore → continue with Iterating phase
- If user wants to review → pause for their input
- If all specs are already complete (unlikely after first wave) → transition to Ready to Build

---

## Exploration Wave Dispatch

When executing an exploration wave (used in both Initial Discovery and Iterating phases):

### Step 1: Select Draft Specs to Explore

Identify all specs that need exploration:

```
Glob for: specs/*.md and specs/components/*.md
For each file:
  Read YAML frontmatter
  Include if status: draft
```

Prioritize specs with fewer open questions (more likely to complete in this wave).

### Step 2: Read Worker Configuration

Load worker count from config:

```
Read: .ralph/discovery-config.yaml
Extract: workers (default: 1)
```

### Step 3: Partition Specs for Multiple Workers

If `workers > 1`, partition the draft specs across agents:

```
draft_specs = [list of all draft spec file paths]
specs_per_worker = ceil(draft_specs.length / workers)

For worker i (0 to workers-1):
  assigned_specs[i] = draft_specs.slice(i * specs_per_worker, (i+1) * specs_per_worker)
```

If `workers = 1`, assign all draft specs to a single agent.

### Step 4: Dispatch Discovery Agent(s)

For each worker, use the Task tool to dispatch a discovery-agent:

```
Use Task tool with:
  subagent_type: discovery-agent
  prompt: |
    Explore the assigned specs for the target website.

    ## Configuration
    - Target URL: <target.url>
    - Scope: <target.scope>
    - URL patterns include: <target.url_patterns.include>
    - URL patterns exclude: <target.url_patterns.exclude>
    - Fidelity: <fidelity>
    - Backend scope: <backend_scope>

    ## Assigned Specs
    Explore the following draft specs in order:
    <list each assigned spec path>

    ## Known Components
    These components have already been identified:
    <list specs/components/*.md files>

    ## Instructions
    Follow the discovery agent methodology to:
    1. Navigate to each spec's source_urls
    2. Observe and document the page according to fidelity level
    3. Discover linked pages within scope
    4. Create draft specs for discovered pages
    5. Update spec status when complete
    6. Return an exploration summary
```

If dispatching multiple workers in parallel, invoke multiple Task tools simultaneously.

### Step 5: Collect Results

After agent(s) complete:

1. **Re-read all specs** to capture updates made by agents
2. **Compile changes**:
   - Count specs with status changed from draft to complete
   - Count new spec files created
   - Count new component specs identified
   - Collect all open questions from specs

The exploration summary is then presented to the user (see Progress Summary section below).

---

## Progress Summary Generation

After each exploration wave completes, generate a progress summary for the user following this procedure:

### Step 1: Re-read All Specs

After agents finish, read the current state of all specs:

```
Glob for: specs/*.md and specs/components/*.md
For each file:
  Read full content
  Parse YAML frontmatter (status, source_urls)
  Extract any "## Open Questions" section content
```

### Step 2: Diff Against Pre-Wave State

Compare current state against what existed before the wave:

1. **New specs**: Files that didn't exist before the wave started
2. **Updated specs**: Files with modified content (additional sections, more details)
3. **Completed specs**: Files where status changed from `draft` to `complete`
4. **New components**: Files in `specs/components/` that didn't exist before

Track which specs were visited by checking if their content grew or status changed.

### Step 3: Compile Summary Data

Aggregate the following metrics:

```
pages_visited = count of specs whose content changed
specs_updated = count of specs with new content (excluding status-only changes)
new_specs_created = count of new spec files
components_identified = count of new files in specs/components/
total_draft = count of specs with status: draft
total_complete = count of specs with status: complete
```

Collect all open questions across specs:

```
For each spec file:
  If has "## Open Questions" section:
    Extract each question (lines starting with "- ")
    Record as: {spec_file, question_text}
```

### Step 4: Format Progress Summary

Present the summary to the user using this template:

```
## Exploration Progress

**Specs:** <total_draft + total_complete> total (<total_draft> draft, <total_complete> complete)
**Components:** <count of files in specs/components/> identified

### Wave Results
- **Pages visited:** <pages_visited>
- **Specs updated:** <specs_updated>
- **New specs created:** <new_specs_created>
- **Components identified:** <components_identified>

### Updated Specs
<For each updated spec, show filename and brief description of what changed>
- <spec-name>.md - <brief change description>

### New Specs
<For each new spec created this wave>
- specs/<spec-name>.md (draft)
- specs/components/<component-name>.md (draft)

### Open Questions (<total_question_count> total)
<Group questions by spec file>
- <spec-name>.md: <question text>
- <spec-name>.md: <question text>
...

<If there are many questions, show first 5-10 and note "... and N more">
```

### Handling Empty Results

If a wave produces no changes (agents found nothing new):

```
## Exploration Progress

The discovery agents explored the assigned pages but found no new content to document.

**Possible reasons:**
- All observable content has already been captured
- The pages may require different interactions to reveal content
- The scope may need adjustment

Would you like to:
- Run another wave with different specs
- Review and edit existing specs
- Adjust the scope configuration
- Mark specs as complete and proceed to building
```

### Handling Partial Results

If some agents encountered issues:

```
## Exploration Progress

**Specs:** <counts as above>

### Wave Results
<standard metrics>

### Issues Encountered
- <spec-name>.md: <issue description, e.g., "Page failed to load", "Auth wall encountered">

<If auth wall>
The target site requires authentication. Please log in to the site in your browser and let me know when ready to continue.

<If page errors>
Some pages couldn't be fully explored. Would you like to retry or skip them?
```

---

## Iterating Phase

When in **Iterating** state (config exists, specs exist, at least one has `status: draft`), execute the following workflow:

### Step 1: Read and Categorize Specs

Load all specs and categorize by status:

```
Glob for: specs/*.md and specs/components/*.md
For each file:
  Read YAML frontmatter
  Categorize as draft or complete
  Extract any "## Open Questions" section
```

### Step 2: Present Status Summary

Show the user the current discovery status:

```
## Discovery Status

**Specs:** <total> total (<draft_count> draft, <complete_count> complete)
**Components:** <component_count> identified

### Open Questions (<question_count>)
<Group by spec file, show first 5-10>
- <spec-name>.md: <question text>
- <spec-name>.md: <question text>
...

Ready to run another exploration wave, or would you like to address some questions first?
```

### Step 3: Offer User Interaction Options

Present options using AskUserQuestion:

```yaml
questions:
  - question: "What would you like to do next?"
    header: "Next Action"
    options:
      - label: "Run another wave"
        description: "Dispatch agents to explore remaining draft specs"
      - label: "Answer questions"
        description: "Help resolve open questions from the specs"
      - label: "Review specs"
        description: "Edit specs directly or mark them complete"
      - label: "Finish discovery"
        description: "Mark remaining specs complete and proceed to building"
    multiSelect: false
```

### Step 4: Handle User Response

Based on user selection, execute the appropriate action:

#### Option: "Run another wave"

1. Proceed to **Exploration Wave Dispatch** (documented above)
2. After wave completes, return to Step 1 of Iterating Phase

#### Option: "Answer questions"

1. Present the full list of open questions grouped by spec
2. For each question the user answers conversationally:
   - Read the relevant spec file
   - Update the spec content based on the user's answer
   - Remove the answered question from the "## Open Questions" section
   - If user says a question is "not important" or "doesn't matter":
     - Remove the question
     - Optionally note as intentionally skipped in spec content
3. After addressing questions, return to Step 1 to show updated status

#### Option: "Review specs"

1. Ask which spec the user wants to review:
   ```yaml
   questions:
     - question: "Which spec would you like to review?"
       header: "Select Spec"
       options:
         - label: "Show me the list"
           description: "Display all specs with their status"
       multiSelect: false
   ```
2. The user can:
   - Select "Other" and name a specific spec file
   - Edit specs directly in their editor
   - Tell you changes to make conversationally
3. When user updates a spec:
   - If they resolve all open questions and document all observable content → mark `status: complete`
   - If they want to mark it complete manually → update the frontmatter
4. After review, return to Step 1 to show updated status

#### Option: "Finish discovery"

1. Confirm the user wants to proceed:
   ```yaml
   questions:
     - question: "Are you sure you want to finish discovery? Remaining draft specs will be marked complete."
       header: "Confirm"
       options:
         - label: "Yes, finish"
           description: "Mark all drafts complete and proceed to building"
         - label: "No, continue"
           description: "Keep iterating on specs"
       multiSelect: false
   ```
2. If confirmed:
   - For each spec with `status: draft`:
     - Update frontmatter to `status: complete`
     - Add note if there were unresolved questions: "Note: Marked complete by user request"
   - Transition to **Ready to Build** state

### Handling User Guidance Mid-Flow

The user may provide guidance at any point during iteration. Handle these inputs:

#### Scope Changes

When user requests scope changes like "Ignore the blog section" or "Go deeper on checkout":

1. Read current `.ralph/discovery-config.yaml`
2. Update the relevant fields:
   - For exclusions: Add patterns to `target.url_patterns.exclude`
   - For inclusions: Add patterns to `target.url_patterns.include`
   - For natural language changes: Update `target.scope`
3. Write updated config
4. Re-evaluate existing specs:
   - If newly excluded URLs match existing specs → note that those specs are now out of scope
   - If newly included areas → create draft specs for those entry points
5. Inform user of changes and return to status summary

Example transformations:
- "Ignore the blog" → Add `/blog/*` to exclude patterns
- "Focus on checkout" → Update scope description, prioritize checkout-related drafts
- "That's enough for the header" → Mark header component spec as complete

#### Fidelity Adjustments

When user requests fidelity changes:

1. If global change ("Make everything pixel-perfect"):
   - Update `fidelity` in config
   - Note that future waves will use new fidelity
2. If per-spec change ("Match colors exactly on homepage"):
   - Add a note to that specific spec indicating higher fidelity requirement
   - Discovery agents will respect per-spec overrides

#### Direct Answers to Questions

When user answers an open question directly in conversation:

1. Identify which spec contains the question
2. Read the spec file
3. Update the relevant section with the user's answer
4. Remove the question from "## Open Questions"
5. Acknowledge the update: "Updated <spec-name>.md with your answer about <topic>"

---

## Ready to Build Transition

When in **Ready to Build** state (all specs have `status: complete`, or user selects "Finish discovery"), execute the following workflow:

### Step 1: Confirm User Is Ready

Present the build confirmation to the user:

```
## Ready to Clone

All specs are now complete. Here's what will be built:

**Pages:** <count of page specs>
**Components:** <count of component specs>
**Fidelity:** <fidelity from config>
**Backend:** <backend_scope from config>

This will generate a Ralph implementation plan and begin building the clone.
```

Use AskUserQuestion for confirmation:

```yaml
questions:
  - question: "Ready to proceed with building the clone?"
    header: "Build"
    options:
      - label: "Yes, generate plan"
        description: "Create implementation plan from specs"
      - label: "Review specs first"
        description: "Let me review the specs before building"
      - label: "Run more discovery"
        description: "Continue exploring the target site"
    multiSelect: false
```

### Step 2: Handle User Response

Based on user selection:

#### Option: "Yes, generate plan"

Proceed to Step 3 (Invoke Ralph Plan).

#### Option: "Review specs first"

1. List all specs with their status
2. Allow user to read/edit specs
3. When user is satisfied, return to Step 1

#### Option: "Run more discovery"

1. Transition back to **Iterating** state
2. User can run additional exploration waves
3. When satisfied, they can return to finish discovery

### Step 3: Invoke Ralph Plan

Run the Ralph planning workflow to generate implementation tasks:

```
I'll now generate an implementation plan from the specs.

This will:
1. Analyze all page and component specs
2. Identify dependencies and build order
3. Create specific implementation tasks
4. Estimate effort based on fidelity level

Generating plan...
```

Invoke the Ralph plan skill:

```
Use Skill tool with:
  skill: plan
```

The Ralph planning loop will:
- Read all specs in `specs/` directory
- Generate `.ralph/IMPLEMENTATION_PLAN.md`
- Create prioritized work items

### Step 4: Present Plan for Approval

After Ralph plan completes, present the plan summary:

```
## Implementation Plan Ready

Ralph has generated an implementation plan:

**Total tasks:** <count from plan>
**Priority 1 items:** <count>
**Estimated components:** <list>

Would you like to review the plan before building?
```

Use AskUserQuestion:

```yaml
questions:
  - question: "How would you like to proceed with the implementation plan?"
    header: "Plan Review"
    options:
      - label: "Start building"
        description: "Begin implementation with the current plan"
      - label: "Review plan"
        description: "Let me read through the plan first"
      - label: "Adjust specs"
        description: "Go back and modify specs before building"
    multiSelect: false
```

### Step 5: Handle Plan Response

Based on user selection:

#### Option: "Start building"

Proceed to Step 6 (Invoke Ralph Build).

#### Option: "Review plan"

1. Read and display `.ralph/IMPLEMENTATION_PLAN.md`
2. Allow user to request changes or additions
3. If changes needed, re-run planning or manually update plan
4. When satisfied, return to Step 4

#### Option: "Adjust specs"

1. Transition back to **Iterating** state
2. User can modify specs
3. When done, re-run planning from Step 3

### Step 6: Invoke Ralph Build

Begin the Ralph build workflow:

```
Starting implementation...

Ralph will work through each task in the plan, building the clone
according to your specs. You'll be asked for clarification if
questions arise during implementation.
```

Invoke the Ralph build skill:

```
Use Skill tool with:
  skill: build
```

### Step 7: Building Proceeds

From this point, the normal Ralph workflow takes over:

1. Ralph works through implementation tasks
2. User can review and provide feedback
3. Tests are written and run
4. Changes are committed incrementally
5. Build continues until plan is complete

The clone-site skill's orchestration is complete once building begins. Users can:
- Interact normally with Ralph during building
- Run `/clone-site` again if they need to add more specs mid-build
- The cloned site takes shape as Ralph implements each spec

---

## Workflow Resumability

The clone-site workflow is designed to be fully resumable across sessions. Users can close Claude Code at any point and return later to continue from where they left off.

### State Persistence Model

All workflow state is stored in files, not in memory:

1. **Configuration State** - `.ralph/discovery-config.yaml`
   - Target URL, scope, fidelity, backend scope, worker count
   - Persists all setup decisions

2. **Discovery State** - `specs/*.md` and `specs/components/*.md`
   - Each spec file contains its own status (draft/complete)
   - Open questions are recorded within each spec
   - Progress is captured incrementally as agents update files

### No In-Memory Session State

The orchestrator maintains no session-specific state:
- No conversation history is required to determine workflow state
- No tracking of "current wave" or "previous actions"
- Each invocation starts fresh by reading files

### Resumption Behavior

When the `/clone-site` skill is invoked:

1. **Automatic State Detection**
   - Check if `.ralph/discovery-config.yaml` exists
   - Count and read all spec files
   - Parse status from each spec's frontmatter
   - Determine current state using the State Matrix

2. **Seamless Continuation**
   - **No config** → Resume at Setup phase
   - **Config, no/few specs** → Resume at Initial Discovery
   - **Config, draft specs exist** → Resume at Iterating phase
   - **Config, all specs complete** → Resume at Ready to Build

3. **Context Recovery**
   - User doesn't need to explain what happened before
   - All relevant context is reconstructed from files
   - Open questions, progress metrics, scope - all re-read

### User Experience

The resumability model means:

```
# Session 1
User: Clone example.com
→ Completes setup
→ Runs first discovery wave
→ Reviews progress, answers 2 questions
→ Closes Claude Code

# Session 2 (hours or days later)
User: /clone-site
→ Orchestrator reads files, detects "Iterating" state
→ Shows current status: "8 draft specs, 4 complete, 3 open questions"
→ Offers options: run another wave, answer questions, etc.
→ Work continues seamlessly
```

### Supporting Long-Running Projects

This design supports cloning projects that span:
- Multiple work sessions
- Days or weeks of elapsed time
- Different team members (if files are in shared repo)

Each session picks up exactly where the previous one ended by reading the canonical state from files.

---

## Error Recovery

Handle errors that may occur during discovery. Each error type has a specific recovery procedure.

### Agent Timeout or Failure

When a discovery agent times out or fails to complete:

1. **Note Affected Specs**

   Identify which specs were being explored when the failure occurred:
   ```
   The discovery agent encountered an issue while exploring:
   - <list of spec files that were assigned to the agent>
   ```

2. **Present Partial Results**

   If the agent produced any updates before failing:
   - Read the affected spec files
   - Report any changes that were successfully made
   - List specs that weren't reached

3. **Offer Recovery Options**

   Use AskUserQuestion:
   ```yaml
   questions:
     - question: "The discovery agent didn't complete successfully. How would you like to proceed?"
       header: "Recovery"
       options:
         - label: "Retry failed specs"
           description: "Run another wave targeting the same specs"
         - label: "Skip and continue"
           description: "Mark affected specs as needing manual review"
         - label: "Review what we have"
           description: "See partial results before deciding"
       multiSelect: false
   ```

### Auth Wall Encountered

When the discovery agent encounters a login wall or authentication requirement:

1. **Stop Exploration Immediately**

   The agent should not attempt to bypass authentication.

2. **Notify User**

   ```
   The target site requires login at <URL where auth was encountered>.

   To continue exploring authenticated content:
   1. Open your browser and navigate to the target site
   2. Log in with your credentials
   3. Let me know when you're logged in

   The discovery agent will then use your authenticated browser session.
   ```

3. **Wait for User Confirmation**

   Use the authentication handling procedure documented in the Authentication Handling section.

4. **Resume from Same Point**

   After user confirms login:
   - Re-dispatch agent to the same specs that encountered the auth wall
   - Note in the spec that authentication was required to access this page

### Scope Ambiguity

When the agent is uncertain whether a page or feature is in scope:

1. **Note the Question**

   The agent should add to the spec's Open Questions section rather than making assumptions.

2. **Surface to User**

   During the progress summary, highlight scope questions:
   ```
   ### Scope Questions

   The discovery agent wasn't sure about these:
   - Should /products/deals be included? It seems related to products but might be a marketing page.
   - Is the blog sidebar considered part of the navigation component?
   ```

3. **Collect User Clarification**

   ```yaml
   questions:
     - question: "How should I handle these scope questions?"
       header: "Scope"
       options:
         - label: "Include them"
           description: "Explore these pages/features"
         - label: "Exclude them"
           description: "Skip these and note as out of scope"
         - label: "Let me decide each"
           description: "I'll answer each question individually"
       multiSelect: false
   ```

4. **Update Configuration**

   Based on user response:
   - Update `target.url_patterns` to clarify patterns
   - Update `target.scope` with natural language clarification
   - Remove the scope questions from Open Questions

### Browser Unavailable

When browser automation tools (Chrome MCP) are not available or fail to connect:

1. **Detect the Issue**

   Chrome MCP tools will fail if:
   - The Claude in Chrome extension is not installed
   - The extension is not running
   - No browser window is open
   - The connection is interrupted

2. **Inform User**

   ```
   Browser automation is required for website discovery but isn't available.

   Please ensure:
   1. The "Claude in Chrome" browser extension is installed
   2. Chrome/Chromium is running with the extension active
   3. The extension shows as connected

   Once the browser is ready, let me know to continue.
   ```

3. **Wait for User Confirmation**

   ```yaml
   questions:
     - question: "Is the browser ready for automation?"
       header: "Browser"
       options:
         - label: "Yes, try again"
           description: "Retry with browser automation"
         - label: "Help me set it up"
           description: "Show instructions for Chrome MCP setup"
         - label: "Continue without browser"
           description: "Write specs conversationally instead"
       multiSelect: false
   ```

4. **Handle Fallback**

   If user selects "Continue without browser":
   - Switch to conversational spec writing mode
   - Ask user to describe pages and features manually
   - User can paste screenshots or describe what they see
   - Create specs from user descriptions instead of automated exploration

### Target Site Unreachable

When the target URL cannot be loaded:

1. **Report the Issue**

   ```
   Unable to reach the target site: <target.url>

   This could be due to:
   - The site is temporarily down
   - Network connectivity issues
   - The URL may be incorrect
   - The site may block automated access
   ```

2. **Verify with User**

   ```yaml
   questions:
     - question: "The target site couldn't be reached. What would you like to do?"
       header: "Unreachable"
       options:
         - label: "Retry now"
           description: "Try loading the site again"
         - label: "Check the URL"
           description: "Let me verify or update the target URL"
         - label: "I'll investigate"
           description: "Give me a moment to check the site manually"
       multiSelect: false
   ```

3. **Update Configuration if Needed**

   If user provides a corrected URL:
   - Update `target.url` in `.ralph/discovery-config.yaml`
   - Update the entry point spec's `source_urls`
   - Retry the discovery

### Rate Limiting or Bot Detection

When the target site blocks or throttles automated access:

1. **Detect the Signs**

   Bot detection may manifest as:
   - CAPTCHA challenges appearing
   - Requests being blocked or returning errors
   - Content not loading despite page load
   - Redirects to "are you human?" pages
   - Significantly slower responses

2. **Report to User**

   ```
   The target site appears to be blocking automated access.

   Signs detected:
   - <describe what happened: CAPTCHA, blocking, etc.>

   This is common for sites with bot protection (Cloudflare, reCAPTCHA, etc.).
   ```

3. **Suggest Alternatives**

   ```yaml
   questions:
     - question: "The site has bot protection. How would you like to proceed?"
       header: "Bot Detection"
       options:
         - label: "Manual navigation"
           description: "I'll browse and describe what I see"
         - label: "Slower exploration"
           description: "Try again with longer delays between pages"
         - label: "Switch to conversational"
           description: "Write specs from my descriptions instead"
       multiSelect: false
   ```

4. **Fall Back to Conversational Spec Writing**

   If automated exploration isn't viable:

   ```
   Let's write specs from your descriptions instead. As you browse the target site:

   1. Navigate to a page you want to clone
   2. Describe what you see (or paste a screenshot)
   3. I'll create/update the spec based on your description
   4. We'll repeat for each page or component

   This is slower but works around bot protection.
   ```

   In conversational mode:
   - Create specs from user's verbal descriptions
   - Ask clarifying questions about layout, behavior, interactions
   - User can paste screenshots which you can analyze
   - Build specs incrementally through dialogue
   - Mark as `status: draft` until user confirms completeness

5. **Document the Limitation**

   Add a note to `.ralph/discovery-config.yaml`:
   ```yaml
   notes: |
     Bot detection encountered. Using conversational spec writing.
   ```

### General Error Handling Principles

When any error occurs:

1. **Never lose progress** - Always save partial results before reporting errors
2. **Be specific** - Tell the user exactly what failed and why
3. **Offer choices** - Give the user options for how to proceed
4. **Enable recovery** - Make it easy to retry or work around issues
5. **Document blockers** - Record issues in specs or config for future reference
