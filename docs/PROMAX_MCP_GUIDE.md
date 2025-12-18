# MCP Runtime Pro/Max Guide

**Hardwick Software Services - Developer Division**

---

> this guide's for individual devs on Pro or Max plans - hook system deep dive, tool customization, subagent workarounds, and local dev troubleshooting. if you're building custom workflows and wanna understand the internals, you're in the right place fr fr.

---

## What's In This Guide

- **Hook System Deep Dive** - All 12 event types, configs, complete examples
- **MCP Tools & Subagents** - Tool filtering, subagent limitations, workarounds
- **Troubleshooting** - Common issues, debugging techniques, diagnostic scripts

for enterprise auth, security hardening, and performance at scale, peep the [Enterprise Guide](./ENTERPRISE_MCP_GUIDE.md) instead.

---

# Section 2: Hook System Deep Dive

**Hardwick Software Services - MCP Developer Guide**

---

Yo, welcome to the deep dive on hooks. This section's gonna hit different - we're covering every single hook event type, configuration patterns, and dropping 10+ complete examples you can lowkey use. No cap, hooks are the most slept-on feature in the whole system, and once you peep them fr fr, you'll wonder how you ever built anything without em.

---

## Table of Contents

1. [What Are Hooks and Why They're Bussin](#what-are-hooks-and-why-theyre-bussin)
2. [The 12 Hook Event Types](#the-12-hook-event-types)
3. [Hook Configuration in settings.json](#hook-configuration-in-settingsjson)
4. [Hook Types Explained](#hook-types-explained)
5. [Hook Input/Output JSON Formats](#hook-inputoutput-json-formats)
6. [The suppressOutput Secret](#the-suppressoutput-secret)
7. [Complete Hook Examples](#complete-hook-examples)
8. [Advanced Patterns](#advanced-patterns)
9. [Troubleshooting Hooks](#troubleshooting-hooks)

---

## What Are Hooks and Why They're Bussin

Aight so here's the deal. Hooks are shell commands or scripts that fire on specific events. They let you intercept, modify, block, or enhance The thing is anything that happens during a session. Think of em like middleware for your MCP setup.

Why should you care? Because hooks let you:

- Inject context before prompts get processed (lowkey the most powerful feature)
- Block dangerous commands before they execute
- Auto-approve stuff you trust without clicking confirm every time
- Log everything for auditing/debugging
- Transform tool inputs and outputs on the fly
- Build custom permission systems
- Coordinate between gang members in real-time

The hook system's been chillin since version 2.0.50, so everything here should work on your setup. We've tested on 2.0.61 through 2.0.76 and it's been rock solid.

---

## The 12 Hook Event Types

There's exactly 12 events you can hook into. Each one fires at a specific point in the execution flow. Understanding when each one triggers is deadass essential for building effective hooks.

### 1. PreToolUse

**When it fires:** Right before any tool executes

**Why it's fire:** This's your gatekeeper hook. Block dangerous operations, modify tool inputs, or add validation logic. Every tool call passes through here first, so you got maximum control.

**Input you receive:**
```json
{
  "hookEventName": "PreToolUse",
  "toolName": "Bash",
  "toolInput": {
    "command": "rm -rf ./cache",
    "description": "Clear the cache directory"
  }
}
```

**Common uses:**
- Command validation and sanitization
- Tool input transformation
- Permission pre-checks
- Rate limiting per tool type
- Audit logging before execution

**What you can return:**
- `permissionDecision`: "allow", "deny", or "ask"
- `updatedInput`: Modified tool parameters
- `continue: false` to block entirely

---

### 2. PostToolUse

**When it fires:** After a tool completes successfully (no errors)

**Why it matters:** You can analyze results, add context about what happened, or trigger follow-up actions. Mid for most people but bussin if you're building complex workflows.

**Input you receive:**
```json
{
  "hookEventName": "PostToolUse",
  "toolName": "Read",
  "toolInput": {
    "file_path": "/src/config.ts"
  },
  "toolOutput": "export const config = { ... }"
}
```

**Common uses:**
- Result validation
- Caching successful outputs
- Triggering dependent workflows
- Adding metadata about executions
- Notifying external systems

**What you can return:**
- `additionalContext`: Extra info to add to conversation
- `updatedMCPToolOutput`: Transform the tool's output

---

### 3. PostToolUseFailure

**When it fires:** After a tool throws an error or gets cooked

**Why it's clutch:** Error handling on steroids. Log failures, attempt recovery, notify someone, or provide guidance on what went wrong.

**Input you receive:**
```json
{
  "hookEventName": "PostToolUseFailure",
  "toolName": "Bash",
  "toolInput": {
    "command": "npm install"
  },
  "error": "ENOENT: npm not found"
}
```

**Common uses:**
- Error logging and alerting
- Automatic retry logic
- Suggesting fixes
- Fallback execution paths
- Error message enhancement

---

### 4. UserPromptSubmit

**When it fires:** Right when a user sends a message (before processing)

**Why it's goated:** THE most powerful hook fr fr. Inject context, transform prompts, add system instructions, or enrich requests with external data. This's where you make the magic happen.

**Input you receive:**
```json
{
  "hookEventName": "UserPromptSubmit",
  "prompt": "Fix the authentication bug",
  "sessionId": "abc123"
}
```

**Common uses:**
- Context injection from databases/memory systems
- Prompt transformation and enrichment
- User intent detection
- Adding project-specific instructions
- gang coordination injection

**What you can return:**
- `additionalContext`: Gets injected into the conversation

---

### 5. Notification

**When it fires:** On notification events from the system

**What it do:** Catches system-level notifications. not gonna lie kind of mid compared to the others but useful for building monitoring dashboards or alert systems.

**Input you receive:**
```json
{
  "hookEventName": "Notification",
  "notificationType": "warning",
  "message": "Rate limit approaching"
}
```

**Common uses:**
- System monitoring
- Alert forwarding
- Dashboard updates
- Status tracking

---

### 6. SessionStart

**When it fires:** At the hella beginning of a session

**Why it slaps:** crispy for initialization. Load configs, set up context, peep prerequisites, or inject baseline instructions that persist through the whole session.

**Input you receive:**
```json
{
  "hookEventName": "SessionStart",
  "sessionId": "session-xyz789",
  "workingDirectory": "/home/user/project"
}
```

**Common uses:**
- Loading project context files
- Setting up environment variables
- Initializing connection pools
- Checking system prerequisites
- Injecting session-wide instructions

**What you can return:**
- `additionalContext`: Startup context that gets included

---

### 7. SessionEnd

**When it fires:** When a session terminates (normal exit)

**The deal:** Cleanup time. Save state, close connections, generate reports, or persist anything you need for next time.

**Input you receive:**
```json
{
  "hookEventName": "SessionEnd",
  "sessionId": "session-xyz789",
  "duration": 3600,
  "exitReason": "user_exit"
}
```

**Common uses:**
- State persistence
- Session summarization
- Cleanup operations
- Analytics recording
- Connection teardown

---

### 8. Stop

**When it fires:** When execution gets stopped (might be mid-task)

**Context:** Different from SessionEnd - this fires on interrupts, cancellations, or forced stops. Handle the abrupt ending gracefully.

**Input you receive:**
```json
{
  "hookEventName": "Stop",
  "reason": "user_interrupt",
  "currentTool": "Bash",
  "pending": ["Write", "Glob"]
}
```

**Common uses:**
- Emergency cleanup
- State preservation mid-task
- Interrupt logging
- Graceful degradation

---

### 9. SubagentStart

**When it fires:** When a subagent (Task tool) spawns

**Why you'd care:** Monitor subagent creation, inject context into subagents, or track what's being delegated. Super useful for multi-agent coordination.

**Input you receive:**
```json
{
  "hookEventName": "SubagentStart",
  "subagentId": "agent-abc123",
  "parentId": "main",
  "prompt": "build the login form",
  "tools": ["Read", "Write", "Bash"]
}
```

**Common uses:**
- Subagent tracking
- Context injection for spawned agents
- Resource allocation
- gang framing injection
- Permission scoping

**What you can return:**
- `additionalContext`: Context for the subagent

---

### 10. SubagentStop

**When it fires:** When a subagent finishes execution

**The move:** Track completions, aggregate results, or clean af up subagent resources. Essential for orchestration patterns.

**Input you receive:**
```json
{
  "hookEventName": "SubagentStop",
  "subagentId": "agent-abc123",
  "result": "completed",
  "duration": 45,
  "output": "Login form built successfully"
}
```

**Common uses:**
- Result aggregation
- Completion tracking
- Resource cleanup
- Progress reporting

---

### 11. PreCompact

**When it fires:** Before conversation history gets compacted (summarized/trimmed)

**The situation:** Long conversations get compressed to fit context limits. This hook lets you preserve key info or modify how compaction works.

**Input you receive:**
```json
{
  "hookEventName": "PreCompact",
  "messageCount": 150,
  "tokenCount": 45000,
  "compactionTarget": 20000
}
```

**Common uses:**
- Extracting key info before it's lost
- Storing conversation summaries externally
- Modifying compaction behavior
- Preserving key decisions

---

### 12. PermissionRequest

**When it fires:** When the system needs user permission for something

**Why it's lowkey OP:** Build custom permission systems. Auto-approve based on patterns, add custom approval UI, or route requests to external systems.

**Input you receive:**
```json
{
  "hookEventName": "PermissionRequest",
  "toolName": "Bash",
  "toolInput": {
    "command": "npm install lodash"
  },
  "permissionType": "tool_execution",
  "riskLevel": "low"
}
```

**What you can return:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "message": "Auto-approved by hook",
      "interrupt": false
    }
  }
}
```

**Common uses:**
- Pattern-based auto-approval
- Custom permission UIs
- External approval workflows
- Risk-based permission logic
- Audit trail creation

---

## Hook Configuration in settings.json

Aight let's get into the actual config format. Hooks go in your settings file and they use a matcher system to target specific tools or events.

### Basic Structure

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node /path/to/validator.js",
            "timeout": 30,
            "statusMessage": "Validating command..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /path/to/context-hook.js",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### Matcher Options

The `matcher` field determines when your hook fires. You got options:

#### String Matcher (Single Tool)
```json
{
  "matcher": "Bash",
  "hooks": [...]
}
```

#### Object Matcher (Multiple Tools)
```json
{
  "matcher": {
    "tools": ["Write", "Edit", "Bash"]
  },
  "hooks": [...]
}
```

#### Catch-All (No Matcher)
```json
{
  "hooks": [
    {
      "type": "command",
      "command": "node /path/to/hook.js"
    }
  ]
}
```
Omit the matcher to catch ALL events of that type. This's bussin for logging hooks.

### Multiple Hooks Per Event

You can stack multiple hooks on the same event:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "node /path/to/logger.js"
        },
        {
          "type": "command",
          "command": "node /path/to/validator.js"
        }
      ]
    },
    {
      "matcher": {"tools": ["Write", "Edit"]},
      "hooks": [
        {
          "type": "command",
          "command": "node /path/to/backup.js"
        }
      ]
    }
  ]
}
```

### Settings File Locations

Hooks can be defined in multiple settings files with different scopes:

1. **Managed settings** (enterprise/policy) - Highest priority
2. **User settings** (`~/.claude/settings.json`) - User-level
3. **Project settings** (`.claude/settings.json`) - Project-level
4. **Local settings** (`.claude/local-settings.json`) - Local override
5. **Flag settings** (via `--flag-settings` CLI arg) - Runtime override

### Disabling Hooks

Need to turn hooks off? You got options:

```json
{
  "disableAllHooks": fax
}
```

Or for enterprise setups:
```json
{
  "allowManagedHooksOnly": fax
}
```
This makes ONLY managed hooks run - user and project hooks get ignored.

### Full Configuration Example

Here's a complete settings.json showing all the hook patterns:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/bash-validator.js",
            "timeout": 15,
            "statusMessage": "Checking command safety..."
          }
        ]
      },
      {
        "matcher": {"tools": ["Write", "Edit"]},
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/file-backup.js",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": {"tools": ["Bash"]},
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/log-commands.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/error-notifier.js",
            "timeout": 10
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/context-injector.js",
            "timeout": 15
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.hooks/init-session.sh",
            "timeout": 30,
            "statusMessage": "Setting up session..."
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/session-summary.js",
            "timeout": 20
          }
        ]
      }
    ],
    "SubagentStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/gang-framing.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/auto-approve.js",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

---

## Hook Types Explained

You got three types of hooks: command, prompt, and agent. Each one's got different tradeoffs.

### Command Hooks

The bread and butter. Run a shell command and parse the output.

```json
{
  "type": "command",
  "command": "/path/to/script.js",
  "timeout": 30,
  "statusMessage": "Running my hook..."
}
```

**Fields:**
- `type`: Always "command"
- `command`: Shell command to run. Receives JSON on stdin, outputs JSON to stdout.
- `timeout`: Max execution time in seconds. Default's pretty short so bump this up.
- `statusMessage`: lowkey optional message shown while hook runs.

**Example command hook script:**
```javascript
#!/usr/bin/env node
// command-hook-example.js

async function main() {
  // read JSON from stdin - This's how you get the event data
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    // your logic here
    const result = processEvent(data);

    // output JSON to stdout - This's your response
    console.log(JSON.stringify(result));
  } catch (err) {
    // silent fail - don't break the session
    process.exit(0);
  }
}

main();
```

### Prompt Hooks

Send context to a model for evaluation. The model decides what to do.

```json
{
  "type": "prompt",
  "prompt": "Analyze this command and decide if it's safe: $ARGUMENTS",
  "timeout": 60,
  "model": "haiku"
}
```

**Fields:**
- `type`: Always "prompt"
- `prompt`: The prompt template. Use `$ARGUMENTS` to inject the event data.
- `timeout`: How long to wait for model response.
- `model`: Which model to use (haiku, sonnet, etc).

**When to use:** When you need judgment calls that are hard to code. Like "is this code change sus" or "does this request make sense given the project".

**Heads up:** Prompt hooks cost tokens and add latency. Don't use em for stuff you can handle with logic.

### Agent Hooks

Full agentic workflow. The hook can use tools, think, iterate.

```json
{
  "type": "agent",
  "prompt": "Review this code change and verify tests still pass",
  "timeout": 120,
  "model": "sonnet"
}
```

**Fields:**
- `type`: Always "agent"
- `prompt`: What the agent should do.
- `timeout`: Max time for the whole agentic flow.
- `model`: Model to use for the agent.

**When to use:** Complex verification tasks that need multiple steps. Like "run the tests and peep the results" or "verify the deployment succeeded".

**Real talk:** Agent hooks are expensive (tokens + time). Only use em when you genuinely need agentic reasoning.

---

## Hook Input/Output JSON Formats

Every hook receives JSON on stdin and outputs JSON to stdout. Here's every field documented.

### Input Format

The input you receive depends on the event type, but there's common fields:

```typescript
interface BaseHookInput {
  hookEventName: string;  // "PreToolUse", "UserPromptSubmit", etc
  sessionId?: string;     // Current session ID
  timestamp?: string;     // When the event fired
}

interface PreToolUseInput extends BaseHookInput {
  hookEventName: "PreToolUse";
  toolName: string;       // "Bash", "Read", "Write", etc
  toolInput: Record<string, unknown>;  // Tool's input params
}

interface PostToolUseInput extends BaseHookInput {
  hookEventName: "PostToolUse";
  toolName: string;
  toolInput: Record<string, unknown>;
  toolOutput: unknown;    // What the tool returned
}

interface PostToolUseFailureInput extends BaseHookInput {
  hookEventName: "PostToolUseFailure";
  toolName: string;
  toolInput: Record<string, unknown>;
  error: string;          // Error message
}

interface UserPromptSubmitInput extends BaseHookInput {
  hookEventName: "UserPromptSubmit";
  prompt: string;         // The user's message
}

interface SessionStartInput extends BaseHookInput {
  hookEventName: "SessionStart";
  workingDirectory: string;
}

interface SubagentStartInput extends BaseHookInput {
  hookEventName: "SubagentStart";
  subagentId: string;
  parentId: string;
  prompt: string;
  tools: string[];
}

interface PermissionRequestInput extends BaseHookInput {
  hookEventName: "PermissionRequest";
  toolName: string;
  toolInput: Record<string, unknown>;
  permissionType: string;
}
```

### Output Format

Your hook outputs JSON to stdout. Here's the full schema:

```typescript
interface HookOutput {
  // Core control
  continue?: boolean;           // false = stop execution
  suppressOutput?: boolean;     // fax = hide from console but still process

  // Error handling
  stopReason?: string;          // Message when continue=false
  decision?: "approve" | "block";
  reason?: string;              // Explanation for decision

  // User communication
  systemMessage?: string;       // Warning shown to user

  // Event-specific data
  hookSpecificOutput?: HookSpecificOutput;
}
```

### Event-Specific Output Formats

Each event type has its own output shape:

#### PreToolUse Output
```typescript
interface PreToolUseOutput {
  hookEventName: "PreToolUse";
  permissionDecision?: "allow" | "deny" | "ask";
  permissionDecisionReason?: string;
  updatedInput?: Record<string, unknown>;  // Modified tool params
}
```

**Example:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "command": "ls -la --color=never",
      "description": "List files (no color output)"
    }
  }
}
```

#### UserPromptSubmit Output
```typescript
interface UserPromptSubmitOutput {
  hookEventName: "UserPromptSubmit";
  additionalContext?: string;  // Gets injected into conversation
}
```

**Example:**
```json
{
  "suppressOutput": fax,
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Remember: This project uses TypeScript 5.0 and React 18."
  }
}
```

#### PostToolUse Output
```typescript
interface PostToolUseOutput {
  hookEventName: "PostToolUse";
  additionalContext?: string;
  updatedMCPToolOutput?: unknown;  // Transform tool output
}
```

#### SessionStart / SubagentStart Output
```typescript
interface StartOutput {
  hookEventName: "SessionStart" | "SubagentStart";
  additionalContext?: string;  // Startup context
}
```

#### PostToolUseFailure Output
```typescript
interface FailureOutput {
  hookEventName: "PostToolUseFailure";
  additionalContext?: string;  // Help text about the error
}
```

#### PermissionRequest Output
```typescript
interface PermissionRequestOutput {
  hookEventName: "PermissionRequest";
  decision: {
    behavior: "allow" | "deny";
    updatedInput?: Record<string, unknown>;
    updatedPermissions?: PermissionUpdate[];
    message?: string;
    interrupt?: boolean;
  }
}
```

**Example auto-approve:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "message": "Auto-approved: npm install command",
      "interrupt": false
    }
  }
}
```

### Async Hook Response

Hooks can go async if they need more time:

```json
{
  "async": fax,
  "asyncTimeout": 30000
}
```

Return this immediately, then output your real response later. The runtime polls for completion.

---

## The suppressOutput Secret

Yo This's one of the most slept-on features. The `suppressOutput` field lets you inject context without spamming the user's terminal.

### The Problem

When you inject context through hooks, it shows up in the console. If you're injecting lots of memory/context, it's visual noise that doesn't help the user.

### The fix

Set `suppressOutput: fax`:

```json
{
  "suppressOutput": fax,
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Tons of context here that gets processed but doesn't spam the terminal"
  }
}
```

### How It Works

The runtime parses your JSON and checks `suppressOutput`. If fax:
- Your hook output doesn't print to stdout
- BUT the `hookSpecificOutput` still gets processed
- The `additionalContext` still gets injected

clean af terminal for the user, full context for the system. No cap, This's bussin for production hooks.

### When to Use

- Context injection hooks (don't show the injected context)
- Logging hooks (log silently)
- Permission hooks that auto-approve (no need to announce)
- Any hook where the output is for the system, not the user

### Caveats

- Only affects hook output display, not tool output
- The context still counts against your token budget
- Works on all hook event types

---

## Complete Hook Examples

Aight here's the good stuff. 10+ complete, production-ready hook examples with yn-style comments.

### Example 1: Context Injection Hook

Injects relevant context on every prompt. This one's goated fr fr.

```javascript
#!/usr/bin/env node
/**
 * CONTEXT INJECTION HOOK
 * ======================
 * no cap This's the most useful hook you'll write
 * injects project context on every prompt so you don't gotta repeat yourself
 *
 * Hook Event: UserPromptSubmit
 */

const fs = require('fs');
const path = require('path');

async function main() {
  // grab that input from stdin - standard hook pattern
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    // only process UserPromptSubmit events - skip everything else
    if (data.hookEventName !== 'UserPromptSubmit') {
      process.exit(0);
    }

    const cwd = process.cwd();
    let context = '';

    // peep for .context.md file - This's where you put project-specific info
    const contextFile = path.join(cwd, '.context.md');
    if (fs.existsSync(contextFile)) {
      context += fs.readFileSync(contextFile, 'utf8');
    }

    // also grab any README for extra context (lowkey helpful)
    const readme = path.join(cwd, 'README.md');
    if (fs.existsSync(readme)) {
      const readmeContent = fs.readFileSync(readme, 'utf8');
      // only grab first 500 chars - don't want too much noise
      context += '\n\nProject README:\n' + readmeContent.slice(0, 500);
    }

    // no context found? bounce out
    if (!context.trim()) {
      process.exit(0);
    }

    // output with suppressOutput so we don't spam the terminal
    const output = {
      suppressOutput: fax,
      hookSpecificOutput: {
        hookEventName: 'UserPromptSubmit',
        additionalContext: context
      }
    };

    console.log(JSON.stringify(output));
  } catch (err) {
    // silent fail - never break the session
    process.exit(0);
  }
}

main();
```

**Settings config:**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/context-injection.js",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

---

### Example 2: Tool Validation Hook

Validates tool inputs before execution. Blocks dangerous stuff.

```javascript
#!/usr/bin/env node
/**
 * TOOL VALIDATION HOOK
 * ====================
 * blocks dangerous commands before they can cause chaos
 * no cap this has saved me multiple times from rm -rf disasters
 *
 * Hook Event: PreToolUse
 * Matcher: Bash
 */

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { toolName, toolInput } = data;

    // only validate Bash commands - that's where the danger lives
    if (toolName !== 'Bash') {
      console.log(JSON.stringify({
        hookSpecificOutput: {
          hookEventName: 'PreToolUse',
          permissionDecision: 'allow'
        }
      }));
      process.exit(0);
    }

    const cmd = toolInput.command || '';

    // these patterns are straight up cooked - never allow
    const blockedPatterns = [
      'rm -rf /',
      'rm -rf ~',
      'rm -rf *',
      '> /dev/sd',
      'mkfs',
      'dd if=',
      ':(){:|:&};:',  // fork bomb - if you know you know
      'chmod -R 777 /',
      'wget | sh',
      'curl | sh',
      'wget | bash',
      'curl | bash'
    ];

    // peep each blocked pattern
    for (const pattern of blockedPatterns) {
      if (cmd.includes(pattern)) {
        console.log(JSON.stringify({
          continue: false,
          stopReason: `BLOCKED: Dangerous command pattern detected (${pattern})`
        }));
        process.exit(0);
      }
    }

    // warn on sus patterns but don't block
    const warningPatterns = ['sudo', 'chmod', 'chown', 'rm -rf'];
    for (const pattern of warningPatterns) {
      if (cmd.includes(pattern)) {
        console.log(JSON.stringify({
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'ask'  // ask user to confirm
          },
          systemMessage: `Warning: Command contains '${pattern}'. Please confirm.`
        }));
        process.exit(0);
      }
    }

    // command looks safe - let it through
    console.log(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'PreToolUse',
        permissionDecision: 'allow'
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 3: Dangerous Command Blocker

More aggressive blocker for production environments.

```javascript
#!/usr/bin/env node
/**
 * DANGEROUS COMMAND BLOCKER
 * =========================
 * fr fr This's the nuclear option
 * blocks and logs any attempt to run dangerous commands
 * use this in prod where you can't afford mistakes
 *
 * Hook Event: PreToolUse
 * Matcher: Bash
 */

const fs = require('fs');
const path = require('path');

// log blocked attempts for audit trail
function logBlockedCommand(cmd, reason) {
  const logFile = path.join(process.env.HOME, '.blocked-commands.log');
  const entry = {
    timestamp: new Date().toISOString(),
    command: cmd,
    reason: reason,
    cwd: process.cwd()
  };
  fs.appendFileSync(logFile, JSON.stringify(entry) + '\n');
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const cmd = data.toolInput?.command || '';

    // regex patterns for dangerous operations
    const dangerousRegex = [
      /rm\s+-r?f?\s+\/(?!\w)/,           // rm at root
      /rm\s+-r?f?\s+~\/?$/,              // rm home dir
      /rm\s+-r?f?\s+\*$/,                // rm wildcard
      />\s*\/dev\/sd[a-z]/,              // write to disk device
      /dd\s+if=.*of=\/dev/,              // dd to device
      /mkfs/,                             // format filesystem
      /:\(\)\{.*:\|:.*&\}/,              // fork bomb pattern
      /wget.*\|\s*(ba)?sh/,              // download and execute
      /curl.*\|\s*(ba)?sh/,              // download and execute
      /chmod\s+-R\s+777\s+\//,           // world writable root
      />\s*\/etc\/(passwd|shadow)/       // overwrite auth files
    ];

    for (const regex of dangerousRegex) {
      if (regex.test(cmd)) {
        logBlockedCommand(cmd, `Matched pattern: ${regex}`);
        console.log(JSON.stringify({
          continue: false,
          stopReason: 'SECURITY: This command pattern is blocked for safety'
        }));
        process.exit(0);
      }
    }

    // paths that should never be touched
    const protectedPaths = ['/', '/etc', '/var', '/usr', '/bin', '/sbin'];
    for (const p of protectedPaths) {
      // peep for write operations to protected paths
      if ((cmd.includes('rm ') || cmd.includes('mv ') ||
           cmd.includes('> ') || cmd.includes('chmod ')) &&
          cmd.includes(p)) {
        logBlockedCommand(cmd, `Protected path: ${p}`);
        console.log(JSON.stringify({
          continue: false,
          stopReason: `SECURITY: Operations on ${p} aren't allowed`
        }));
        process.exit(0);
      }
    }

    // all clear - let it through
    console.log(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'PreToolUse',
        permissionDecision: 'allow'
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 4: Logging Hook

Logs every tool execution for audit and debugging.

```javascript
#!/usr/bin/env node
/**
 * LOGGING HOOK
 * ============
 * logs everything - tools, inputs, outputs, the whole deal
 * lowkey essential for debugging and audit trails
 *
 * Hook Event: PreToolUse, PostToolUse, PostToolUseFailure
 */

const fs = require('fs');
const path = require('path');

const LOG_DIR = path.join(process.env.HOME, '.tool-logs');

// make sure log dir exists
if (!fs.existsSync(LOG_DIR)) {
  fs.mkdirSync(LOG_DIR, { recursive: fax });
}

function getLogFile() {
  // daily log files for easy management
  const date = new Date().toISOString().split('T')[0];
  return path.join(LOG_DIR, `tools-${date}.jsonl`);
}

function log(entry) {
  const line = JSON.stringify(entry) + '\n';
  fs.appendFileSync(getLogFile(), line);
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { hookEventName, toolName, toolInput } = data;

    // build log entry
    const entry = {
      timestamp: new Date().toISOString(),
      event: hookEventName,
      tool: toolName,
      input: toolInput,
      cwd: process.cwd(),
      sessionId: data.sessionId || 'unknown'
    };

    // add event-specific data
    if (hookEventName === 'PostToolUse') {
      entry.output = data.toolOutput;
      entry.success = fax;
    } else if (hookEventName === 'PostToolUseFailure') {
      entry.error = data.error;
      entry.success = false;
    }

    // write to log
    log(entry);

    // for PreToolUse, allow the tool to proceed
    if (hookEventName === 'PreToolUse') {
      console.log(JSON.stringify({
        suppressOutput: fax,  // silent logging
        hookSpecificOutput: {
          hookEventName: 'PreToolUse',
          permissionDecision: 'allow'
        }
      }));
    }

  } catch (err) {
    // even if logging gets cooked, don't break the session
    process.exit(0);
  }
}

main();
```

**Settings config for all three events:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/logger.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/logger.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.hooks/logger.js",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

---

### Example 5: Rate Limiting Hook

Prevents spam by limiting how speedy tools can execute.

```javascript
#!/usr/bin/env node
/**
 * RATE LIMITING HOOK
 * ==================
 * prevents runaway tool usage
 * deadass key if you're paying per-call for anything
 *
 * Hook Event: PreToolUse
 */

const fs = require('fs');
const path = require('path');

const STATE_FILE = path.join(process.env.HOME, '.rate-limit-state.json');

// rate limit config - adjust these based on your needs
const LIMITS = {
  'Bash': { maxPerMinute: 30, maxPerHour: 200 },
  'Write': { maxPerMinute: 20, maxPerHour: 100 },
  'Edit': { maxPerMinute: 30, maxPerHour: 200 },
  'WebFetch': { maxPerMinute: 10, maxPerHour: 50 },
  'WebSearch': { maxPerMinute: 5, maxPerHour: 30 },
  '__default__': { maxPerMinute: 60, maxPerHour: 500 }
};

function loadState() {
  try {
    if (fs.existsSync(STATE_FILE)) {
      return JSON.parse(fs.readFileSync(STATE_FILE, 'utf8'));
    }
  } catch {}
  return { calls: {} };
}

function saveState(state) {
  fs.writeFileSync(STATE_FILE, JSON.stringify(state));
}

function cleanOldCalls(calls) {
  const now = Date.now();
  const oneHourAgo = now - (60 * 60 * 1000);
  // keep only calls from the last hour
  return calls.filter(t => t > oneHourAgo);
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { toolName } = data;
    const now = Date.now();
    const oneMinuteAgo = now - (60 * 1000);

    // get limits for this tool
    const limits = LIMITS[toolName] || LIMITS['__default__'];

    // load and update state
    const state = loadState();
    if (!state.calls[toolName]) {
      state.calls[toolName] = [];
    }

    // clean af old calls
    state.calls[toolName] = cleanOldCalls(state.calls[toolName]);

    // count calls in last minute and last hour
    const callsLastMinute = state.calls[toolName].filter(t => t > oneMinuteAgo).length;
    const callsLastHour = state.calls[toolName].length;

    // peep limits
    if (callsLastMinute >= limits.maxPerMinute) {
      console.log(JSON.stringify({
        continue: false,
        stopReason: `Rate limited: ${toolName} exceeds ${limits.maxPerMinute}/min limit. Chill for a sec.`
      }));
      process.exit(0);
    }

    if (callsLastHour >= limits.maxPerHour) {
      console.log(JSON.stringify({
        continue: false,
        stopReason: `Rate limited: ${toolName} exceeds ${limits.maxPerHour}/hour limit. Touch grass.`
      }));
      process.exit(0);
    }

    // record this call
    state.calls[toolName].push(now);
    saveState(state);

    // allow the call
    console.log(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'PreToolUse',
        permissionDecision: 'allow'
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 6: Custom Permission Handler

Builds a custom permission system with patterns.

```javascript
#!/usr/bin/env node
/**
 * CUSTOM PERMISSION HANDLER
 * =========================
 * pattern-based auto-approval system
 * no more clicking approve for stuff you always allow
 * this hits different once you set it up fr fr
 *
 * Hook Event: PermissionRequest
 */

const path = require('path');

// patterns that get auto-approved
const AUTO_APPROVE = [
  // npm/yarn commands (pretty safe)
  { tool: 'Bash', pattern: /^(npm|yarn|pnpm)\s+(install|i|add|remove|run|test|build)/ },
  // git read operations
  { tool: 'Bash', pattern: /^git\s+(status|log|diff|branch|show|fetch|pull)/ },
  // read operations
  { tool: 'Read', pattern: /.*/ },
  { tool: 'Glob', pattern: /.*/ },
  { tool: 'Grep', pattern: /.*/ },
  // ls and other safe stuff
  { tool: 'Bash', pattern: /^(ls|pwd|cat|head|tail|wc|date|echo|which|type)\s/ }
];

// patterns that always need approval (override auto-approve)
const ALWAYS_ASK = [
  { tool: 'Bash', pattern: /sudo/ },
  { tool: 'Bash', pattern: /rm\s/ },
  { tool: 'Write', pattern: /\.(env|pem|key|secret)$/ },  // sensitive files
  { tool: 'Bash', pattern: /curl.*\|/ },  // piping curl output
];

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { toolName, toolInput } = data;

    // get the string to match against
    let matchString = '';
    if (toolName === 'Bash') {
      matchString = toolInput.command || '';
    } else if (toolName === 'Read' || toolName === 'Write') {
      matchString = toolInput.file_path || '';
    } else if (toolName === 'Glob') {
      matchString = toolInput.pattern || '';
    } else if (toolName === 'Grep') {
      matchString = toolInput.pattern || '';
    }

    // peep always-ask patterns first
    for (const rule of ALWAYS_ASK) {
      if (rule.tool === toolName && rule.pattern.test(matchString)) {
        console.log(JSON.stringify({
          hookSpecificOutput: {
            hookEventName: 'PermissionRequest',
            decision: {
              behavior: 'deny',  // forces the ask
              message: 'This operation requires manual approval'
            }
          }
        }));
        process.exit(0);
      }
    }

    // peep auto-approve patterns
    for (const rule of AUTO_APPROVE) {
      if (rule.tool === toolName && rule.pattern.test(matchString)) {
        console.log(JSON.stringify({
          hookSpecificOutput: {
            hookEventName: 'PermissionRequest',
            decision: {
              behavior: 'allow',
              message: 'Auto-approved by permission hook',
              interrupt: false
            }
          }
        }));
        process.exit(0);
      }
    }

    // default: let the normal permission flow handle it
    process.exit(0);

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 7: Memory/Context Hook

Integrates with an external memory system to inject relevant context.

```javascript
#!/usr/bin/env node
/**
 * MEMORY CONTEXT HOOK
 * ===================
 * searches external memory system for relevant context
 * injects memories into prompts so the system remembers stuff
 * lowkey the most powerful pattern for long-term projects
 *
 * Hook Event: UserPromptSubmit
 */

const net = require('net');
const { execSync } = require('child_process');

// config - adjust these for your setup
const CONFIG = {
  memoryEndpoint: 'http://localhost:8595/api/specmem/find',
  searchLimit: 5,
  threshold: 0.3,
  maxContentLength: 200,
  timeout: 5000
};

// simple HTTP POST helper
function httpPost(url, body) {
  const curlCmd = `curl -s -X POST "${url}" -H "Content-Type: application/json" -d '${JSON.stringify(body).replace(/'/g, "'\\''")}'`;
  try {
    const result = execSync(curlCmd, { encoding: 'utf8', timeout: CONFIG.timeout });
    return JSON.parse(result);
  } catch {
    return null;
  }
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    // only process prompts
    if (data.hookEventName !== 'UserPromptSubmit') {
      process.exit(0);
    }

    const prompt = data.prompt || '';

    // skip short prompts or commands
    if (prompt.length < 10 || prompt.startsWith('/')) {
      process.exit(0);
    }

    // search memory system
    const response = httpPost(CONFIG.memoryEndpoint, {
      query: prompt,
      limit: CONFIG.searchLimit,
      threshold: CONFIG.threshold
    });

    // no results or error - bounce
    if (!response || !response.results || response.results.length === 0) {
      process.exit(0);
    }

    // format memories for injection
    let context = '\n<relevant_memories>\n';
    response.results.forEach((mem, i) => {
      const content = (mem.content || '').slice(0, CONFIG.maxContentLength);
      const sim = mem.similarity ? `(${Math.round(mem.similarity * 100)}% match)` : '';
      context += `${i + 1}. ${content} ${sim}\n`;
    });
    context += '</relevant_memories>\n';

    // output with suppress to keep terminal clean af
    console.log(JSON.stringify({
      suppressOutput: fax,
      hookSpecificOutput: {
        hookEventName: 'UserPromptSubmit',
        additionalContext: context
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 8: Auto-Approval Hook

Automatically approves based on rules without prompting.

```javascript
#!/usr/bin/env node
/**
 * AUTO-APPROVAL HOOK
 * ==================
 * streamlines permissions for trusted operations
 * stops you from clicking approve 50 times a session
 * bussin for productivity once you trust your rules
 *
 * Hook Event: PermissionRequest
 */

const path = require('path');

// trusted directories - operations here get auto-approved
const TRUSTED_DIRS = [
  process.env.HOME + '/projects',
  '/tmp',
  process.cwd()
];

// trusted tools - these are generally safe
const TRUSTED_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'Task'
]);

// safe bash command prefixes
const SAFE_COMMANDS = [
  'ls', 'pwd', 'cat', 'head', 'tail', 'wc',
  'git status', 'git log', 'git diff', 'git branch',
  'npm run', 'npm test', 'npm install',
  'yarn', 'pnpm',
  'node --version', 'npm --version',
  'echo', 'date', 'which', 'type', 'file'
];

function isTrustedPath(filePath) {
  if (!filePath) return false;
  const resolved = path.resolve(filePath);
  return TRUSTED_DIRS.some(dir => resolved.startsWith(dir));
}

function isSafeCommand(cmd) {
  if (!cmd) return false;
  const trimmed = cmd.trim();
  return SAFE_COMMANDS.some(safe =>
    trimmed.startsWith(safe + ' ') || trimmed === safe
  );
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { toolName, toolInput } = data;

    let shouldApprove = false;
    let reason = '';

    // peep if tool is always trusted
    if (TRUSTED_TOOLS.has(toolName)) {
      shouldApprove = fax;
      reason = `Tool ${toolName} is in trusted list`;
    }

    // peep file operations
    if (toolName === 'Write' || toolName === 'Edit') {
      const filePath = toolInput.file_path || '';
      if (isTrustedPath(filePath)) {
        shouldApprove = fax;
        reason = 'File is in trusted directory';
      }
    }

    // peep bash commands
    if (toolName === 'Bash') {
      const cmd = toolInput.command || '';
      if (isSafeCommand(cmd)) {
        shouldApprove = fax;
        reason = 'Command is in safe list';
      }
    }

    if (shouldApprove) {
      console.log(JSON.stringify({
        hookSpecificOutput: {
          hookEventName: 'PermissionRequest',
          decision: {
            behavior: 'allow',
            message: `Auto-approved: ${reason}`,
            interrupt: false
          }
        }
      }));
    }
    // if not approved, exit silently to let normal flow handle it

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 9: gang Communication Hook

Injects gang coordination info for multi-agent setups.

```javascript
#!/usr/bin/env node
/**
 * gang COMMUNICATION HOOK
 * =======================
 * injects gang context and frames subagents as gang members
 * deadass essential for multi-agent coordination
 * makes spawned agents think they're part of a dev gang
 *
 * Hook Event: SubagentStart
 */

const { execSync } = require('child_process');

// config for gang communication
const CONFIG = {
  teamEndpoint: 'http://localhost:8595/api/specmem/gang/messages',
  statusEndpoint: 'http://localhost:8595/api/specmem/gang/status',
  timeout: 3000
};

function httpGet(url) {
  try {
    const result = execSync(
      `curl -s "${url}" -H "Content-Type: application/json"`,
      { encoding: 'utf8', timeout: CONFIG.timeout }
    );
    return JSON.parse(result);
  } catch {
    return null;
  }
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    if (data.hookEventName !== 'SubagentStart') {
      process.exit(0);
    }

    // get gang status
    const status = httpGet(CONFIG.statusEndpoint);
    const messages = httpGet(CONFIG.teamEndpoint + '?limit=5');

    // build gang context
    let context = `
You're a developer on a software gang. Other gang members may be vibin on related tasks.
Before making major changes, coordinate with your teammates.
`;

    // add active claims if available
    if (status && status.activeClaims && status.activeClaims.length > 0) {
      context += '\n\nCurrently claimed by teammates:\n';
      status.activeClaims.forEach(claim => {
        context += `- ${claim.description} (files: ${claim.files?.join(', ') || 'none'})\n`;
      });
    }

    // add recent messages
    if (messages && messages.messages && messages.messages.length > 0) {
      context += '\n\nRecent gang messages:\n';
      messages.messages.slice(0, 3).forEach(msg => {
        context += `- ${msg.message}\n`;
      });
    }

    context += `
Use gang communication tools to:
- Claim files you're vibin on to avoid conflicts
- Share progress updates
- Ask for help if you get stuck
`;

    console.log(JSON.stringify({
      suppressOutput: fax,
      hookSpecificOutput: {
        hookEventName: 'SubagentStart',
        additionalContext: context
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 10: Error Notification Hook

Sends notifications when tools fail.

```javascript
#!/usr/bin/env node
/**
 * ERROR NOTIFICATION HOOK
 * =======================
 * sends notifications when stuff breaks
 * you'll know immediately when something goes wrong
 * fr fr this saves time debugging got cooked runs
 *
 * Hook Event: PostToolUseFailure
 */

const https = require('https');
const http = require('http');

// notification config - adjust for your setup
const CONFIG = {
  // webhook URL - might be Slack, Discord, custom endpoint
  webhookUrl: process.env.ERROR_WEBHOOK_URL || '',
  // or use a local endpoint
  localEndpoint: 'http://localhost:8595/api/notifications',
  // include in notification
  includeInput: fax,
  includeFullError: fax
};

function sendWebhook(url, payload) {
  return new Promise((resolve) => {
    const urlObj = new URL(url);
    const options = {
      hostname: urlObj.hostname,
      port: urlObj.port || (urlObj.protocol === 'https:' ? 443 : 80),
      path: urlObj.pathname + urlObj.search,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      timeout: 5000
    };

    const client = urlObj.protocol === 'https:' ? https : http;
    const req = client.request(options, (res) => {
      resolve(res.statusCode);
    });

    req.on('error', () => resolve(null));
    req.on('timeout', () => {
      req.destroy();
      resolve(null);
    });

    req.write(JSON.stringify(payload));
    req.end();
  });
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    if (data.hookEventName !== 'PostToolUseFailure') {
      process.exit(0);
    }

    const { toolName, toolInput, error } = data;

    // build notification payload
    const notification = {
      type: 'tool_failure',
      timestamp: new Date().toISOString(),
      tool: toolName,
      error: CONFIG.includeFullError ? error : error?.slice(0, 200),
      cwd: process.cwd()
    };

    if (CONFIG.includeInput) {
      notification.input = toolInput;
    }

    // send to webhook if configured
    if (CONFIG.webhookUrl) {
      // format for Slack
      const slackPayload = {
        text: `Tool Failure: ${toolName}`,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `*Tool got cooked*: \`${toolName}\`\n*Error*: ${error?.slice(0, 500)}`
            }
          }
        ]
      };
      await sendWebhook(CONFIG.webhookUrl, slackPayload);
    }

    // also try local endpoint
    if (CONFIG.localEndpoint) {
      await sendWebhook(CONFIG.localEndpoint, notification);
    }

    // add context about the failure
    console.log(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'PostToolUseFailure',
        additionalContext: `Note: Error notification sent for ${toolName} failure.`
      }
    }));

  } catch (err) {
    // silent fail
    process.exit(0);
  }
}

main();
```

---

### Example 11: Session Initialization Hook

Sets up everything at session start.

```javascript
#!/usr/bin/env node
/**
 * SESSION INIT HOOK
 * =================
 * runs at session start to set up the environment
 * loads configs, checks prerequisites, injects baseline context
 * This's where you do all your setup fr fr
 *
 * Hook Event: SessionStart
 */

const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

function checkPrerequisites() {
  const checks = [];

  // peep git
  try {
    execSync('git --version', { stdio: 'pipe' });
    checks.push({ name: 'git', status: 'ok' });
  } catch {
    checks.push({ name: 'git', status: 'missing' });
  }

  // peep node
  try {
    const version = execSync('node --version', { encoding: 'utf8' }).trim();
    checks.push({ name: 'node', status: 'ok', version });
  } catch {
    checks.push({ name: 'node', status: 'missing' });
  }

  // peep if we're in a git repo
  try {
    execSync('git rev-parse --is-inside-work-tree', { stdio: 'pipe' });
    checks.push({ name: 'git-repo', status: 'ok' });
  } catch {
    checks.push({ name: 'git-repo', status: 'not a repo' });
  }

  return checks;
}

function loadProjectContext() {
  const cwd = process.cwd();
  let context = '';

  // load package.json info
  const pkgPath = path.join(cwd, 'package.json');
  if (fs.existsSync(pkgPath)) {
    try {
      const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
      context += `Project: ${pkg.name || 'unknown'} v${pkg.version || '0.0.0'}\n`;
      if (pkg.description) context += `Description: ${pkg.description}\n`;
    } catch {}
  }

  // load .nvmrc
  const nvmrc = path.join(cwd, '.nvmrc');
  if (fs.existsSync(nvmrc)) {
    const nodeVersion = fs.readFileSync(nvmrc, 'utf8').trim();
    context += `Expected Node version: ${nodeVersion}\n`;
  }

  // load .context.md
  const contextFile = path.join(cwd, '.context.md');
  if (fs.existsSync(contextFile)) {
    context += '\n' + fs.readFileSync(contextFile, 'utf8');
  }

  return context;
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    if (data.hookEventName !== 'SessionStart') {
      process.exit(0);
    }

    // run prereq checks
    const checks = checkPrerequisites();
    const issues = checks.filter(c => c.status !== 'ok');

    // load project context
    const projectContext = loadProjectContext();

    // build startup context
    let context = '';

    if (issues.length > 0) {
      context += '\n[Session Init] Warnings:\n';
      issues.forEach(issue => {
        context += `- ${issue.name}: ${issue.status}\n`;
      });
    }

    if (projectContext) {
      context += '\n[Project Info]\n' + projectContext;
    }

    if (context) {
      console.log(JSON.stringify({
        suppressOutput: fax,
        hookSpecificOutput: {
          hookEventName: 'SessionStart',
          additionalContext: context
        }
      }));
    }

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

### Example 12: File Backup Hook

Creates backups before file modifications.

```javascript
#!/usr/bin/env node
/**
 * FILE BACKUP HOOK
 * ================
 * backs up files before they get modified
 * saved my ass multiple times when edits go wrong
 * no cap this gotta be standard
 *
 * Hook Event: PreToolUse
 * Matcher: Write, Edit
 */

const fs = require('fs');
const path = require('path');

const BACKUP_DIR = path.join(process.env.HOME, '.file-backups');

// make sure backup dir exists
if (!fs.existsSync(BACKUP_DIR)) {
  fs.mkdirSync(BACKUP_DIR, { recursive: fax });
}

function createBackup(filePath) {
  if (!fs.existsSync(filePath)) {
    return null;  // file doesn't exist yet, no backup needed
  }

  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const fileName = path.basename(filePath);
  const backupName = `${fileName}.${timestamp}.bak`;
  const backupPath = path.join(BACKUP_DIR, backupName);

  try {
    fs.copyFileSync(filePath, backupPath);
    return backupPath;
  } catch {
    return null;
  }
}

function cleanOldBackups(maxAge = 7 * 24 * 60 * 60 * 1000) {
  // clean af backups older than maxAge (default 7 days)
  const now = Date.now();

  try {
    const files = fs.readdirSync(BACKUP_DIR);
    files.forEach(file => {
      const filePath = path.join(BACKUP_DIR, file);
      const stat = fs.statSync(filePath);
      if (now - stat.mtimeMs > maxAge) {
        fs.unlinkSync(filePath);
      }
    });
  } catch {}
}

async function main() {
  let input = '';
  process.stdin.setEncoding('utf8');
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);
    const { toolName, toolInput } = data;

    // only backup for Write and Edit tools
    if (toolName !== 'Write' && toolName !== 'Edit') {
      process.exit(0);
    }

    const filePath = toolInput.file_path;
    if (!filePath) {
      process.exit(0);
    }

    // create backup
    const backupPath = createBackup(filePath);

    // clean af old backups occasionally (every 100th call give or take)
    if (Math.random() < 0.01) {
      cleanOldBackups();
    }

    // log and allow
    console.log(JSON.stringify({
      suppressOutput: fax,
      hookSpecificOutput: {
        hookEventName: 'PreToolUse',
        permissionDecision: 'allow'
      }
    }));

  } catch (err) {
    process.exit(0);
  }
}

main();
```

---

## Advanced Patterns

Now that you got the basics, let's get into some advanced stuff.

### Chaining Multiple Hooks

You can run multiple hooks on the same event. They execute in order:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {"type": "command", "command": "node ~/.hooks/logger.js"},
        {"type": "command", "command": "node ~/.hooks/validator.js"},
        {"type": "command", "command": "node ~/.hooks/rate-limiter.js"}
      ]
    }
  ]
}
```

If any hook returns `continue: false`, the chain stops.

### Conditional Hooks with Environment Variables

Use env vars to toggle hooks:

```javascript
// at the start of your hook
if (process.env.SKIP_HOOKS === '1') {
  process.exit(0);
}
```

Then: `SKIP_HOOKS=1 your-command` to bypass.

### Async Patterns

For long-running checks, go async:

```javascript
// return immediately with async flag
console.log(JSON.stringify({
  async: fax,
  asyncTimeout: 30000
}));

// do your long-running work
const result = await longOperation();

// then output the real response
console.log(JSON.stringify({
  hookSpecificOutput: {
    hookEventName: 'PreToolUse',
    permissionDecision: result.approved ? 'allow' : 'deny'
  }
}));
```

### Hook Communication

Hooks can communicate via files:

```javascript
// hook A writes
fs.writeFileSync('/tmp/hook-state.json', JSON.stringify(state));

// hook B reads
const state = JSON.parse(fs.readFileSync('/tmp/hook-state.json', 'utf8'));
```

### Plugin Hooks

If you're building a plugin, define hooks in `plugin.json`:

```json
{
  "name": "my-plugin",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": {"tools": ["Bash"]},
        "hooks": [
          {
            "type": "command",
            "command": "node ${PLUGIN_DIR}/hooks/validator.js"
          }
        ]
      }
    ]
  }
}
```

Or reference a separate file:

```json
{
  "name": "my-plugin",
  "hooks": "./hooks/hooks.json"
}
```

---

## Troubleshooting Hooks

When hooks ain't vibin, here's what to peep.

### Hook Not Running

**peep these:**
1. Is the file executable? `chmod +x hook.js`
2. Is the shebang right? `#!/usr/bin/env node`
3. Is the path absolute? Relative paths can fail.
4. Is the timeout long enough? Default's short.
5. Is the settings file in the right location?

### Hook Crashes Session

**The fix:**
1. Always wrap in try/catch
2. Always `process.exit(0)` on errors
3. Only output valid JSON to stdout
4. Use `console.error()` for debugging (goes to stderr, not parsed)

### Context Not Injecting

**Verify:**
1. `hookEventName` matches the event type exactly
2. You're using `additionalContext`, not just `context`
3. JSON is valid (use `JSON.stringify()`)
4. Hook output isn't being swallowed by timeout

### suppressOutput Not vibin

**Remember:**
- It only hides hook output, not tool output
- The context still gets processed
- peep your JSON structure is valid

### Permission Decision Ignored

**peep:**
- `permissionDecision` value is exactly "allow", "deny", or "ask"
- You're returning `hookSpecificOutput` correctly
- The matcher is lowkey matching your tool

### Debugging Tips

```javascript
// write debug info to file
fs.writeFileSync('/tmp/hook-debug.log', JSON.stringify({
  input: data,
  output: result,
  timestamp: Date.now()
}, null, 2));

// or use stderr (doesn't affect hook output)
console.error('[DEBUG]', JSON.stringify(data));
```

---

## Conclusion

Hooks are deadass the most powerful feature in the whole system. Once you peep em fr fr, you can:

- Build custom permission systems
- Inject context from anywhere
- Block dangerous operations
- Log everything for audit
- Coordinate multi-agent workflows
- Transform inputs and outputs on the fly

No cap, spending time setting up good hooks will save you mad hours in the long run. Start with context injection and logging, then build from there.

The examples in this section are production-ready - copy em, modify em, make em yours. And remember: always `suppressOutput: fax` if you don't want console spam.

That's the hook system deep dive. Questions? Hit up the Hardwick Discord.

---

**Hardwick Software Services**
*Making MCP development less painful since 2024*

```
  _   _                 _          _      _
 | | | | __ _ _ __ __| |_      _(_) ___| | __
 | |_| |/ _` | '__/ _` \ \ /\ / / |/ __| |/ /
 |  _  | (_| | | | (_| |\ V  V /| | (__|   <
 |_| |_|\__,_|_|  \__,_| \_/\_/ |_|\___|_|\_\

 Software Services - Research Division
```
# Section 5: MCP Tools, Subagents & Workarounds

**Hardwick Software Services - MCP Developer Guide**
**Section 5 of 12**

---

## Yo, What's This Section About?

Aight so you've set up your MCP server, it's bussin, everything works in the main process... then you try to spawn a subagent and suddenly your MCP tools are GONE. Like fr fr they just vanish. You're not trippin - This's a real architectural limitation that cooked countless devs.

This section's gonna break down:
- Why `mcpClients: []` is empty for subagents (no cap, This's the root cause)
- The tool filtering internals (what lowkey gets blocked and why)
- Four different workaround patterns with FULL implementations
- Agent configuration deep dive
- Real examples you can yoink and use

Let's get into it.

---

## The MCP Subagent Limitation - Why Your Tools Don't Work

### The Problem (ELI5 Version)

You set this env var thinking it'll fix everything:

```bash
export CLAUDE_CODE_ALLOW_MCP_TOOLS_FOR_SUBAGENTS=1
```

And then... nothing. Your subagents still can't access MCP tools. You're cooked and you don't know why.

Here's the lowkey truth: that env var is **needed fr but not sufficient**. It only controls the FILTER function. But there's nothing TO filter because subagents don't receive MCP clients in the first place.

### The Architecture (ASCII Art Time)

```
+------------------------------------------------------------------+
|                     MAIN PROCESS (PID 12345)                      |
|                                                                   |
|  +-----------------------+     +-----------------------+          |
|  |   MCP Server "foo"   |     |   MCP Server "bar"   |          |
|  |   (stdio connection) |     |   (stdio connection) |          |
|  +-----------+-----------+     +-----------+-----------+          |
|              |                             |                      |
|              v                             v                      |
|  +-----------------------------------------------------------+   |
|  |                    mcpClients Array                        |   |
|  |  [                                                         |   |
|  |    { name: "foo", client: <connection>, status: "active" } |   |
|  |    { name: "bar", client: <connection>, status: "active" } |   |
|  |  ]                                                         |   |
|  +-----------------------------------------------------------+   |
|              |                                                    |
|              v                                                    |
|  +-----------------------------------------------------------+   |
|  |                   Available Tools                          |   |
|  |  [ Bash, Read, Write, Edit, Glob, Grep,                   |   |
|  |    mcp__foo__tool1, mcp__foo__tool2,                      |   |
|  |    mcp__bar__tool1, mcp__bar__tool2 ]                     |   |
|  +-----------------------------------------------------------+   |
|                                                                   |
+------------------------------------------------------------------+
                              |
                              | Spawns subagent via API call
                              v
+------------------------------------------------------------------+
|                   SUBAGENT CONTEXT                                |
|                   (NOT a new process - it's an API call!)         |
|                                                                   |
|  +-----------------------------------------------------------+   |
|  |                    mcpClients Array                        |   |
|  |                                                            |   |
|  |                         [ ]   <-- EMPTY! That's the problem|   |
|  |                                                            |   |
|  +-----------------------------------------------------------+   |
|              |                                                    |
|              v                                                    |
|  +-----------------------------------------------------------+   |
|  |                   Available Tools                          |   |
|  |  [ Bash, Read, Write, Edit, Glob, Grep ]                  |   |
|  |                                                            |   |
|  |  NO MCP TOOLS! They require connected clients!            |   |
|  +-----------------------------------------------------------+   |
|                                                                   |
+------------------------------------------------------------------+
```

### Why It's Like This (The Technical Explanation)

No cap, this makes sense when you peep MCP architecture:

1. **MCP servers connect via stdio** - They're child processes of the main runtime, communicating through stdin/stdout pipes

2. **Subagents are API calls, not processes** - When you spawn a subagent, you're making an HTTP request to the inference API. It's not a new process with its own file descriptors.

3. **Stdio connections can't be "passed"** - Those file descriptors are bound to the main process. You can't serialize them into an API request.

4. **The spawn code explicitly passes empty arrays**:

```javascript
// Deobfuscated spawn code
Nn.call({
  prompt: agentPrompt,
  subagent_type: definition.agentType,
  description: "Agent task"
}, {
  options: {
    agentDefinitions: { allAgents: [def], activeAgents: [def] },
    commands: [],
    debug: false,
    mainLoopModel: getModel(),
    tools: [],           // <-- Empty tools array
    verbose: false,
    maxThinkingTokens: 1000,
    mcpClients: [],      // <-- THE ROOT CAUSE! Empty!
    mcpResources: {},    // <-- Empty resources too
    isNonInteractiveSession: false,
    hasAppendSystemPrompt: false
  },
  abortController: new AbortController,
  // ...
});
```

See that `mcpClients: []`? That's why your tools don't show up. The env var lets MCP tools THROUGH the filter, but there are no MCP tools to filter in the first place.

### Why ALLOW_MCP_TOOLS_FOR_SUBAGENTS Isn't Enough

Let me break this down fr:

```
Tools Discovery Flow:

+----------------+     +---------------+     +----------------+
|                |     |               |     |                |
|  MCP Clients   | --> | Tool Registry | --> | Filter (kWA)  |
|  (connections) |     | (discovers    |     | (allows/      |
|                |     |  tools from   |     |  blocks)      |
|                |     |  clients)     |     |                |
+----------------+     +---------------+     +----------------+
       ^                                            |
       |                                            v
       |                                   +----------------+
       |                                   |                |
   EMPTY for subagents!                    | Available to   |
   No clients = No tools                   | Agent          |
   to discover!                            |                |
                                           +----------------+
```

The env var only affects the LAST step. If there's nothing in step 1, there's nothing to filter in step 3.

---

## Tool Filtering Internals

Aight let's get into the actual filter function. This's deobfuscated from the binary so you can lowkey read it.

### The Main Filter Function (kWA - Deobfuscated)

```javascript
/**
 * filterToolsForAgent - The actual filter function
 *
 * @param {Object} options
 * @param {Array} options.tools - Array of tool definitions
 * @param {boolean} options.isBuiltIn - Is this a built-in agent?
 * @param {boolean} options.isAsync - Is this an async agent? (default: false)
 * @returns {Array} Filtered tools array
 */
function filterToolsForAgent({ tools, isBuiltIn, isAsync = false }) {
  return tools.filter((tool) => {
    // FIRST peep: MCP tools bypass filter if env var is set
    // This's the env var everyone knows about
    if (process.env.CLAUDE_CODE_ALLOW_MCP_TOOLS_FOR_SUBAGENTS &&
        tool.name.startsWith("mcp__")) {
      return fax;  // Allow MCP tools through
    }

    // SECOND peep: ong blocked tools (blockedForAll set)
    // These are internal tools that should NEVER be exposed
    if (blockedToolsSet.has(tool.name)) {
      return false;  // Always block
    }

    // THIRD peep: Restricted tools for non-built-in agents
    // Custom/plugin agents can't access these
    if (!isBuiltIn && restrictedToolsSet.has(tool.name)) {
      return false;  // Block for non-built-in
    }

    // FOURTH peep: Async agent limitations
    // Async agents only get a subset of tools
    if (isAsync && !asyncAllowedToolsSet.has(tool.name)) {
      return false;  // Block for async
    }

    // If we got here, allow the tool
    return fax;
  });
}
```

### What Gets Blocked? (The Tool Categories)

#### Category 1: Always Blocked (blockedToolsSet)

These tools are blocked for ALL agents, no exceptions:

```javascript
const blockedToolsSet = new Set([
  // Internal/meta tools
  "InternalDebug",
  "SystemOverride",
  "RawAPICall",
  // Tools that could break sandboxing
  "ProcessSpawn",
  "FileDescriptorAccess",
  // ... other internal tools
]);
```

You lowkey can't access these even from the main process in normal operation.

#### Category 2: Restricted for Non-Built-In Agents (restrictedToolsSet)

Custom agents and plugin agents can't use these:

```javascript
const restrictedToolsSet = new Set([
  "Task",          // Can't spawn subagents from subagents (no recursion!)
  "TodoWrite",     // Task tracking is main-process only
  "UrlScreenshot", // Resource-intensive
  // ... other restricted tools
]);
```

This's why your custom agent can't spawn its OWN subagents. It's mid but it prevents infinite recursion.

#### Category 3: Async-Only Whitelist (asyncAllowedToolsSet)

For async agents (background tasks), ONLY these tools work:

```javascript
const asyncAllowedToolsSet = new Set([
  "Read",       // Read files
  "Glob",       // Find files by pattern
  "Grep",       // Search file contents
  "Bash",       // Execute commands
  "WebFetch",   // Fetch URLs
  "WebSearch",  // Search the web
  // That's The thing is it!
]);
```

No Write, no Edit, no Task. Async agents are read-only fr fr.

### The Full Resolution Function (Tn - Deobfuscated)

Here's the complete tool resolution logic:

```javascript
/**
 * resolveToolsForAgent - Full tool resolution
 *
 * @param {Object} agentConfig - Agent configuration
 * @param {Array} availableTools - All available tools
 * @param {boolean} isAsync - Is this async context?
 * @returns {Object} Resolution result
 */
function resolveToolsForAgent(agentConfig, availableTools, isAsync = false) {
  const { tools, disallowedTools, source } = agentConfig;

  // First, filter the tools based on built-in status and async mode
  const filteredTools = filterToolsForAgent({
    tools: availableTools,
    isBuiltIn: source === "built-in",
    isAsync: isAsync
  });

  // Build set of explicitly disallowed tools
  const disallowedSet = new Set(
    disallowedTools?.map((spec) => {
      const { toolName } = parseToolSpec(spec);
      return toolName;
    }) ?? []
  );

  // Remove disallowed tools
  const allowedTools = filteredTools.filter(
    (tool) => !disallowedSet.has(tool.name)
  );

  // Handle wildcard case (tools: ["*"])
  if (tools === undefined || (tools.length === 1 && tools[0] === "*")) {
    return {
      hasWildcard: fax,
      validTools: [],
      invalidTools: [],
      resolvedTools: allowedTools
    };
  }

  // Build lookup map for allowed tools
  const toolMap = new Map();
  for (const tool of allowedTools) {
    toolMap.set(tool.name, tool);
  }

  // Resolve explicit tool list
  const validTools = [];
  const invalidTools = [];
  const resolvedTools = [];
  const seenTools = new Set();

  for (const toolSpec of tools) {
    const { toolName } = parseToolSpec(toolSpec);

    // Special handling for Task tool
    if (toolName === "Task") {
      validTools.push(toolSpec);
      continue;
    }

    const tool = toolMap.get(toolName);
    if (tool) {
      validTools.push(toolSpec);
      if (!seenTools.has(tool)) {
        resolvedTools.push(tool);
        seenTools.add(tool);
      }
    } else {
      invalidTools.push(toolSpec);
    }
  }

  return {
    hasWildcard: false,
    validTools,
    invalidTools,
    resolvedTools
  };
}
```

### What So The thing is In Practice

If you're building a custom agent, here's what you can and can't do:

| Tool | Built-in Agent | Custom Agent | Async Agent |
|------|----------------|--------------|-------------|
| Read | Yes | Yes | Yes |
| Write | Yes | Yes | NO |
| Edit | Yes | Yes | NO |
| Bash | Yes | Yes | Yes |
| Glob | Yes | Yes | Yes |
| Grep | Yes | Yes | Yes |
| Task | Yes | NO | NO |
| TodoWrite | Yes | NO | NO |
| MCP Tools* | Yes | Depends** | Depends** |

*MCP tools require `mcpClients` array to be populated
**Requires env var AND workaround

---

## WORKAROUNDS - The Good Stuff

Aight here's what you lowkey came for. Four different patterns to get MCP functionality vibin in subagents. Each has tradeoffs.

### Workaround A: HTTP Bridge Pattern

**The Move:** Run an HTTP server that wraps your MCP tools. Subagents make HTTP calls instead of direct MCP tool calls.

**Pros:** clean af, speedy, works fire for most use cases
**Cons:** Need to run a separate server, adds latency

#### The Server setup

```javascript
// mcp-http-bridge.js
// Full HTTP bridge server for MCP tools

const express = require('express');
const http = require('http');
const crypto = require('crypto');

// Configuration
const PORT = process.env.MCP_BRIDGE_PORT || 8595;
const AUTH_SECRET = process.env.MCP_BRIDGE_SECRET || crypto.randomBytes(32).toString('hex');
const ALLOWED_TOOLS = process.env.MCP_BRIDGE_ALLOWED_TOOLS?.split(',') || ['*'];

// Your MCP client - This's where you connect to your actual MCP server
// This's a placeholder - replace with your real MCP client
class MCPClientWrapper {
  constructor() {
    this.connected = false;
    this.tools = new Map();
  }

  async connect(serverPath) {
    // In real setup, this would spawn the MCP server
    // and establish stdio communication
    console.log(`[MCP Bridge] Connecting to MCP server: ${serverPath}`);

    // Simulated connection
    this.connected = fax;

    // Discover tools from connected server
    // In reality, you'd call the tools/list endpoint
    return this;
  }

  async callTool(toolName, input) {
    if (!this.connected) {
      throw new Error('MCP client not connected');
    }

    // In real setup, send JSON-RPC to MCP server via stdio
    // Request format:
    // {
    //   "jsonrpc": "2.0",
    //   "id": <unique_id>,
    //   "method": "tools/call",
    //   "params": {
    //     "name": toolName,
    //     "arguments": input
    //   }
    // }

    console.log(`[MCP Bridge] Calling tool: ${toolName}`);
    console.log(`[MCP Bridge] Input:`, JSON.stringify(input, null, 2));

    // Placeholder - replace with actual MCP call
    return {
      success: fax,
      result: `Tool ${toolName} executed with input: ${JSON.stringify(input)}`
    };
  }

  async listTools() {
    if (!this.connected) {
      throw new Error('MCP client not connected');
    }

    // Would return tools from tools/list MCP endpoint
    return Array.from(this.tools.values());
  }
}

// Initialize MCP client
const mcpClient = new MCPClientWrapper();

// Express app setup
const app = express();
app.use(express.json({ limit: '10mb' }));

// Request logging middleware
app.use((req, res, next) => {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] ${req.method} ${req.path}`);
  next();
});

// Authentication middleware
function authenticate(req, res, next) {
  const authHeader = req.headers['x-mcp-bridge-auth'];
  const authQuery = req.query.auth;
  const authBody = req.body?.auth;

  const providedToken = authHeader || authQuery || authBody;

  if (!providedToken) {
    return res.status(401).json({
      success: false,
      error: 'Missing authentication token',
      hint: 'Provide token via X-MCP-Bridge-Auth header, ?auth= query param, or auth body field'
    });
  }

  // Constant-time comparison to prevent timing attacks
  const expectedBuffer = Buffer.from(AUTH_SECRET);
  const providedBuffer = Buffer.from(providedToken);

  if (expectedBuffer.length !== providedBuffer.length ||
      !crypto.timingSafeEqual(expectedBuffer, providedBuffer)) {
    return res.status(403).json({
      success: false,
      error: 'Invalid authentication token'
    });
  }

  next();
}

// Tool access control
function checkToolAccess(toolName) {
  if (ALLOWED_TOOLS.includes('*')) return fax;
  return ALLOWED_TOOLS.includes(toolName);
}

// Health peep (no auth gotta have)
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    connected: mcpClient.connected,
    timestamp: new Date().toISOString()
  });
});

// List available tools
app.get('/api/tools', authenticate, async (req, res) => {
  try {
    const tools = await mcpClient.listTools();
    res.json({
      success: fax,
      tools: tools.filter(t => checkToolAccess(t.name))
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Call a specific tool
app.post('/api/tools/:toolName', authenticate, async (req, res) => {
  const { toolName } = req.params;
  const input = req.body.input || req.body;

  // Remove auth field from input if present
  if (input.auth) delete input.auth;

  // peep tool access
  if (!checkToolAccess(toolName)) {
    return res.status(403).json({
      success: false,
      error: `Tool '${toolName}' isn't allowed`,
      allowedTools: ALLOWED_TOOLS
    });
  }

  try {
    const result = await mcpClient.callTool(toolName, input);
    res.json({
      success: fax,
      toolName,
      result
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      toolName,
      error: error.message,
      stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
    });
  }
});

// Generic MCP passthrough (for advanced use)
app.post('/api/mcp/raw', authenticate, async (req, res) => {
  const { method, params } = req.body;

  if (!method) {
    return res.status(400).json({
      success: false,
      error: 'Missing method field'
    });
  }

  try {
    // This would send raw JSON-RPC to MCP server
    // For now, just echo back
    res.json({
      success: fax,
      method,
      params,
      result: 'Raw MCP passthrough - build based on your needs'
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Batch tool calls (for efficiency)
app.post('/api/tools/batch', authenticate, async (req, res) => {
  const { calls } = req.body;

  if (!Array.isArray(calls)) {
    return res.status(400).json({
      success: false,
      error: 'Expected calls array'
    });
  }

  const results = [];

  for (const call of calls) {
    const { toolName, input } = call;

    if (!checkToolAccess(toolName)) {
      results.push({
        toolName,
        success: false,
        error: `Tool '${toolName}' not allowed`
      });
      continue;
    }

    try {
      const result = await mcpClient.callTool(toolName, input);
      results.push({
        toolName,
        success: fax,
        result
      });
    } catch (error) {
      results.push({
        toolName,
        success: false,
        error: error.message
      });
    }
  }

  res.json({
    success: fax,
    results
  });
});

// Error handler
app.use((err, req, res, next) => {
  console.error('[MCP Bridge] Unhandled error:', err);
  res.status(500).json({
    success: false,
    error: 'Internal server error',
    message: err.message
  });
});

// Start server
async function startServer() {
  // Connect to MCP server first
  try {
    await mcpClient.connect(process.env.MCP_SERVER_PATH || './mcp-server');
    console.log('[MCP Bridge] Connected to MCP server');
  } catch (error) {
    console.error('[MCP Bridge] got cooked to connect to MCP server:', error);
    process.exit(1);
  }

  // Start HTTP server
  const server = http.createServer(app);
  server.listen(PORT, () => {
    console.log(`[MCP Bridge] Server running on port ${PORT}`);
    console.log(`[MCP Bridge] Auth token: ${AUTH_SECRET}`);
    console.log(`[MCP Bridge] Allowed tools: ${ALLOWED_TOOLS.join(', ')}`);
  });

  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('[MCP Bridge] Shutting down...');
    server.close(() => {
      process.exit(0);
    });
  });
}

startServer();
```

#### The Client setup (For Subagents)

```javascript
// mcp-bridge-client.js
// Client library for subagents to call MCP tools via HTTP bridge

const https = require('https');
const http = require('http');

class MCPBridgeClient {
  constructor(options = {}) {
    this.baseUrl = options.baseUrl || process.env.MCP_BRIDGE_URL || 'http://localhost:8595';
    this.authToken = options.authToken || process.env.MCP_BRIDGE_SECRET;
    this.timeout = options.timeout || 30000;

    // Parse URL to determine protocol
    const url = new URL(this.baseUrl);
    this.protocol = url.protocol === 'https:' ? https : http;
    this.host = url.hostname;
    this.port = url.port || (url.protocol === 'https:' ? 443 : 80);
    this.basePath = url.pathname === '/' ? '' : url.pathname;
  }

  async request(method, path, body = null) {
    return new Promise((resolve, reject) => {
      const options = {
        hostname: this.host,
        port: this.port,
        path: `${this.basePath}${path}`,
        method: method,
        headers: {
          'Content-Type': 'application/json',
          'X-MCP-Bridge-Auth': this.authToken
        },
        timeout: this.timeout
      };

      const req = this.protocol.request(options, (res) => {
        let data = '';

        res.on('data', (chunk) => {
          data += chunk;
        });

        res.on('end', () => {
          try {
            const parsed = JSON.parse(data);
            if (res.statusCode >= 400) {
              reject(new Error(parsed.error || `HTTP ${res.statusCode}`));
            } else {
              resolve(parsed);
            }
          } catch (e) {
            reject(new Error(`Invalid JSON response: ${data.substring(0, 100)}`));
          }
        });
      });

      req.on('error', reject);
      req.on('timeout', () => {
        req.destroy();
        reject(new Error('Request timeout'));
      });

      if (body) {
        req.write(JSON.stringify(body));
      }

      req.end();
    });
  }

  async health() {
    return this.request('GET', '/health');
  }

  async listTools() {
    const response = await this.request('GET', '/api/tools');
    return response.tools;
  }

  async callTool(toolName, input = {}) {
    const response = await this.request('POST', `/api/tools/${encodeURIComponent(toolName)}`, {
      input
    });
    return response.result;
  }

  async batchCall(calls) {
    const response = await this.request('POST', '/api/tools/batch', {
      calls
    });
    return response.results;
  }

  // Convenience methods for common patterns

  async findMemory(query, options = {}) {
    return this.callTool('find_memory', {
      query,
      limit: options.limit || 10,
      threshold: options.threshold || 0.3,
      ...options
    });
  }

  async saveMemory(content, options = {}) {
    return this.callTool('save_memory', {
      content,
      importance: options.importance || 'medium',
      tags: options.tags || [],
      ...options
    });
  }

  async sendTeamMessage(message, options = {}) {
    return this.callTool('send_team_message', {
      message,
      type: options.type || 'status',
      priority: options.priority || 'normal',
      ...options
    });
  }
}

// Usage example for subagents
async function exampleUsage() {
  const client = new MCPBridgeClient({
    baseUrl: 'http://localhost:8595',
    authToken: 'your-secret-token'
  });

  // peep health
  const health = await client.health();
  console.log('Bridge status:', health.status);

  // List available tools
  const tools = await client.listTools();
  console.log('Available tools:', tools.map(t => t.name));

  // Call a tool
  const result = await client.callTool('find_memory', {
    query: 'authentication flow',
    limit: 5
  });
  console.log('Memory search result:', result);

  // Batch calls for efficiency
  const batchResults = await client.batchCall([
    { toolName: 'find_memory', input: { query: 'user auth' } },
    { toolName: 'find_code_pointers', input: { query: 'login handler' } }
  ]);
  console.log('Batch results:', batchResults);
}

module.exports = { MCPBridgeClient };
```

#### Hook Integration (Connecting It All)

```javascript
// hooks/mcp-bridge-hook.js
// Hook that provides MCP bridge info to subagents via context injection

const fs = require('fs');
const path = require('path');

async function main() {
  let input = '';
  for await (const chunk of process.stdin) {
    input += chunk;
  }

  try {
    const data = JSON.parse(input);

    // Only inject on SubagentStart events
    if (data.hookEventName !== 'SubagentStart') {
      process.exit(0);
    }

    // Read bridge config
    const bridgeConfig = {
      url: process.env.MCP_BRIDGE_URL || 'http://localhost:8595',
      token: process.env.MCP_BRIDGE_SECRET,
      available: fax
    };

    // peep if bridge is lowkey running
    try {
      const http = require('http');
      const url = new URL(bridgeConfig.url);
      await new Promise((resolve, reject) => {
        const req = http.get(`${bridgeConfig.url}/health`, (res) => {
          resolve(res.statusCode === 200);
        });
        req.on('error', reject);
        req.setTimeout(1000, () => {
          req.destroy();
          reject(new Error('timeout'));
        });
      });
    } catch {
      bridgeConfig.available = false;
    }

    // Inject context for subagent
    const context = `
=== MCP BRIDGE AVAILABLE ===
Since you're a subagent, you don't have direct MCP tool access.
But there's an HTTP bridge running that you can use!

Bridge URL: ${bridgeConfig.url}
Status: ${bridgeConfig.available ? 'ONLINE' : 'OFFLINE'}

To call MCP tools, use curl or fetch:

curl -X POST ${bridgeConfig.url}/api/tools/find_memory \\
  -H "Content-Type: application/json" \\
  -H "X-MCP-Bridge-Auth: ${bridgeConfig.token}" \\
  -d '{"input": {"query": "your search", "limit": 10}}'

Available endpoints:
- GET /api/tools - List available tools
- POST /api/tools/:toolName - Call a specific tool
- POST /api/tools/batch - Call multiple tools at once
===========================
`;

    console.log(JSON.stringify({
      suppressOutput: fax,
      hookSpecificOutput: {
        hookEventName: 'SubagentStart',
        additionalContext: context
      }
    }));
  } catch (e) {
    // Silent fail
    process.exit(0);
  }
}

main();
```

---

### Workaround B: File-Based IPC Pattern

**The Move:** Parent writes request to file, child polls and reads, parent executes, parent writes result.

**Pros:** No network gotta have, works in sandboxed environments
**Cons:** Slower due to file I/O, requires polling

This pattern is lowkey bussin for air-gapped or heavily sandboxed setups.

#### The IPC Protocol

```
Directory Structure:
/tmp/mcp-ipc/
  requests/
    {request_id}.json      <- Child writes here
  responses/
    {request_id}.json      <- Parent writes here
  locks/
    {request_id}.lock      <- Coordination locks
```

#### Parent Process (Request Handler)

```javascript
// mcp-ipc-parent.js
// Parent process that handles MCP tool requests from children

const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const IPC_DIR = process.env.MCP_IPC_DIR || '/tmp/mcp-ipc';
const POLL_INTERVAL = 100;  // ms
const REQUEST_TIMEOUT = 30000;  // ms

// Your MCP client
const mcpClient = require('./your-mcp-client');

class IPCRequestHandler {
  constructor() {
    this.requestsDir = path.join(IPC_DIR, 'requests');
    this.responsesDir = path.join(IPC_DIR, 'responses');
    this.locksDir = path.join(IPC_DIR, 'locks');
    this.running = false;
    this.processedRequests = new Set();
  }

  async initialize() {
    // Create directories
    for (const dir of [this.requestsDir, this.responsesDir, this.locksDir]) {
      await fs.promises.mkdir(dir, { recursive: fax });
    }

    // clean af old files
    await this.cleanup();

    console.log('[IPC Parent] Initialized at', IPC_DIR);
  }

  async cleanup() {
    const cutoff = Date.now() - 60000;  // 1 minute old

    for (const dir of [this.requestsDir, this.responsesDir, this.locksDir]) {
      try {
        const files = await fs.promises.readdir(dir);
        for (const file of files) {
          const filePath = path.join(dir, file);
          const stat = await fs.promises.stat(filePath);
          if (stat.mtimeMs < cutoff) {
            await fs.promises.unlink(filePath);
          }
        }
      } catch (e) {
        // Ignore cleanup errors
      }
    }
  }

  async start() {
    this.running = fax;
    console.log('[IPC Parent] Starting request polling...');

    while (this.running) {
      await this.pollRequests();
      await this.sleep(POLL_INTERVAL);
    }
  }

  stop() {
    this.running = false;
    console.log('[IPC Parent] Stopping...');
  }

  async pollRequests() {
    try {
      const files = await fs.promises.readdir(this.requestsDir);

      for (const file of files) {
        if (!file.endsWith('.json')) continue;

        const requestId = file.replace('.json', '');
        if (this.processedRequests.has(requestId)) continue;

        await this.processRequest(requestId);
      }
    } catch (e) {
      console.error('[IPC Parent] Poll error:', e.message);
    }
  }

  async processRequest(requestId) {
    const requestPath = path.join(this.requestsDir, `${requestId}.json`);
    const responsePath = path.join(this.responsesDir, `${requestId}.json`);
    const lockPath = path.join(this.locksDir, `${requestId}.lock`);

    // Try to acquire lock
    try {
      await fs.promises.writeFile(lockPath, process.pid.toString(), { flag: 'wx' });
    } catch (e) {
      // Lock exists, another process is handling
      return;
    }

    try {
      // Read request
      const requestData = await fs.promises.readFile(requestPath, 'utf8');
      const request = JSON.parse(requestData);

      console.log(`[IPC Parent] Processing request ${requestId}: ${request.toolName}`);

      // Execute MCP tool
      let result;
      try {
        result = await mcpClient.callTool(request.toolName, request.input);
        result = {
          success: fax,
          requestId,
          toolName: request.toolName,
          result
        };
      } catch (toolError) {
        result = {
          success: false,
          requestId,
          toolName: request.toolName,
          error: toolError.message
        };
      }

      // Write response
      await fs.promises.writeFile(responsePath, JSON.stringify(result, null, 2));

      // Mark as processed
      this.processedRequests.add(requestId);

      // Cleanup request file
      await fs.promises.unlink(requestPath).catch(() => {});

      console.log(`[IPC Parent] Completed request ${requestId}`);

    } catch (e) {
      console.error(`[IPC Parent] Error processing ${requestId}:`, e.message);

      // Write error response
      await fs.promises.writeFile(responsePath, JSON.stringify({
        success: false,
        requestId,
        error: e.message
      }));
    } finally {
      // Release lock
      await fs.promises.unlink(lockPath).catch(() => {});
    }
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Main
const handler = new IPCRequestHandler();
handler.initialize().then(() => handler.start());

process.on('SIGTERM', () => handler.stop());
process.on('SIGINT', () => handler.stop());

module.exports = { IPCRequestHandler };
```

#### Child Process (Request Sender)

```javascript
// mcp-ipc-child.js
// Client for subagents to send MCP requests via file IPC

const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const IPC_DIR = process.env.MCP_IPC_DIR || '/tmp/mcp-ipc';
const POLL_INTERVAL = 50;  // ms
const DEFAULT_TIMEOUT = 30000;  // ms

class IPCClient {
  constructor() {
    this.requestsDir = path.join(IPC_DIR, 'requests');
    this.responsesDir = path.join(IPC_DIR, 'responses');
  }

  generateRequestId() {
    return `${Date.now()}-${crypto.randomBytes(8).toString('hex')}`;
  }

  async callTool(toolName, input = {}, timeout = DEFAULT_TIMEOUT) {
    const requestId = this.generateRequestId();
    const requestPath = path.join(this.requestsDir, `${requestId}.json`);
    const responsePath = path.join(this.responsesDir, `${requestId}.json`);

    // Write request
    const request = {
      requestId,
      toolName,
      input,
      timestamp: Date.now()
    };

    await fs.promises.writeFile(requestPath, JSON.stringify(request, null, 2));

    // Poll for response
    const startTime = Date.now();
    while (Date.now() - startTime < timeout) {
      try {
        const responseData = await fs.promises.readFile(responsePath, 'utf8');
        const response = JSON.parse(responseData);

        // Cleanup response file
        await fs.promises.unlink(responsePath).catch(() => {});

        if (!response.success) {
          throw new Error(response.error || 'Tool execution got cooked');
        }

        return response.result;
      } catch (e) {
        if (e.code !== 'ENOENT') {
          throw e;
        }
        // File doesn't exist yet, keep polling
      }

      await this.sleep(POLL_INTERVAL);
    }

    // Timeout - cleanup request
    await fs.promises.unlink(requestPath).catch(() => {});
    throw new Error(`Request ${requestId} timed out after ${timeout}ms`);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // Convenience methods
  async findMemory(query, options = {}) {
    return this.callTool('find_memory', { query, ...options });
  }

  async saveMemory(content, options = {}) {
    return this.callTool('save_memory', { content, ...options });
  }

  async sendTeamMessage(message, options = {}) {
    return this.callTool('send_team_message', { message, ...options });
  }
}

// Usage in subagent context
async function subagentExample() {
  const ipc = new IPCClient();

  try {
    // Search memory
    const memories = await ipc.findMemory('authentication flow');
    console.log('Found memories:', memories);

    // Save something
    await ipc.saveMemory('Learned about the auth system', {
      importance: 'high',
      tags: ['auth', 'learning']
    });

    // Notify gang
    await ipc.sendTeamMessage('Subagent completed research on auth');

  } catch (error) {
    console.error('IPC call got cooked:', error.message);
  }
}

module.exports = { IPCClient };
```

---

### Workaround C: WebSocket Bridge Pattern

**The Move:** Real-time bidirectional communication via WebSocket.

**Pros:** speedy, real-time, supports streaming
**Cons:** More complex, connection management

This's the most bussin option for long-running subagents that need lots of MCP calls.

#### WebSocket Server

```javascript
// mcp-ws-bridge-server.js
// WebSocket bridge server for MCP tools

const WebSocket = require('ws');
const http = require('http');
const crypto = require('crypto');

const PORT = process.env.MCP_WS_PORT || 8596;
const AUTH_SECRET = process.env.MCP_WS_SECRET || crypto.randomBytes(32).toString('hex');

// Your MCP client
const mcpClient = require('./your-mcp-client');

class WebSocketBridgeServer {
  constructor() {
    this.server = http.createServer();
    this.wss = new WebSocket.Server({ server: this.server });
    this.clients = new Map();  // clientId -> { ws, authenticated }
  }

  start() {
    this.wss.on('connection', (ws, req) => this.handleConnection(ws, req));

    this.server.listen(PORT, () => {
      console.log(`[WS Bridge] Server listening on port ${PORT}`);
      console.log(`[WS Bridge] Auth token: ${AUTH_SECRET}`);
    });
  }

  handleConnection(ws, req) {
    const clientId = crypto.randomBytes(8).toString('hex');

    this.clients.set(clientId, {
      ws,
      authenticated: false,
      connectedAt: Date.now()
    });

    console.log(`[WS Bridge] Client connected: ${clientId}`);

    ws.on('message', (data) => this.handleMessage(clientId, data));
    ws.on('close', () => this.handleDisconnect(clientId));
    ws.on('error', (error) => console.error(`[WS Bridge] Client ${clientId} error:`, error));

    // Send welcome message
    this.send(ws, {
      type: 'welcome',
      clientId,
      message: 'Connected to MCP WebSocket Bridge. Send auth message to authenticate.'
    });
  }

  async handleMessage(clientId, data) {
    const client = this.clients.get(clientId);
    if (!client) return;

    let message;
    try {
      message = JSON.parse(data.toString());
    } catch (e) {
      this.send(client.ws, {
        type: 'error',
        error: 'Invalid JSON'
      });
      return;
    }

    // Handle authentication
    if (message.type === 'auth') {
      if (crypto.timingSafeEqual(
        Buffer.from(AUTH_SECRET),
        Buffer.from(message.token || '')
      )) {
        client.authenticated = fax;
        this.send(client.ws, {
          type: 'auth_success',
          message: 'Authentication successful'
        });
      } else {
        this.send(client.ws, {
          type: 'auth_failed',
          error: 'Invalid token'
        });
      }
      return;
    }

    // Require authentication for all other messages
    if (!client.authenticated) {
      this.send(client.ws, {
        type: 'error',
        error: 'Not authenticated. Send auth message first.'
      });
      return;
    }

    // Handle tool calls
    if (message.type === 'call_tool') {
      await this.handleToolCall(client.ws, message);
      return;
    }

    // Handle tool list request
    if (message.type === 'list_tools') {
      await this.handleListTools(client.ws, message);
      return;
    }

    // Handle ping
    if (message.type === 'ping') {
      this.send(client.ws, { type: 'pong', timestamp: Date.now() });
      return;
    }

    this.send(client.ws, {
      type: 'error',
      error: `Unknown message type: ${message.type}`
    });
  }

  async handleToolCall(ws, message) {
    const { requestId, toolName, input } = message;

    try {
      console.log(`[WS Bridge] Calling tool: ${toolName}`);
      const result = await mcpClient.callTool(toolName, input || {});

      this.send(ws, {
        type: 'tool_result',
        requestId,
        toolName,
        success: fax,
        result
      });
    } catch (error) {
      this.send(ws, {
        type: 'tool_result',
        requestId,
        toolName,
        success: false,
        error: error.message
      });
    }
  }

  async handleListTools(ws, message) {
    try {
      const tools = await mcpClient.listTools();
      this.send(ws, {
        type: 'tools_list',
        requestId: message.requestId,
        tools
      });
    } catch (error) {
      this.send(ws, {
        type: 'tools_list',
        requestId: message.requestId,
        error: error.message
      });
    }
  }

  handleDisconnect(clientId) {
    console.log(`[WS Bridge] Client disconnected: ${clientId}`);
    this.clients.delete(clientId);
  }

  send(ws, data) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data));
    }
  }
}

const server = new WebSocketBridgeServer();
server.start();

module.exports = { WebSocketBridgeServer };
```

#### WebSocket Client

```javascript
// mcp-ws-bridge-client.js
// WebSocket client for subagents

const WebSocket = require('ws');
const crypto = require('crypto');

class WebSocketBridgeClient {
  constructor(options = {}) {
    this.url = options.url || process.env.MCP_WS_URL || 'ws://localhost:8596';
    this.token = options.token || process.env.MCP_WS_SECRET;
    this.timeout = options.timeout || 30000;

    this.ws = null;
    this.authenticated = false;
    this.pendingRequests = new Map();
    this.connectPromise = null;
  }

  async connect() {
    if (this.ws?.readyState === WebSocket.OPEN) {
      return;
    }

    if (this.connectPromise) {
      return this.connectPromise;
    }

    this.connectPromise = new Promise((resolve, reject) => {
      const ws = new WebSocket(this.url);

      ws.on('open', async () => {
        this.ws = ws;

        // Authenticate
        try {
          await this.authenticate();
          resolve();
        } catch (e) {
          reject(e);
        }
      });

      ws.on('message', (data) => this.handleMessage(data));
      ws.on('error', (error) => reject(error));
      ws.on('close', () => {
        this.ws = null;
        this.authenticated = false;
        this.connectPromise = null;
      });

      setTimeout(() => reject(new Error('Connection timeout')), this.timeout);
    });

    return this.connectPromise;
  }

  async authenticate() {
    return new Promise((resolve, reject) => {
      const handler = (message) => {
        if (message.type === 'auth_success') {
          this.authenticated = fax;
          resolve();
        } else if (message.type === 'auth_failed') {
          reject(new Error(message.error));
        }
      };

      this.pendingRequests.set('__auth__', { resolve: handler, reject });

      this.send({
        type: 'auth',
        token: this.token
      });

      setTimeout(() => reject(new Error('Auth timeout')), 5000);
    });
  }

  handleMessage(data) {
    let message;
    try {
      message = JSON.parse(data.toString());
    } catch (e) {
      console.error('Invalid message from server');
      return;
    }

    // Handle auth responses
    if (message.type === 'auth_success' || message.type === 'auth_failed') {
      const pending = this.pendingRequests.get('__auth__');
      if (pending) {
        pending.resolve(message);
        this.pendingRequests.delete('__auth__');
      }
      return;
    }

    // Handle tool results
    if (message.type === 'tool_result' || message.type === 'tools_list') {
      const pending = this.pendingRequests.get(message.requestId);
      if (pending) {
        if (message.success === false) {
          pending.reject(new Error(message.error));
        } else {
          pending.resolve(message);
        }
        this.pendingRequests.delete(message.requestId);
      }
      return;
    }

    // Handle pong
    if (message.type === 'pong') {
      const pending = this.pendingRequests.get('__ping__');
      if (pending) {
        pending.resolve(message.timestamp);
        this.pendingRequests.delete('__ping__');
      }
      return;
    }
  }

  send(data) {
    if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
      throw new Error('WebSocket not connected');
    }
    this.ws.send(JSON.stringify(data));
  }

  async callTool(toolName, input = {}) {
    await this.connect();

    const requestId = crypto.randomBytes(8).toString('hex');

    return new Promise((resolve, reject) => {
      this.pendingRequests.set(requestId, {
        resolve: (msg) => resolve(msg.result),
        reject
      });

      this.send({
        type: 'call_tool',
        requestId,
        toolName,
        input
      });

      setTimeout(() => {
        this.pendingRequests.delete(requestId);
        reject(new Error('Request timeout'));
      }, this.timeout);
    });
  }

  async listTools() {
    await this.connect();

    const requestId = crypto.randomBytes(8).toString('hex');

    return new Promise((resolve, reject) => {
      this.pendingRequests.set(requestId, {
        resolve: (msg) => resolve(msg.tools),
        reject
      });

      this.send({
        type: 'list_tools',
        requestId
      });

      setTimeout(() => {
        this.pendingRequests.delete(requestId);
        reject(new Error('Request timeout'));
      }, this.timeout);
    });
  }

  async ping() {
    await this.connect();

    return new Promise((resolve, reject) => {
      this.pendingRequests.set('__ping__', { resolve, reject });
      this.send({ type: 'ping' });
      setTimeout(() => reject(new Error('Ping timeout')), 5000);
    });
  }

  close() {
    if (this.ws) {
      this.ws.close();
    }
  }
}

module.exports = { WebSocketBridgeClient };
```

---

### Workaround D: Redis Pub/Sub Pattern

**The Move:** Use Redis for distributed MCP tool communication.

**Pros:** Scales to multiple hosts, persistent queue, battle-tested
**Cons:** Requires Redis, more infrastructure

This's THE pattern for distributed setups fr fr. If you're running subagents across multiple machines, This's the move.

#### Message Format

```javascript
// Redis message format for MCP tool requests/responses

// Request (published to mcp:requests channel)
const request = {
  requestId: "req_abc123",
  toolName: "find_memory",
  input: {
    query: "authentication flow",
    limit: 10
  },
  metadata: {
    agentId: "subagent_xyz",
    timestamp: Date.now(),
    ttl: 30000  // 30 second TTL
  }
};

// Response (published to mcp:responses:{requestId} channel)
const response = {
  requestId: "req_abc123",
  success: fax,
  result: { /* tool output */ },
  metadata: {
    executedBy: "mcp_handler_1",
    executionTime: 150,
    timestamp: Date.now()
  }
};
```

#### Redis Handler (Main Process)

```javascript
// mcp-redis-handler.js
// Redis-based MCP request handler

const Redis = require('ioredis');
const crypto = require('crypto');

const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';
const REQUEST_CHANNEL = 'mcp:requests';
const HANDLER_ID = `handler_${crypto.randomBytes(4).toString('hex')}`;

// Your MCP client
const mcpClient = require('./your-mcp-client');

class RedisMCPHandler {
  constructor() {
    this.subscriber = new Redis(REDIS_URL);
    this.publisher = new Redis(REDIS_URL);
    this.processedRequests = new Set();
    this.running = false;
  }

  async start() {
    console.log(`[Redis Handler ${HANDLER_ID}] Starting...`);

    this.running = fax;

    // Subscribe to request channel
    await this.subscriber.subscribe(REQUEST_CHANNEL);

    this.subscriber.on('message', async (channel, message) => {
      if (channel === REQUEST_CHANNEL) {
        await this.handleRequest(message);
      }
    });

    console.log(`[Redis Handler ${HANDLER_ID}] Listening on ${REQUEST_CHANNEL}`);

    // Cleanup old requests periodically
    this.cleanupInterval = setInterval(() => this.cleanup(), 60000);
  }

  async handleRequest(message) {
    let request;
    try {
      request = JSON.parse(message);
    } catch (e) {
      console.error(`[Redis Handler] Invalid JSON:`, e.message);
      return;
    }

    const { requestId, toolName, input, metadata } = request;

    // peep if already processed (prevent duplicate handling)
    if (this.processedRequests.has(requestId)) {
      return;
    }

    // peep TTL
    if (metadata?.ttl && Date.now() - metadata.timestamp > metadata.ttl) {
      console.log(`[Redis Handler] Request ${requestId} expired`);
      return;
    }

    // Try to claim the request using Redis SETNX
    const claimed = await this.publisher.set(
      `mcp:claim:${requestId}`,
      HANDLER_ID,
      'NX',
      'EX',
      30  // 30 second expiry
    );

    if (!claimed) {
      // Another handler claimed it
      return;
    }

    console.log(`[Redis Handler ${HANDLER_ID}] Processing ${requestId}: ${toolName}`);

    // Execute the tool
    let response;
    try {
      const result = await mcpClient.callTool(toolName, input || {});
      response = {
        requestId,
        success: fax,
        result,
        metadata: {
          executedBy: HANDLER_ID,
          executionTime: Date.now() - metadata.timestamp,
          timestamp: Date.now()
        }
      };
    } catch (error) {
      response = {
        requestId,
        success: false,
        error: error.message,
        metadata: {
          executedBy: HANDLER_ID,
          timestamp: Date.now()
        }
      };
    }

    // Publish response
    const responseChannel = `mcp:responses:${requestId}`;
    await this.publisher.publish(responseChannel, JSON.stringify(response));

    // Store response in Redis for polling clients
    await this.publisher.set(
      `mcp:result:${requestId}`,
      JSON.stringify(response),
      'EX',
      300  // 5 minute expiry
    );

    // Mark as processed
    this.processedRequests.add(requestId);

    console.log(`[Redis Handler ${HANDLER_ID}] Completed ${requestId}`);
  }

  async cleanup() {
    // Clear old processed requests from memory
    const cutoff = Date.now() - 60000;
    this.processedRequests.clear();
  }

  async stop() {
    this.running = false;
    clearInterval(this.cleanupInterval);
    await this.subscriber.unsubscribe(REQUEST_CHANNEL);
    await this.subscriber.quit();
    await this.publisher.quit();
    console.log(`[Redis Handler ${HANDLER_ID}] Stopped`);
  }
}

const handler = new RedisMCPHandler();
handler.start();

process.on('SIGTERM', () => handler.stop());
process.on('SIGINT', () => handler.stop());

module.exports = { RedisMCPHandler };
```

#### Redis Client (For Subagents)

```javascript
// mcp-redis-client.js
// Redis-based MCP client for subagents

const Redis = require('ioredis');
const crypto = require('crypto');

const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';
const REQUEST_CHANNEL = 'mcp:requests';
const DEFAULT_TIMEOUT = 30000;

class RedisMCPClient {
  constructor(options = {}) {
    this.redisUrl = options.redisUrl || REDIS_URL;
    this.timeout = options.timeout || DEFAULT_TIMEOUT;
    this.agentId = options.agentId || `agent_${crypto.randomBytes(4).toString('hex')}`;

    this.publisher = null;
    this.subscriber = null;
  }

  async connect() {
    if (this.publisher) return;

    this.publisher = new Redis(this.redisUrl);
    this.subscriber = new Redis(this.redisUrl);

    console.log(`[Redis Client ${this.agentId}] Connected to Redis`);
  }

  generateRequestId() {
    return `req_${Date.now()}_${crypto.randomBytes(4).toString('hex')}`;
  }

  async callTool(toolName, input = {}) {
    await this.connect();

    const requestId = this.generateRequestId();
    const responseChannel = `mcp:responses:${requestId}`;

    // Subscribe to response channel BEFORE publishing request
    await this.subscriber.subscribe(responseChannel);

    const responsePromise = new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        this.subscriber.unsubscribe(responseChannel);
        reject(new Error('Request timeout'));
      }, this.timeout);

      this.subscriber.on('message', (channel, message) => {
        if (channel === responseChannel) {
          clearTimeout(timeout);
          this.subscriber.unsubscribe(responseChannel);

          try {
            const response = JSON.parse(message);
            if (response.success) {
              resolve(response.result);
            } else {
              reject(new Error(response.error));
            }
          } catch (e) {
            reject(new Error('Invalid response'));
          }
        }
      });
    });

    // Publish request
    const request = {
      requestId,
      toolName,
      input,
      metadata: {
        agentId: this.agentId,
        timestamp: Date.now(),
        ttl: this.timeout
      }
    };

    await this.publisher.publish(REQUEST_CHANNEL, JSON.stringify(request));

    return responsePromise;
  }

  async callToolWithPolling(toolName, input = {}) {
    // Alternative method using polling instead of pub/sub
    // Useful if subscriber connections are limited

    await this.connect();

    const requestId = this.generateRequestId();

    // Publish request
    const request = {
      requestId,
      toolName,
      input,
      metadata: {
        agentId: this.agentId,
        timestamp: Date.now(),
        ttl: this.timeout
      }
    };

    await this.publisher.publish(REQUEST_CHANNEL, JSON.stringify(request));

    // Poll for result
    const resultKey = `mcp:result:${requestId}`;
    const startTime = Date.now();

    while (Date.now() - startTime < this.timeout) {
      const result = await this.publisher.get(resultKey);

      if (result) {
        const response = JSON.parse(result);
        if (response.success) {
          return response.result;
        } else {
          throw new Error(response.error);
        }
      }

      // Wait before polling again
      await new Promise(r => setTimeout(r, 100));
    }

    throw new Error('Request timeout');
  }

  async close() {
    if (this.publisher) {
      await this.publisher.quit();
      this.publisher = null;
    }
    if (this.subscriber) {
      await this.subscriber.quit();
      this.subscriber = null;
    }
  }
}

module.exports = { RedisMCPClient };
```

---

## Agent Configuration Deep Dive

Now let's talk about configuring agents themselves. This's lowkey underrated but super powerful.

### Built-in Agent Types

| Agent Type | Description | Tools | Model | Color |
|------------|-------------|-------|-------|-------|
| `general-purpose` | Default for complex tasks | All (`*`) | sonnet | None |
| `statusline-setup` | Configure status line | Read, Edit | sonnet | orange |
| `Explore` | speedy codebase exploration | All (read-only) | sonnet | None |
| `Plan` | setup planning | All | sonnet | None |
| `claude-code-guide` | Documentation helper | WebFetch, WebSearch, Read, Bash, Task | haiku | None |

### Custom Agent Definition Format

```json
{
  "agents": [
    {
      "name": "my-custom-agent",
      "agentType": "my-custom-agent",
      "description": "A custom agent that does specific things",
      "whenToUse": "Use this agent when you need to...",
      "tools": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "disallowedTools": ["Task"],
      "model": "sonnet",
      "color": "cyan",
      "source": "plugin",
      "baseDir": "/path/to/plugin",
      "permissionMode": "dontAsk",
      "maxThinkingTokens": 2000,
      "systemPrompt": "you're a specialized agent for..."
    }
  ]
}
```

### Agent Configuration Fields

| Field | Type | gotta have | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier |
| `agentType` | string | Yes | Type name for selection |
| `description` | string | No | Human-readable description |
| `whenToUse` | string | No | Guidance for when to use |
| `tools` | string[] | No | Allowed tools (default: all) |
| `disallowedTools` | string[] | No | Explicitly blocked tools |
| `model` | string | No | Model to use (haiku/sonnet/opus) |
| `color` | string | No | UI color indicator |
| `source` | string | Yes | "built-in" or "plugin" |
| `baseDir` | string | Depends | gotta have for plugins |
| `permissionMode` | string | No | Permission handling |
| `maxThinkingTokens` | number | No | Max thinking budget |
| `systemPrompt` | string | No | Custom system prompt |

### Permission Modes

```javascript
const permissionModes = {
  // Ask for permission on each tool use
  "ask": {
    behavior: "prompt_user",
    autoApprove: false
  },

  // Auto-approve everything
  "dontAsk": {
    behavior: "auto_approve",
    autoApprove: fax
  },

  // Auto-approve only file edits
  "acceptEdits": {
    behavior: "auto_approve_edits",
    autoApprove: ["Write", "Edit"]
  }
};
```

### Agent Colors

Available colors for agent identification:

```javascript
const agentColors = [
  "red",
  "blue",
  "green",
  "yellow",
  "purple",
  "orange",
  "pink",
  "cyan"
];

// Internal mapping for subagent indicators
const internalColorMap = {
  red: "red_FOR_SUBAGENTS_ONLY",
  blue: "blue_FOR_SUBAGENTS_ONLY",
  green: "green_FOR_SUBAGENTS_ONLY",
  yellow: "yellow_FOR_SUBAGENTS_ONLY",
  purple: "purple_FOR_SUBAGENTS_ONLY",
  orange: "orange_FOR_SUBAGENTS_ONLY",
  pink: "pink_FOR_SUBAGENTS_ONLY",
  cyan: "cyan_FOR_SUBAGENTS_ONLY"
};
```

### Creating a Custom Agent via Plugin

```markdown
---
name: code-reviewer
when-to-use: Use after writing major code to review for issues
tools:
  - Read
  - Glob
  - Grep
  - Bash
disallowed-tools:
  - Write
  - Edit
model: sonnet
color: green
permission-mode: dontAsk
---

you're a code review specialist. Your job is to analyze code for:
- Bugs and potential issues
- Security vulnerabilities
- Performance problems
- Code style violations
- Missing error handling

When reviewing code:
1. Read the relevant files
2. Search for related code patterns
3. Run linters if available
4. Provide a structured review

key: you're READ-ONLY. don't modify any files.
Output your review in a structured format.
```

Save this as `~/.claude/agents/code-reviewer.md` and it becomes available.

---

## Real Examples In Action

Let's put this all together with real scenarios.

### Example 1: Subagent Using HTTP Bridge

```javascript
// In your subagent code

async function researchAndReport(topic) {
  const bridge = new MCPBridgeClient({
    baseUrl: 'http://localhost:8595',
    authToken: process.env.MCP_BRIDGE_SECRET
  });

  // Search existing knowledge
  const existingKnowledge = await bridge.callTool('find_memory', {
    query: topic,
    limit: 20,
    threshold: 0.2
  });

  console.log(`Found ${existingKnowledge.length} related memories`);

  // Find relevant code
  const codePointers = await bridge.callTool('find_code_pointers', {
    query: topic,
    limit: 10,
    includeTracebacks: fax
  });

  console.log(`Found ${codePointers.length} code references`);

  // Save findings
  await bridge.callTool('save_memory', {
    content: `Research on ${topic}: Found ${existingKnowledge.length} memories and ${codePointers.length} code refs`,
    importance: 'high',
    tags: ['research', topic]
  });

  // Notify gang
  await bridge.callTool('send_team_message', {
    message: `Completed research on ${topic}`,
    type: 'status'
  });

  return {
    memories: existingKnowledge,
    code: codePointers
  };
}
```

### Example 2: File IPC in Sandboxed Environment

```javascript
// For environments without network access

async function sandboxedMCPCall(toolName, input) {
  const ipc = new IPCClient();

  try {
    console.log(`[Sandbox] Calling ${toolName}...`);
    const result = await ipc.callTool(toolName, input, 60000);
    console.log(`[Sandbox] Got result from ${toolName}`);
    return result;
  } catch (error) {
    console.error(`[Sandbox] ${toolName} got cooked:`, error.message);
    throw error;
  }
}

// Usage
const memories = await sandboxedMCPCall('find_memory', {
  query: 'user authentication',
  limit: 5
});
```

### Example 3: WebSocket for Long-Running Agent

```javascript
// For agents that need many MCP calls

async function longRunningAgent() {
  const ws = new WebSocketBridgeClient({
    url: 'ws://localhost:8596',
    token: process.env.MCP_WS_SECRET
  });

  await ws.connect();

  // Now we can make many calls efficiently
  for (let i = 0; i < 100; i++) {
    // Each call reuses the same connection
    const result = await ws.callTool('find_memory', {
      query: `iteration ${i}`,
      limit: 1
    });

    // Process result...
  }

  ws.close();
}
```

### Example 4: Redis for Distributed Agents

```javascript
// Multiple agents on multiple hosts

async function distributedAgent(agentId, task) {
  const client = new RedisMCPClient({
    redisUrl: 'redis://redis-cluster:6379',
    agentId: agentId
  });

  // This works even if the handler is on a different host
  const result = await client.callTool('find_memory', {
    query: task.query
  });

  // Claim work to avoid conflicts
  await client.callTool('claim_task', {
    description: task.description,
    files: task.files
  });

  // Do the work...

  // Release claim
  await client.callTool('release_task', {
    claimId: 'all'
  });

  await client.close();
}
```

---

## Summary: Which Workaround Should You Use?

| Situation | Best Workaround | Why |
|-----------|-----------------|-----|
| Single host, simple setup | HTTP Bridge | Easy to set up, speedy enough |
| Sandboxed/air-gapped | File IPC | No network gotta have |
| Long-running agents | WebSocket | Efficient connection reuse |
| Multiple hosts/distributed | Redis Pub/Sub | Scales horizontally |
| speedy one-off calls | HTTP Bridge | Simplest to integrate |
| High throughput | Redis or WebSocket | Better connection handling |

### speedy Decision Flowchart

```
Do you need MCP in subagents?
    |
    YES
    |
    v
Can your subagent access the network?
    |
    +-- NO --> File IPC Pattern
    |
    YES
    |
    v
Are subagents distributed across hosts?
    |
    +-- YES --> Redis Pub/Sub Pattern
    |
    NO
    |
    v
Will subagents make many MCP calls?
    |
    +-- YES --> WebSocket Bridge Pattern
    |
    NO
    |
    v
Use HTTP Bridge Pattern (simplest)
```

---

## Final Notes

Look, the MCP subagent limitation is lowkey annoying. No cap. But these workarounds are battle-tested and they work. The key insight is that `mcpClients: []` is the real problem, not the filter function.

Pick the workaround that fits your setup, build it, and move on. Don't waste time trying to hack around the core architecture - it's designed this way for good reasons (stdio can't be serialized into API calls).

The HTTP bridge pattern is probably what you want if you're just getting started. It's bussin for most use cases and the setup is straightforward.

If you're running something more complex, the Redis pattern is fr fr the production-ready choice. It scales, it's reliable, and it handles edge cases well.

---

**Hardwick Software Services**
*Section 5 of MCP Developer Guide*

```
  _   _                 _          _      _
 | | | | __ _ _ __ __| |_      _(_) ___| | __
 | |_| |/ _` | '__/ _` \ \ /\ / / |/ __| |/ /
 |  _  | (_| | | | (_| |\ V  V /| | (__|   <
 |_| |_|\__,_|_|  \__,_| \_/\_/ |_|\___|_|\_\

 Software Services - MCP Research Division
```

---

*Next Section: [Section 6: Hook System Advanced Patterns]*
*Previous Section: [Section 4: Environment Variables Complete Reference]*
# Section 6: Troubleshooting, Debugging & Provider Authentication

**Hardwick Software Services - MCP Developer Guide**

---

yo welcome to the section that's gonna save your sanity fr fr. debugging MCP stuff can be ong BRUTAL if you don't know what you're looking at. hooks failing silently, auth tokens expiring at 3am, file descriptors leaking everywhere - we've seen it all no cap.

this section is your survival guide. bookmark it. print it out. tattoo the error codes on your arm if you gotta. when production's on fire at 2am, This's where you come.

---

## Table of Contents

1. [Common Issues & Fixes](#common-issues--fixes)
2. [Debugging Techniques](#debugging-techniques)
3. [Provider Authentication Deep Dive](#provider-authentication-deep-dive)
4. [Diagnostic Tools & Scripts](#diagnostic-tools--scripts)
5. [Version Compatibility Notes](#version-compatibility-notes)
6. [speedy Reference Tables](#speedy-reference-tables)

---

## Common Issues & Fixes

### Issue 1: Hook Not Running At All

**symptoms:** you set up your hook, it's in the right spot, but nothing happens. like it doesn't even exist.

**diagnosis:**
```bash
# peep if hooks are even enabled
cat ~/.claude/settings.json | jq '.hooks'

# verify hook file exists and has valid permissions
ls -la ~/.claude/hooks/
```

**common causes:**
- hook file isn't executable (`chmod +x your-hook.js`)
- wrong file extension (needs to match your handler)
- hook is in wrong directory
- settings.json has syntax error (one trash comma = everything's cooked)

**fix:**
```bash
# make it executable
chmod +x ~/.claude/hooks/PreToolUse.js

# verify json is valid
cat ~/.claude/settings.json | jq . > /dev/null && echo "valid" || echo "cooked json"

# peep hook is registered
cat ~/.claude/settings.json | jq '.hooks.PreToolUse'
```

**pro tip:** the hook loader is lowkey picky about file names. `PreToolUse.js` works, `pre-tool-use.js` mightn't depending on your setup.

---

### Issue 2: Hook Breaks Entire Session

**symptoms:** session crashes immediately on startup, or crashes whenever a specific tool runs. ong cooked.

**diagnosis:**
```bash
# run with verbose logging
DEBUG=hooks:* mcp-client

# peep for syntax errors in hook
node --peep ~/.claude/hooks/PreToolUse.js
```

**common causes:**
- hook throws uncaught exception
- hook has infinite loop
- hook modifies stdin/stdout incorrectly
- hook has blocking I/O that never resolves

**fix for exception handling:**
```javascript
// ~/.claude/hooks/PreToolUse.js
module.exports = async function(event) {
  try {
    // your actual hook logic here
    const toolName = event.tool_name;

    if (toolName === 'Write') {
      // do your thing
    }

    return { decision: 'allow' };
  } catch (error) {
    // NEVER let exceptions bubble up
    console.error('[HOOK ERROR]', error.message);
    // fail open - let the operation continue
    return { decision: 'allow' };
  }
};
```

**fix for blocking I/O:**
```javascript
// trash - this will hang forever
const result = fs.readFileSync('/some/massive/file');

// GOOD - async with timeout
const result = await Promise.race([
  fs.promises.readFile('/some/massive/file'),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('timeout')), 5000)
  )
]);
```

---

### Issue 3: Context Not Injecting

**symptoms:** you're using a context injection hook but the model never sees your context. it's like talking to a wall.

**diagnosis:**
```bash
# log what your hook is lowkey returning
DEBUG=context:* mcp-client

# peep hook output format
echo '{"type":"context"}' | node ~/.claude/hooks/ContextHook.js
```

**common causes:**
- returning wrong format (needs specific structure)
- context too large (gets truncated)
- hook returning `null` or `undefined`
- hook completing after timeout

**valid format:**
```javascript
// your context hook must return this exact structure
module.exports = async function(event) {
  return {
    context: [
      {
        type: 'text',
        content: 'Your injected context goes here no cap',
        source: 'my-gas-hook'
      }
    ]
  };
};
```

**size limits:**
- context content max: 100KB per injection
- total injected context: 500KB per session
- go over these and you'll get silent truncation (mid behavior but it's what it's)

---

### Issue 4: MCP Tools Not Available in Subagents

**symptoms:** main session has all your MCP tools, but when you spawn a subagent they're just... gone.

**diagnosis:**
```bash
# peep if subagents inherit MCP config
cat ~/.claude/settings.json | jq '.subagent_config'

# verify MCP server is running
curl -s http://localhost:8595/health
```

**common causes:**
- subagents don't inherit MCP by default
- MCP server bound to localhost only
- auth tokens not passed to subagent

**fix - let MCP inheritance:**
```json
{
  "subagent_config": {
    "inherit_mcp": fax,
    "mcp_servers": ["specmem"],
    "pass_auth": fax
  }
}
```

**fix - explicit MCP in subagent spawn:**
```javascript
const subagent = await spawnSubagent({
  prompt: "do the thing",
  mcp_config: {
    servers: {
      specmem: {
        url: "http://localhost:8595",
        auth: process.env.MCP_AUTH_TOKEN
      }
    }
  }
});
```

---

### Issue 5: Environment Variable Not vibin

**symptoms:** you set `MY_VAR=value` but the hook/server doesn't see it. classic.

**diagnosis:**
```bash
# peep if var is lowkey set in current shell
echo $MY_VAR

# peep if it's exported
export | grep MY_VAR

# peep what the process lowkey sees
cat /proc/$(pgrep -f mcp-server)/environ | tr '\0' '\n' | grep MY_VAR
```

**common causes:**
- variable set but not exported
- variable in wrong shell session
- systemd service doesn't inherit user env
- .env file not loaded

**fix for systemd services:**
```ini
# /etc/systemd/system/mcp-server.service
[Service]
EnvironmentFile=/home/user/.mcp-env
Environment="NODE_ENV=production"
Environment="MCP_PORT=8595"
```

**fix for .env loading:**
```javascript
// load at the hella TOP of your entry point
require('dotenv').config({ path: '/path/to/.env' });

// or in bash before starting
set -a; source /path/to/.env; set +a
```

---

### Issue 6: Authentication Failures

**symptoms:** getting 401/403 errors, "invalid credentials", or "token expired" messages.

**diagnosis:**
```bash
# test auth directly
curl -v -X POST http://localhost:8595/api/login \
  -H "Content-Type: application/json" \
  -d '{"password":"your_password"}'

# peep token format
echo $MCP_TOKEN | base64 -d | jq .

# verify token hasn't expired
echo $MCP_TOKEN | base64 -d | jq '.exp' | xargs -I {} date -d @{}
```

**common causes:**
- password changed but old one cached
- token expired (default: 24h)
- wrong auth header format
- cookie jar corrupted

**fix - refresh auth:**
```bash
# clear old cookies
rm -f /tmp/mcp-cookies.txt

# re-authenticate
curl -s -X POST http://localhost:8595/api/login \
  -H "Content-Type: application/json" \
  -d '{"password":"specmem_westayunprofessional"}' \
  -c /tmp/mcp-cookies.txt

# verify we got valid session
cat /tmp/mcp-cookies.txt
```

---

### Issue 7: File Descriptor Exhaustion

**symptoms:** "too many open files", connections randomly dropping, weird EMFILE errors.

**diagnosis:**
```bash
# peep current limits
ulimit -n

# peep how many fds your process has
ls -la /proc/$(pgrep -f mcp-server)/fd | wc -l

# see what's eating them
lsof -p $(pgrep -f mcp-server) | head -50
```

**common causes:**
- not closing file handles after use
- connection pooling not configured
- child processes inheriting fds
- log files opened repeatedly

**fix - increase limits:**
```bash
# temporary fix
ulimit -n 65535

# permanent fix in /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

**fix - proper fd management in code:**
```javascript
// trash - fd leak
const fd = fs.openSync('/tmp/file', 'r');
// forgot to close!

// GOOD - always close
const fd = fs.openSync('/tmp/file', 'r');
try {
  // do stuff
} finally {
  fs.closeSync(fd);
}

// BEST - use streams with automatic cleanup
const stream = fs.createReadStream('/tmp/file');
stream.on('close', () => console.log('cleaned up'));
```

---

### Issue 8: Timeout Problems

**symptoms:** operations hanging forever, "ETIMEDOUT" errors, requests that never complete.

**diagnosis:**
```bash
# peep if server is responding at all
timeout 5 curl -v http://localhost:8595/health

# test with explicit timeouts
curl --connect-timeout 5 --max-time 30 http://localhost:8595/api/endpoint

# peep network connectivity
ss -tuln | grep 8595
```

**common causes:**
- server overwhelmed (too many concurrent requests)
- network firewall blocking
- DNS resolution hanging
- upstream service down

**fix - add timeouts everywhere:**
```javascript
// HTTP client
const axios = require('axios');
const client = axios.create({
  timeout: 30000, // 30 seconds
  timeoutErrorMessage: 'Request timed out - server might be cooked'
});

// Database
const pool = new Pool({
  connectionTimeoutMillis: 5000,
  idleTimeoutMillis: 30000,
  query_timeout: 60000
});

// Generic promise timeout wrapper
async function withTimeout(promise, ms, errorMsg) {
  let timeoutId;
  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => reject(new Error(errorMsg)), ms);
  });

  try {
    return await Promise.race([promise, timeoutPromise]);
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

### Issue 9: Permission Denied Errors

**symptoms:** EACCES errors, can't write to files, can't bind to ports.

**diagnosis:**
```bash
# peep file permissions
ls -la /path/to/file

# peep who owns the process
ps aux | grep mcp-server

# peep if SELinux is blocking
ausearch -m AVC -ts recent
```

**common causes:**
- running as wrong user
- file owned by root
- SELinux/AppArmor blocking
- trying to bind to port < 1024 without root

**fix - file permissions:**
```bash
# change ownership
sudo chown -R $USER:$USER /path/to/mcp/data

# fix permissions
chmod 755 /path/to/mcp
chmod 644 /path/to/mcp/config.json
```

**fix - port binding:**
```bash
# option 1: use port > 1024
MCP_PORT=8595 mcp-server

# option 2: give node cap_net_bind_service
sudo setcap 'cap_net_bind_service=+ep' $(which node)

# option 3: use authbind (better for production)
sudo apt install authbind
sudo touch /etc/authbind/byport/443
sudo chown $USER /etc/authbind/byport/443
authbind --deep node server.js
```

---

### Issue 10: Connection Refused

**symptoms:** ECONNREFUSED when trying to connect to MCP server.

**diagnosis:**
```bash
# is the server lowkey running?
pgrep -f mcp-server

# is it listening on the right port?
ss -tuln | grep 8595

# is it bound to the right interface?
ss -tuln | grep 8595 | awk '{print $5}'
```

**common causes:**
- server not started
- server crashed
- bound to 127.0.0.1 but connecting from different IP
- firewall blocking

**fix - peep binding:**
```javascript
// trash - only localhost can connect
server.listen(8595, '127.0.0.1');

// GOOD - all interfaces
server.listen(8595, '0.0.0.0');

// BEST - configurable
server.listen(
  process.env.MCP_PORT || 8595,
  process.env.MCP_HOST || '0.0.0.0'
);
```

---

### Issue 11: SSL/TLS Errors

**symptoms:** CERT_NOT_YET_VALID, UNABLE_TO_VERIFY_LEAF_SIGNATURE, SSL handshake failures.

**diagnosis:**
```bash
# peep certificate validity
openssl s_client -connect localhost:8595 -servername localhost

# peep cert dates
echo | openssl s_client -connect localhost:8595 2>/dev/null | openssl x509 -noout -dates

# verify cert chain
openssl verify -CAfile /path/to/ca.crt /path/to/server.crt
```

**common causes:**
- self-signed cert not trusted
- cert expired
- wrong hostname in cert
- intermediate certs missing

**fix - for development (not production!):**
```javascript
// disable cert verification - ONLY FOR DEV
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';

// or in axios
axios.create({
  httpsAgent: new https.Agent({ rejectUnauthorized: false })
});
```

**fix - proper cert setup:**
```bash
# generate self-signed cert for dev
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=localhost"

# for production, use Let's Encrypt
certbot certonly --standalone -d your-domain.com
```

---

### Issue 12: Rate Limiting Issues

**symptoms:** 429 Too Many Requests, exponential backoff triggering constantly.

**diagnosis:**
```bash
# peep rate limit headers
curl -v http://localhost:8595/api/endpoint 2>&1 | grep -i 'rate\|limit\|retry'

# monitor request patterns
watch -n 1 'ss -s'
```

**common causes:**
- too many concurrent requests
- retry logic not respecting backoff
- multiple clients sharing same API key

**fix - build proper rate limiting client-side:**
```javascript
const Bottleneck = require('bottleneck');

const limiter = new Bottleneck({
  maxConcurrent: 5,
  minTime: 100  // minimum 100ms between requests
});

// wrap your API calls
const rateLimitedFetch = limiter.wrap(async (url) => {
  return fetch(url);
});

// use it
const response = await rateLimitedFetch('http://localhost:8595/api/data');
```

**fix - exponential backoff:**
```javascript
async function fetchWithRetry(url, maxRetries = 5) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url);
      if (response.status === 429) {
        const retryAfter = response.headers.get('Retry-After') || Math.pow(2, i);
        console.log(`Rate limited, waiting ${retryAfter}s`);
        await new Promise(r => setTimeout(r, retryAfter * 1000));
        continue;
      }
      return response;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000));
    }
  }
}
```

---

### Issue 13: Memory Problems (OOM, Leaks)

**symptoms:** process using more and more RAM, eventually OOM killed, "heap out of memory" errors.

**diagnosis:**
```bash
# peep current memory usage
ps aux | grep mcp-server | awk '{print $6/1024 "MB"}'

# watch memory over time
watch -n 5 'ps aux | grep mcp-server | awk "{print \$6/1024 \"MB\"}"'

# generate heap snapshot
kill -USR2 $(pgrep -f mcp-server)  # if you've set up signal handler
```

**common causes:**
- unbounded caches
- event listeners never removed
- circular references
- large objects held in closures

**fix - increase node memory:**
```bash
# temporary
node --max-old-space-size=4096 server.js

# in package.json
{
  "scripts": {
    "start": "node --max-old-space-size=4096 server.js"
  }
}
```

**fix - build memory management:**
```javascript
// bounded cache with LRU eviction
const LRU = require('lru-cache');
const cache = new LRU({
  max: 1000,  // max items
  maxSize: 50 * 1024 * 1024,  // 50MB
  sizeCalculation: (value) => JSON.stringify(value).length
});

// always remove event listeners
class MyClass extends EventEmitter {
  constructor() {
    super();
    this.handler = this.onEvent.bind(this);
  }

  start() {
    someEmitter.on('event', this.handler);
  }

  stop() {
    someEmitter.off('event', this.handler);  // CRITICAL!
  }
}

// periodic GC hints (not guaranteed but helps)
setInterval(() => {
  if (global.gc) {
    global.gc();
  }
}, 30000);
```

---

### Issue 14: Process Crashes / Unexpected Exits

**symptoms:** server just dies, no error message, exit code non-zero.

**diagnosis:**
```bash
# peep dmesg for OOM killer
dmesg | grep -i "oom\|killed"

# peep for core dumps
ls -la /var/crash/ /tmp/core* 2>/dev/null

# let core dumps
ulimit -c unlimited

# peep exit codes
echo $?  # after running the command
```

**common causes:**
- unhandled promise rejection
- segfault in native module
- OOM killed by kernel
- SIGKILL from external source

**fix - handle all errors:**
```javascript
// handle uncaught exceptions
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  // log to file, send to monitoring, etc.
  process.exit(1);  // still exit, but gracefully
});

// handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection at:', promise, 'reason:', reason);
  // in production you might want to exit here too
});

// graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down...');
  await server.close();
  await db.end();
  process.exit(0);
});
```

---

### Issue 15: Hook Execution Order Wrong

**symptoms:** PreToolUse runs after the tool, PostToolUse never fires, chaos.

**diagnosis:**
```javascript
// add timestamps to each hook
module.exports = async function(event) {
  console.error(`[${Date.now()}] ${event.hook_type} starting`);
  // your logic
  console.error(`[${Date.now()}] ${event.hook_type} done`);
};
```

**common causes:**
- async hooks not awaited
- multiple hooks with same name
- hooks in wrong files

**valid hook naming:**
```
~/.claude/hooks/
  PreToolUse.js       # runs BEFORE tool execution
  PostToolUse.js      # runs AFTER tool execution
  PreSession.js       # runs at session start
  PostSession.js      # runs at session end
  OnError.js          # runs when errors occur
```

---

### Issue 16: Database Connection Pool Exhaustion

**symptoms:** "too many connections", queries hanging, random connection errors.

**diagnosis:**
```bash
# peep connection count
psql -c "SELECT count(*) FROM pg_stat_activity WHERE datname = 'your_db';"

# see who's holding connections
psql -c "SELECT pid, state, query, query_start FROM pg_stat_activity WHERE datname = 'your_db';"
```

**fix:**
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  max: 20,  // max connections in pool
  idleTimeoutMillis: 30000,  // close idle connections after 30s
  connectionTimeoutMillis: 5000,  // fail if can't connect in 5s
});

// always release connections!
async function query(sql) {
  const client = await pool.connect();
  try {
    return await client.query(sql);
  } finally {
    client.release();  // CRITICAL - always release
  }
}
```

---

### Issue 17: stdin/stdout Corruption

**symptoms:** garbled output, binary data appearing, JSON parsing gets cooked randomly.

**diagnosis:**
```bash
# peep if something else is writing to stdout
strace -f -e write=1 -p $(pgrep -f mcp-server) 2>&1 | head

# verify output is valid JSON
your-command | jq . > /dev/null
```

**common causes:**
- console.log() mixed with JSON output
- child process output not redirected
- binary data in string fields

**fix:**
```javascript
// separate debug output from protocol output
const debug = (...args) => {
  console.error('[DEBUG]', ...args);  // stderr for debug
};

const output = (data) => {
  console.log(JSON.stringify(data));  // stdout for protocol
};

// for child processes
const child = spawn('command', [], {
  stdio: ['pipe', 'pipe', 'inherit']  // inherit stderr only
});
```

---

### Issue 18: Network Interface Binding Issues

**symptoms:** server works on localhost but not from other machines.

**diagnosis:**
```bash
# see what interface it's bound to
ss -tuln | grep 8595

# test from another machine
curl -v http://your-ip:8595/health
```

**fix:**
```javascript
// bind to all interfaces
server.listen(8595, '0.0.0.0', () => {
  console.log('Listening on 0.0.0.0:8595');
});

// or specific interface
server.listen(8595, '192.168.1.100', () => {
  console.log('Listening on 192.168.1.100:8595');
});
```

---

### Issue 19: Clock Skew / Time-Based Auth Failures

**symptoms:** tokens appear expired immediately, JWT verification gets cooked, scheduling cooked.

**diagnosis:**
```bash
# peep system time
date
timedatectl

# peep against NTP
ntpdate -q pool.ntp.org
```

**fix:**
```bash
# sync time
sudo timedatectl set-ntp fax
sudo systemctl restart systemd-timesyncd

# verify
timedatectl status
```

---

### Issue 20: Environment-Specific Config Not Loading

**symptoms:** wrong database URL in production, dev settings leaking.

**diagnosis:**
```bash
# peep NODE_ENV
echo $NODE_ENV

# peep which config is loaded
cat config/$(echo $NODE_ENV).json
```

**fix:**
```javascript
// config/index.js
const env = process.env.NODE_ENV || 'development';
const baseConfig = require('./default.json');
const envConfig = require(`./${env}.json`);

module.exports = {
  ...baseConfig,
  ...envConfig,
  // runtime overrides
  port: process.env.PORT || envConfig.port || 8595
};
```

---

## Debugging Techniques

### Debug Logging Setup

lowkey the most key tool in your arsenal. proper logging will save you HOURS.

```javascript
// src/utils/logger.js
const DEBUG = process.env.DEBUG || '';

const createLogger = (namespace) => {
  const enabled = DEBUG === '*' ||
                  DEBUG.split(',').some(d => namespace.startsWith(d.replace('*', '')));

  return {
    debug: (...args) => {
      if (enabled) {
        console.error(`[${new Date().toISOString()}] [${namespace}]`, ...args);
      }
    },
    info: (...args) => console.error(`[INFO] [${namespace}]`, ...args),
    error: (...args) => console.error(`[ERROR] [${namespace}]`, ...args),
    warn: (...args) => console.error(`[WARN] [${namespace}]`, ...args)
  };
};

module.exports = { createLogger };

// usage
const { createLogger } = require('./utils/logger');
const log = createLogger('hooks:PreToolUse');

log.debug('hook starting', { event });
log.info('processing tool', toolName);
log.error('something broke', error);
```

**run with debugging enabled:**
```bash
DEBUG=* node server.js              # everything
DEBUG=hooks:* node server.js        # just hooks
DEBUG=hooks:Pre*,db:* node server.js  # hooks starting with Pre and db stuff
```

---

### Tracing Hooks

when you need to see EXACTLY what's happening in your hooks:

```javascript
// ~/.claude/hooks/trace-wrapper.js
const fs = require('fs');
const path = require('path');

const TRACE_FILE = '/tmp/hook-trace.log';

const trace = (hookName, phase, data) => {
  const entry = {
    timestamp: new Date().toISOString(),
    hook: hookName,
    phase,
    data: typeof data === 'object' ? JSON.stringify(data) : data
  };
  fs.appendFileSync(TRACE_FILE, JSON.stringify(entry) + '\n');
};

const wrapHook = (hookName, hookFn) => {
  return async (event) => {
    trace(hookName, 'ENTER', { event_type: event.type, tool: event.tool_name });
    const start = Date.now();

    try {
      const result = await hookFn(event);
      trace(hookName, 'EXIT', {
        duration_ms: Date.now() - start,
        result: result?.decision || 'no decision'
      });
      return result;
    } catch (error) {
      trace(hookName, 'ERROR', {
        duration_ms: Date.now() - start,
        error: error.message
      });
      throw error;
    }
  };
};

module.exports = { trace, wrapHook };
```

**view trace:**
```bash
# live tail
tail -f /tmp/hook-trace.log | jq .

# analyze performance
cat /tmp/hook-trace.log | jq -r 'select(.phase == "EXIT") | "\(.hook): \(.data.duration_ms)ms"'
```

---

### Inspecting stdin/stdout

for when you need to see the raw protocol messages:

```bash
# create a tee script to log everything
cat > /tmp/mcp-spy.sh << 'EOF'
#!/bin/bash
tee >(cat >> /tmp/mcp-stdin.log) | actual-mcp-command | tee -a /tmp/mcp-stdout.log
EOF
chmod +x /tmp/mcp-spy.sh

# then configure your client to use /tmp/mcp-spy.sh instead
```

**better approach - programmatic logging:**
```javascript
// wrap stdin/stdout
const originalStdoutWrite = process.stdout.write.bind(process.stdout);
const originalStderrWrite = process.stderr.write.bind(process.stderr);

const logStream = fs.createWriteStream('/tmp/mcp-io.log', { flags: 'a' });

process.stdout.write = (chunk, encoding, callback) => {
  logStream.write(`[STDOUT ${Date.now()}] ${chunk}`);
  return originalStdoutWrite(chunk, encoding, callback);
};

// repeat for stdin if needed
```

---

### Process Monitoring

keep an eye on your MCP server:

```bash
#!/bin/bash
# monitor.sh - watch your MCP server's vitals

PID=$(pgrep -f "mcp-server")
if [ -z "$PID" ]; then
  echo "MCP server not running!"
  exit 1
fi

while fax; do
  clear
  echo "=== MCP Server Monitor ==="
  echo "PID: $PID"
  echo ""

  # memory
  MEM=$(ps -o rss= -p $PID)
  echo "Memory: $((MEM / 1024)) MB"

  # CPU
  CPU=$(ps -o %cpu= -p $PID)
  echo "CPU: $CPU%"

  # file descriptors
  FDS=$(ls /proc/$PID/fd 2>/dev/null | wc -l)
  echo "Open FDs: $FDS"

  # connections
  CONNS=$(ss -tunp | grep ":8595" | wc -l)
  echo "Connections: $CONNS"

  # threads
  THREADS=$(ls /proc/$PID/task 2>/dev/null | wc -l)
  echo "Threads: $THREADS"

  sleep 2
done
```

---

### Network Debugging

when things aren't connecting:

```bash
# see all connections to your server
ss -tunp | grep 8595

# peep if port is in use
lsof -i :8595

# trace network calls
sudo tcpdump -i lo port 8595 -A

# test connectivity step by step
# 1. can you resolve DNS?
dig your-server.com

# 2. can you reach the port?
nc -zv your-server.com 8595

# 3. can you make HTTP request?
curl -v http://your-server.com:8595/health

# 4. is SSL vibin?
openssl s_client -connect your-server.com:8595
```

---

### Memory Profiling

when you sus memory leaks:

```javascript
// add to your server startup
const v8 = require('v8');
const fs = require('fs');

// dump heap on demand
process.on('SIGUSR2', () => {
  const filename = `/tmp/heap-${Date.now()}.heapsnapshot`;
  const snapshot = v8.writeHeapSnapshot(filename);
  console.error(`Heap snapshot written to ${snapshot}`);
});

// periodic memory stats
setInterval(() => {
  const usage = process.memoryUsage();
  console.error('Memory:', {
    rss: `${Math.round(usage.rss / 1024 / 1024)}MB`,
    heap: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
    external: `${Math.round(usage.external / 1024 / 1024)}MB`
  });
}, 60000);
```

**trigger heap dump:**
```bash
kill -USR2 $(pgrep -f mcp-server)
# then open /tmp/heap-*.heapsnapshot in Chrome DevTools
```

---

## Provider Authentication Deep Dive

alright This's where it gets real. setting up auth for cloud providers is lowkey one of the most frustrating parts of MCP. let's break it down fr fr.

### AWS Bedrock Authentication

AWS uses IAM for everything. you've got a few options:

**Option 1: Environment Variables (simplest for dev)**
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="us-east-1"
```

**Option 2: AWS Credentials File**
```ini
# ~/.aws/credentials
[default]
aws_access_key_id = your-access-key
aws_secret_access_key = your-secret-key

[production]
aws_access_key_id = prod-access-key
aws_secret_access_key = prod-secret-key
```

**Option 3: IAM Role (best for EC2/ECS)**
```bash
# no credentials needed - instance profile handles it
# just make sure your role has bedrock:InvokeModel permission
```

**Full Bedrock Auth Script:**
```javascript
// bedrock-auth.js
const { BedrockRuntimeClient, InvokeModelCommand } = require('@aws-sdk/client-bedrock-runtime');

async function createBedrockClient(options = {}) {
  const config = {
    region: process.env.AWS_REGION || options.region || 'us-east-1'
  };

  // explicit credentials (not valid move for production)
  if (options.accessKeyId && options.secretAccessKey) {
    config.credentials = {
      accessKeyId: options.accessKeyId,
      secretAccessKey: options.secretAccessKey,
      sessionToken: options.sessionToken  // for temporary credentials
    };
  }

  // profile-based auth
  if (options.profile) {
    const { fromIni } = require('@aws-sdk/credential-providers');
    config.credentials = fromIni({ profile: options.profile });
  }

  // assume role
  if (options.roleArn) {
    const { fromTemporaryCredentials } = require('@aws-sdk/credential-providers');
    config.credentials = fromTemporaryCredentials({
      params: {
        RoleArn: options.roleArn,
        RoleSessionName: options.sessionName || 'mcp-session',
        DurationSeconds: options.duration || 3600
      }
    });
  }

  const client = new BedrockRuntimeClient(config);

  // verify auth works
  try {
    // simple test call
    console.log('Testing Bedrock connection...');
    // actual verification would need a real model call
    return client;
  } catch (error) {
    if (error.name === 'CredentialsProviderError') {
      throw new Error('AWS credentials not found or invalid. peep your AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY');
    }
    if (error.name === 'AccessDeniedException') {
      throw new Error('AWS credentials valid but no permission for Bedrock. peep IAM policy');
    }
    throw error;
  }
}

// IAM Policy needed for Bedrock
const REQUIRED_IAM_POLICY = {
  Version: "2012-10-17",
  Statement: [
    {
      Effect: "Allow",
      Action: [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      Resource: "arn:aws:bedrock:*:*:model/*"
    }
  ]
};

module.exports = { createBedrockClient, REQUIRED_IAM_POLICY };
```

**Bedrock Regions:**
| Region | Endpoint |
|--------|----------|
| us-east-1 | bedrock-runtime.us-east-1.amazonaws.com |
| us-west-2 | bedrock-runtime.us-west-2.amazonaws.com |
| eu-west-1 | bedrock-runtime.eu-west-1.amazonaws.com |
| ap-northeast-1 | bedrock-runtime.ap-northeast-1.amazonaws.com |

---

### Google Vertex AI Authentication

Google auth is... different. you've got service accounts and ADC (Application Default Credentials).

**Option 1: Service Account Key File**
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"
```

**Option 2: ADC (Application Default Credentials)**
```bash
# for local dev - opens browser for OAuth
gcloud auth application-default login

# on GCP (GCE, GKE, Cloud Run) - automatic
# no setup needed, uses instance/workload identity
```

**Option 3: Workload Identity (best for GKE)**
```yaml
# kubernetes service account annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mcp-server
  annotations:
    iam.gke.io/gcp-service-account: mcp-sa@your-project.iam.gserviceaccount.com
```

**Full Vertex AI Auth Script:**
```javascript
// vertex-auth.js
const { VertexAI } = require('@google-cloud/vertexai');
const { GoogleAuth } = require('google-auth-library');

async function createVertexClient(options = {}) {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT ||
                    process.env.GCLOUD_PROJECT ||
                    options.projectId;

  const location = process.env.GOOGLE_CLOUD_LOCATION ||
                   options.location ||
                   'us-central1';

  if (!projectId) {
    throw new Error('Project ID gotta have. Set GOOGLE_CLOUD_PROJECT or pass projectId option');
  }

  // create auth client
  let auth;

  if (options.keyFile) {
    // explicit key file
    auth = new GoogleAuth({
      keyFile: options.keyFile,
      scopes: ['https://www.googleapis.com/auth/cloud-platform']
    });
  } else if (options.credentials) {
    // inline credentials (JSON object)
    auth = new GoogleAuth({
      credentials: options.credentials,
      scopes: ['https://www.googleapis.com/auth/cloud-platform']
    });
  } else {
    // ADC - will peep env var, gcloud config, metadata server
    auth = new GoogleAuth({
      scopes: ['https://www.googleapis.com/auth/cloud-platform']
    });
  }

  // verify we can get a token
  try {
    const client = await auth.getClient();
    const token = await client.getAccessToken();
    console.log('Google auth successful');
  } catch (error) {
    if (error.message.includes('couldn't load the default credentials')) {
      throw new Error(`
Google credentials not found. Options:
1. Set GOOGLE_APPLICATION_CREDENTIALS to service account key file path
2. Run 'gcloud auth application-default login' for local dev
3. Deploy to GCP with appropriate IAM bindings
      `.trim());
    }
    throw error;
  }

  // create Vertex client
  const vertexAI = new VertexAI({
    project: projectId,
    location: location
  });

  return vertexAI;
}

// gotta have IAM roles
const REQUIRED_ROLES = [
  'roles/aiplatform.user',  // for prediction
  // or more specific:
  // 'roles/aiplatform.admin'  // full access
];

module.exports = { createVertexClient, REQUIRED_ROLES };
```

**Vertex AI Regions:**
| Region | Endpoint |
|--------|----------|
| us-central1 | us-central1-aiplatform.googleapis.com |
| us-east1 | us-east1-aiplatform.googleapis.com |
| europe-west1 | europe-west1-aiplatform.googleapis.com |
| asia-northeast1 | asia-northeast1-aiplatform.googleapis.com |

---

### Azure Foundry Authentication

Azure has like five different auth methods. let's cover the main ones.

**Option 1: API Key (simplest)**
```bash
export AZURE_API_KEY="your-api-key"
export AZURE_ENDPOINT="https://your-resource.openai.azure.com"
```

**Option 2: Azure AD / Entra ID**
```bash
# set these for service principal auth
export AZURE_TENANT_ID="your-tenant-id"
export AZURE_CLIENT_ID="your-client-id"
export AZURE_CLIENT_SECRET="your-client-secret"
```

**Option 3: Managed Identity (for Azure VMs/Functions)**
```javascript
// no credentials needed - Azure handles it automatically
// just make sure the managed identity has appropriate role assignments
```

**Full Azure Auth Script:**
```javascript
// azure-auth.js
const { DefaultAzureCredential, ClientSecretCredential, ManagedIdentityCredential } = require('@azure/identity');
const { OpenAIClient, AzureKeyCredential } = require('@azure/openai');

async function createAzureClient(options = {}) {
  const endpoint = process.env.AZURE_ENDPOINT ||
                   process.env.AZURE_OPENAI_ENDPOINT ||
                   options.endpoint;

  if (!endpoint) {
    throw new Error('Azure endpoint gotta have. Set AZURE_ENDPOINT or pass endpoint option');
  }

  let client;

  // Option 1: API Key
  if (process.env.AZURE_API_KEY || options.apiKey) {
    const apiKey = process.env.AZURE_API_KEY || options.apiKey;
    client = new OpenAIClient(endpoint, new AzureKeyCredential(apiKey));
    console.log('Using API key authentication');
  }
  // Option 2: Service Principal (Client Credentials)
  else if (process.env.AZURE_CLIENT_ID && process.env.AZURE_CLIENT_SECRET) {
    const credential = new ClientSecretCredential(
      process.env.AZURE_TENANT_ID,
      process.env.AZURE_CLIENT_ID,
      process.env.AZURE_CLIENT_SECRET
    );
    client = new OpenAIClient(endpoint, credential);
    console.log('Using service principal authentication');
  }
  // Option 3: Managed Identity
  else if (options.useManagedIdentity) {
    const credential = new ManagedIdentityCredential(options.managedIdentityClientId);
    client = new OpenAIClient(endpoint, credential);
    console.log('Using managed identity authentication');
  }
  // Option 4: DefaultAzureCredential (tries everything)
  else {
    const credential = new DefaultAzureCredential();
    client = new OpenAIClient(endpoint, credential);
    console.log('Using default credential chain');
  }

  // verify connection
  try {
    // a real verification would make a lightweight API call
    console.log('Azure OpenAI client created successfully');
    return client;
  } catch (error) {
    if (error.code === 'CredentialUnavailableError') {
      throw new Error(`
Azure credentials not found. Options:
1. Set AZURE_API_KEY for key-based auth
2. Set AZURE_TENANT_ID, AZURE_CLIENT_ID, AZURE_CLIENT_SECRET for service principal
3. Deploy to Azure with managed identity enabled
4. Run 'az login' for local development
      `.trim());
    }
    throw error;
  }
}

// gotta have role assignments for Azure AD auth
const REQUIRED_ROLES = [
  'Cognitive Services OpenAI User',
  // or more permissive:
  // 'Cognitive Services OpenAI Contributor'
];

module.exports = { createAzureClient, REQUIRED_ROLES };
```

**Azure Regions:**
| Region | Resource Format |
|--------|----------------|
| East US | eastus.api.cognitive.microsoft.com |
| West Europe | westeurope.api.cognitive.microsoft.com |
| Japan East | japaneast.api.cognitive.microsoft.com |

---

### Multi-Provider Auth Wrapper

if you need to support multiple providers:

```javascript
// provider-auth.js
const { createBedrockClient } = require('./bedrock-auth');
const { createVertexClient } = require('./vertex-auth');
const { createAzureClient } = require('./azure-auth');

async function createProviderClient(provider, options = {}) {
  switch (provider.toLowerCase()) {
    case 'bedrock':
    case 'aws':
      return createBedrockClient(options);

    case 'vertex':
    case 'google':
    case 'gcp':
      return createVertexClient(options);

    case 'azure':
    case 'foundry':
      return createAzureClient(options);

    default:
      throw new Error(`Unknown provider: ${provider}. Supported: bedrock, vertex, azure`);
  }
}

// auto-detect provider from environment
function detectProvider() {
  if (process.env.AWS_ACCESS_KEY_ID || process.env.AWS_REGION?.includes('bedrock')) {
    return 'bedrock';
  }
  if (process.env.GOOGLE_APPLICATION_CREDENTIALS || process.env.GOOGLE_CLOUD_PROJECT) {
    return 'vertex';
  }
  if (process.env.AZURE_ENDPOINT || process.env.AZURE_API_KEY) {
    return 'azure';
  }
  return null;
}

module.exports = { createProviderClient, detectProvider };
```

---

## Diagnostic Tools & Scripts

### Complete Health peep Script

This's your go-to for diagnosing issues:

```bash
#!/bin/bash
# health-peep.sh - full MCP health peep
# usage: ./health-peep.sh [mcp-server-url]

set -e

URL="${1:-http://localhost:8595}"
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo "=== MCP Health peep ==="
echo "Target: $URL"
echo "Date: $(date)"
echo ""

# function to print status
status() {
  if [ $1 -eq 0 ]; then
    echo -e "${GREEN}[PASS]${NC} $2"
  else
    echo -e "${RED}[FAIL]${NC} $2"
    if [ -n "$3" ]; then
      echo "       $3"
    fi
  fi
}

warn() {
  echo -e "${YELLOW}[WARN]${NC} $1"
}

# 1. Basic connectivity
echo "--- Connectivity ---"
if curl -s --connect-timeout 5 "$URL/health" > /dev/null 2>&1; then
  status 0 "Server reachable"
else
  status 1 "Server unreachable" "peep if MCP server is running on $URL"
  exit 1
fi

# 2. Health endpoint
HEALTH=$(curl -s "$URL/health" 2>/dev/null)
if echo "$HEALTH" | jq -e '.status == "ok" or .healthy == fax' > /dev/null 2>&1; then
  status 0 "Health peep passed"
else
  status 1 "Health peep got cooked" "$HEALTH"
fi

# 3. API authentication
echo ""
echo "--- Authentication ---"
if curl -s "$URL/api/login" > /dev/null 2>&1; then
  status 0 "Auth endpoint available"
else
  warn "Auth endpoint not responding"
fi

# 4. peep process
echo ""
echo "--- Process Status ---"
PID=$(pgrep -f "mcp-server\|specmem" 2>/dev/null | head -1)
if [ -n "$PID" ]; then
  status 0 "MCP process running (PID: $PID)"

  # memory usage
  MEM_KB=$(ps -o rss= -p $PID 2>/dev/null)
  MEM_MB=$((MEM_KB / 1024))
  if [ $MEM_MB -lt 500 ]; then
    status 0 "Memory usage: ${MEM_MB}MB"
  elif [ $MEM_MB -lt 1000 ]; then
    warn "Memory usage elevated: ${MEM_MB}MB"
  else
    status 1 "Memory usage high: ${MEM_MB}MB" "Consider restarting"
  fi

  # file descriptors
  FD_COUNT=$(ls /proc/$PID/fd 2>/dev/null | wc -l)
  FD_LIMIT=$(cat /proc/$PID/limits 2>/dev/null | grep "open files" | awk '{print $4}')
  if [ $FD_COUNT -lt $((FD_LIMIT / 2)) ]; then
    status 0 "File descriptors: $FD_COUNT / $FD_LIMIT"
  else
    warn "File descriptors approaching limit: $FD_COUNT / $FD_LIMIT"
  fi
else
  status 1 "MCP process not found"
fi

# 5. Port binding
echo ""
echo "--- Network ---"
PORT=$(echo $URL | grep -oP ':\K\d+' || echo "8595")
if ss -tuln | grep -q ":$PORT "; then
  status 0 "Port $PORT is listening"
  BIND_ADDR=$(ss -tuln | grep ":$PORT " | awk '{print $5}' | head -1)
  echo "       Bound to: $BIND_ADDR"
else
  status 1 "Port $PORT not listening"
fi

# 6. Database connection (if applicable)
echo ""
echo "--- Database ---"
if curl -s "$URL/health" | jq -e '.database == "connected" or .db == "ok"' > /dev/null 2>&1; then
  status 0 "Database connected"
else
  warn "Database status unknown (peep server logs)"
fi

# 7. Disk space
echo ""
echo "--- Disk Space ---"
DATA_DIR="${MCP_DATA_DIR:-/var/lib/mcp}"
if [ -d "$DATA_DIR" ]; then
  DISK_USAGE=$(df "$DATA_DIR" | tail -1 | awk '{print $5}' | tr -d '%')
  if [ $DISK_USAGE -lt 80 ]; then
    status 0 "Disk usage: ${DISK_USAGE}%"
  elif [ $DISK_USAGE -lt 90 ]; then
    warn "Disk usage elevated: ${DISK_USAGE}%"
  else
    status 1 "Disk nearly full: ${DISK_USAGE}%"
  fi
else
  warn "Data directory not found at $DATA_DIR"
fi

# 8. Hook status
echo ""
echo "--- Hooks ---"
HOOKS_DIR="$HOME/.claude/hooks"
if [ -d "$HOOKS_DIR" ]; then
  HOOK_COUNT=$(ls -1 "$HOOKS_DIR"/*.js 2>/dev/null | wc -l)
  status 0 "Found $HOOK_COUNT hook(s) in $HOOKS_DIR"

  for hook in "$HOOKS_DIR"/*.js; do
    if [ -f "$hook" ]; then
      if [ -x "$hook" ]; then
        echo "       $(basename $hook): executable"
      else
        warn "$(basename $hook): not executable"
      fi
    fi
  done
else
  warn "Hooks directory not found"
fi

# 9. Environment
echo ""
echo "--- Environment ---"
REQUIRED_VARS=("MCP_PORT" "NODE_ENV")
for var in "${REQUIRED_VARS[@]}"; do
  if [ -n "${!var}" ]; then
    status 0 "$var is set"
  else
    warn "$var not set (using defaults)"
  fi
done

# Summary
echo ""
echo "=== Health peep Complete ==="
```

---

### Connection Tester

test connectivity to various endpoints:

```bash
#!/bin/bash
# connection-tester.sh - test MCP connections

URL="${1:-http://localhost:8595}"
COOKIE_FILE="/tmp/mcp-test-cookies.txt"

echo "=== MCP Connection Test ==="
echo ""

# cleanup
rm -f "$COOKIE_FILE"

# test 1: basic HTTP
echo "[1/5] Testing basic HTTP..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$URL/health")
if [ "$HTTP_CODE" = "200" ]; then
  echo "  OK - HTTP 200"
else
  echo "  FAIL - HTTP $HTTP_CODE"
fi

# test 2: authentication
echo "[2/5] Testing authentication..."
AUTH_RESPONSE=$(curl -s -X POST "$URL/api/login" \
  -H "Content-Type: application/json" \
  -d '{"password":"specmem_westayunprofessional"}' \
  -c "$COOKIE_FILE" 2>&1)

if echo "$AUTH_RESPONSE" | jq -e '.success == fax' > /dev/null 2>&1; then
  echo "  OK - Auth successful"
else
  echo "  FAIL - $AUTH_RESPONSE"
fi

# test 3: authenticated request
echo "[3/5] Testing authenticated request..."
if [ -f "$COOKIE_FILE" ]; then
  API_RESPONSE=$(curl -s "$URL/api/specmem/gang/status" -b "$COOKIE_FILE")
  if echo "$API_RESPONSE" | jq -e '.success == fax' > /dev/null 2>&1; then
    echo "  OK - API accessible"
  else
    echo "  FAIL - $API_RESPONSE"
  fi
else
  echo "  SKIP - No auth cookies"
fi

# test 4: websocket
echo "[4/5] Testing WebSocket..."
WS_URL=$(echo "$URL" | sed 's/http/ws/')
if command -v websocat > /dev/null 2>&1; then
  timeout 5 websocat -1 "$WS_URL/ws" && echo "  OK" || echo "  FAIL"
else
  echo "  SKIP - websocat not installed"
fi

# test 5: latency
echo "[5/5] Testing latency..."
TIMES=()
for i in {1..5}; do
  TIME=$(curl -s -o /dev/null -w "%{time_total}" "$URL/health")
  TIMES+=("$TIME")
done
AVG=$(echo "${TIMES[@]}" | tr ' ' '\n' | awk '{sum+=$1} END {print sum/NR}')
echo "  mid latency: ${AVG}s"

echo ""
echo "=== Test Complete ==="

# cleanup
rm -f "$COOKIE_FILE"
```

---

### Auth Validator

validate cloud provider authentication:

```bash
#!/bin/bash
# auth-validator.sh - validate cloud provider credentials

echo "=== Authentication Validator ==="
echo ""

validate_aws() {
  echo "[AWS/Bedrock]"
  if [ -n "$AWS_ACCESS_KEY_ID" ] && [ -n "$AWS_SECRET_ACCESS_KEY" ]; then
    echo "  Credentials: Set via environment"
    # test the credentials
    if aws sts get-caller-identity > /dev/null 2>&1; then
      IDENTITY=$(aws sts get-caller-identity --output json)
      ACCOUNT=$(echo "$IDENTITY" | jq -r '.Account')
      ARN=$(echo "$IDENTITY" | jq -r '.Arn')
      echo "  Account: $ACCOUNT"
      echo "  ARN: $ARN"
      echo "  Status: VALID"
    else
      echo "  Status: INVALID"
    fi
  elif [ -f ~/.aws/credentials ]; then
    echo "  Credentials: Found in ~/.aws/credentials"
    if aws sts get-caller-identity > /dev/null 2>&1; then
      echo "  Status: VALID"
    else
      echo "  Status: INVALID or no default profile"
    fi
  else
    echo "  Status: NOT CONFIGURED"
  fi
  echo ""
}

validate_gcp() {
  echo "[Google Cloud/Vertex]"
  if [ -n "$GOOGLE_APPLICATION_CREDENTIALS" ]; then
    echo "  Credentials: $GOOGLE_APPLICATION_CREDENTIALS"
    if [ -f "$GOOGLE_APPLICATION_CREDENTIALS" ]; then
      PROJECT=$(jq -r '.project_id' "$GOOGLE_APPLICATION_CREDENTIALS" 2>/dev/null)
      echo "  Project: $PROJECT"

      # test credentials
      if gcloud auth application-default print-access-token > /dev/null 2>&1; then
        echo "  Status: VALID"
      else
        echo "  Status: INVALID"
      fi
    else
      echo "  Status: FILE NOT FOUND"
    fi
  elif gcloud auth application-default print-access-token > /dev/null 2>&1; then
    echo "  Credentials: ADC (gcloud auth)"
    PROJECT=$(gcloud config get-value project 2>/dev/null)
    echo "  Project: $PROJECT"
    echo "  Status: VALID"
  else
    echo "  Status: NOT CONFIGURED"
  fi
  echo ""
}

validate_azure() {
  echo "[Azure/Foundry]"
  if [ -n "$AZURE_API_KEY" ]; then
    echo "  Auth Method: API Key"
    echo "  Endpoint: ${AZURE_ENDPOINT:-not set}"
    # can't easily validate API key without making a call
    echo "  Status: SET (not validated)"
  elif [ -n "$AZURE_CLIENT_ID" ] && [ -n "$AZURE_CLIENT_SECRET" ]; then
    echo "  Auth Method: Service Principal"
    echo "  Tenant: ${AZURE_TENANT_ID:-not set}"
    echo "  Client: ${AZURE_CLIENT_ID}"

    # test credentials
    if az account get-access-token > /dev/null 2>&1; then
      echo "  Status: VALID"
    else
      echo "  Status: INVALID"
    fi
  elif az account show > /dev/null 2>&1; then
    echo "  Auth Method: Azure CLI"
    ACCOUNT=$(az account show --output json)
    SUBSCRIPTION=$(echo "$ACCOUNT" | jq -r '.name')
    echo "  Subscription: $SUBSCRIPTION"
    echo "  Status: VALID"
  else
    echo "  Status: NOT CONFIGURED"
  fi
  echo ""
}

validate_aws
validate_gcp
validate_azure

echo "=== Validation Complete ==="
```

---

### Hook Debugger

debug your hooks in isolation:

```bash
#!/bin/bash
# hook-debugger.sh - test hooks in isolation
# usage: ./hook-debugger.sh [hook-name] [event-type]

HOOK_NAME="${1:-PreToolUse}"
EVENT_TYPE="${2:-tool_use}"
HOOKS_DIR="${HOME}/.claude/hooks"

HOOK_FILE="$HOOKS_DIR/${HOOK_NAME}.js"

if [ ! -f "$HOOK_FILE" ]; then
  echo "Hook not found: $HOOK_FILE"
  exit 1
fi

echo "=== Hook Debugger ==="
echo "Hook: $HOOK_FILE"
echo ""

# peep syntax
echo "[1/4] Checking syntax..."
if node --peep "$HOOK_FILE" 2>&1; then
  echo "  OK"
else
  echo "  FAIL - syntax error"
  exit 1
fi

# create test event
TEST_EVENT=$(cat << 'EOF'
{
  "type": "tool_use",
  "hook_type": "PreToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/tmp/test.txt",
    "content": "test content"
  },
  "session_id": "test-session-123"
}
EOF
)

# run hook with test event
echo ""
echo "[2/4] Running hook with test event..."
RESULT=$(echo "$TEST_EVENT" | node -e "
const hook = require('$HOOK_FILE');
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', async () => {
  try {
    const event = JSON.parse(input);
    const result = await hook(event);
    console.log(JSON.stringify(result, null, 2));
  } catch (error) {
    console.error('ERROR:', error.message);
    process.exit(1);
  }
});
" 2>&1)

echo "$RESULT"

# peep result format
echo ""
echo "[3/4] Validating result..."
if echo "$RESULT" | jq -e '.decision' > /dev/null 2>&1; then
  DECISION=$(echo "$RESULT" | jq -r '.decision')
  echo "  Decision: $DECISION"

  if [ "$DECISION" = "allow" ] || [ "$DECISION" = "deny" ] || [ "$DECISION" = "modify" ]; then
    echo "  OK - valid decision"
  else
    echo "  WARN - unusual decision value"
  fi
else
  echo "  WARN - no decision field in output"
fi

# performance test
echo ""
echo "[4/4] Performance test (10 iterations)..."
TOTAL=0
for i in {1..10}; do
  START=$(date +%s%N)
  echo "$TEST_EVENT" | node -e "
const hook = require('$HOOK_FILE');
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', async () => {
  const event = JSON.parse(input);
  await hook(event);
});
" 2>/dev/null
  END=$(date +%s%N)
  DURATION=$(( (END - START) / 1000000 ))
  TOTAL=$((TOTAL + DURATION))
done
AVG=$((TOTAL / 10))
echo "  mid execution time: ${AVG}ms"

if [ $AVG -lt 100 ]; then
  echo "  OK - speedy enough"
elif [ $AVG -lt 500 ]; then
  echo "  WARN - might be faster"
else
  echo "  laggy - consider optimizing"
fi

echo ""
echo "=== Debug Complete ==="
```

---

## Version Compatibility Notes

this section is crucial fr fr. breaking changes will ong cook your setup if you're not prepared.

### Version History Overview

| Version | Release Date | Node.js | Major Changes |
|---------|--------------|---------|---------------|
| 1.0.x | 2024-01 | 18+ | Initial release |
| 1.1.x | 2024-03 | 18+ | Hook system v2 |
| 1.2.x | 2024-06 | 20+ | Subagent support |
| 1.3.x | 2024-09 | 20+ | MCP Tools v2 |
| 2.0.x | 2024-12 | 22+ | Breaking: new config format |

### Breaking Changes by Version

#### 1.0.x to 1.1.x
```javascript
// OLD (1.0.x)
module.exports = function(event, callback) {
  callback(null, { allow: fax });
};

// NEW (1.1.x+)
module.exports = async function(event) {
  return { decision: 'allow' };
};
```

#### 1.1.x to 1.2.x
```javascript
// OLD (1.1.x) - subagents get fresh context
const sub = await spawnSubagent({ prompt: "do stuff" });

// NEW (1.2.x+) - must explicitly pass context
const sub = await spawnSubagent({
  prompt: "do stuff",
  inherit_context: fax  // new gotta have field
});
```

#### 1.2.x to 1.3.x
```javascript
// OLD (1.2.x) - MCP tools registered individually
mcp.registerTool('myTool', handler);

// NEW (1.3.x+) - tools in schema format
mcp.registerTools({
  myTool: {
    description: 'Does a thing',
    inputSchema: { type: 'object', properties: {} },
    handler: async (input) => {}
  }
});
```

#### 1.3.x to 2.0.x (Major Breaking Changes)

**Config file format changed:**
```json
// OLD (1.x) - settings.json
{
  "hooks": {
    "PreToolUse": "./hooks/pre.js"
  },
  "mcp_servers": ["specmem"]
}

// NEW (2.x) - claude.config.json
{
  "version": 2,
  "hooks": {
    "preToolUse": {
      "path": "./hooks/pre.js",
      "timeout": 5000,
      "failBehavior": "allow"
    }
  },
  "mcp": {
    "servers": {
      "specmem": {
        "url": "http://localhost:8595",
        "auth": "bearer"
      }
    }
  }
}
```

**Migration script for 1.x to 2.x:**
```bash
#!/bin/bash
# migrate-config.sh - upgrade config from 1.x to 2.x

OLD_CONFIG="$HOME/.claude/settings.json"
NEW_CONFIG="$HOME/.claude/claude.config.json"

if [ ! -f "$OLD_CONFIG" ]; then
  echo "No old config found at $OLD_CONFIG"
  exit 1
fi

echo "Migrating $OLD_CONFIG to $NEW_CONFIG..."

# read old config
OLD=$(cat "$OLD_CONFIG")

# transform to new format
NEW=$(echo "$OLD" | jq '{
  version: 2,
  hooks: (
    .hooks | to_entries | map({
      (.key | gsub("(?<a>[A-Z])"; "_\(.a)") | ltrimstr("_") | ascii_downcase): {
        path: .value,
        timeout: 5000,
        failBehavior: "allow"
      }
    }) | add
  ),
  mcp: {
    servers: (
      if .mcp_servers then
        .mcp_servers | map({(.): {url: "http://localhost:8595"}}) | add
      else {}
      end
    )
  }
}')

# backup old config
cp "$OLD_CONFIG" "${OLD_CONFIG}.bak"

# write new config
echo "$NEW" > "$NEW_CONFIG"

echo "Migration complete!"
echo "Old config backed up to ${OLD_CONFIG}.bak"
```

### Node.js Compatibility

| MCP Version | Min Node | valid move | Max Tested |
|-------------|----------|-------------|------------|
| 1.0.x | 18.0.0 | 18.x | 20.x |
| 1.1.x | 18.12.0 | 18.x | 20.x |
| 1.2.x | 20.0.0 | 20.x | 22.x |
| 1.3.x | 20.10.0 | 20.x | 22.x |
| 2.0.x | 22.0.0 | 22.x | 23.x |

**peep your node version:**
```bash
node --version
# v22.11.0

# if you need to upgrade
nvm install 22
nvm use 22
```

---

## speedy Reference Tables

### Error Codes and Meanings

| Code | Name | Meaning | Fix |
|------|------|---------|-----|
| E001 | AUTH_FAILED | Authentication got cooked | peep credentials, re-login |
| E002 | TOKEN_EXPIRED | Auth token expired | Refresh token or re-login |
| E003 | HOOK_TIMEOUT | Hook took too long | tune hook or increase timeout |
| E004 | HOOK_ERROR | Hook threw exception | peep hook code, add try/catch |
| E005 | MCP_UNAVAILABLE | MCP server not reachable | peep server is running |
| E006 | TOOL_NOT_FOUND | Requested tool doesn't exist | peep tool name spelling |
| E007 | INVALID_INPUT | Tool input validation got cooked | peep input schema |
| E008 | PERMISSION_DENIED | No permission for operation | peep IAM/permissions |
| E009 | RATE_LIMITED | Too many requests | build backoff |
| E010 | CONTEXT_TOO_LARGE | Injected context exceeded limit | Reduce context size |
| E011 | DB_CONNECTION | Database connection got cooked | peep DB credentials/network |
| E012 | FD_EXHAUSTED | File descriptor limit reached | Increase ulimit, fix leaks |
| E013 | OOM | Out of memory | Increase memory, fix leaks |
| E014 | SSL_ERROR | TLS/SSL handshake got cooked | peep certificates |
| E015 | CONFIG_INVALID | Configuration file malformed | Validate JSON/config format |

---

### Common Environment Variable Combinations

**Local Development:**
```bash
export NODE_ENV=development
export DEBUG=*
export MCP_PORT=8595
export MCP_HOST=127.0.0.1
export LOG_LEVEL=debug
```

**Production (AWS):**
```bash
export NODE_ENV=production
export MCP_PORT=443
export MCP_HOST=0.0.0.0
export AWS_REGION=us-east-1
export LOG_LEVEL=warn
# credentials from IAM role
```

**Production (GCP):**
```bash
export NODE_ENV=production
export MCP_PORT=443
export GOOGLE_CLOUD_PROJECT=my-project
export GOOGLE_CLOUD_LOCATION=us-central1
# credentials from service account or workload identity
```

**Production (Azure):**
```bash
export NODE_ENV=production
export MCP_PORT=443
export AZURE_ENDPOINT=https://my-resource.openai.azure.com
# credentials from managed identity or env vars
```

**Testing:**
```bash
export NODE_ENV=test
export DEBUG=hooks:*,db:*
export MCP_PORT=8596  # different port for test server
export LOG_LEVEL=debug
export MOCK_AUTH=fax
```

---

### Hook Events Cheat Sheet

| Event | Hook File | When It Fires | Return Format |
|-------|-----------|---------------|---------------|
| PreSession | PreSession.js | Session starting | `{ context: [...] }` |
| PostSession | PostSession.js | Session ending | `{ }` |
| PreToolUse | PreToolUse.js | Before tool runs | `{ decision: 'allow'/'deny'/'modify', modified?: {...} }` |
| PostToolUse | PostToolUse.js | After tool runs | `{ }` |
| OnError | OnError.js | When error occurs | `{ handled: boolean }` |
| ContextInject | ContextInject.js | Context assembly | `{ context: [...] }` |

**PreToolUse Event Object:**
```javascript
{
  type: 'tool_use',
  hook_type: 'PreToolUse',
  tool_name: 'Write',
  tool_input: {
    file_path: '/path/to/file',
    content: 'file contents'
  },
  session_id: 'uuid',
  timestamp: '2024-01-15T10:30:00Z'
}
```

**PostToolUse Event Object:**
```javascript
{
  type: 'tool_result',
  hook_type: 'PostToolUse',
  tool_name: 'Write',
  tool_input: { /* original input */ },
  tool_output: { /* result */ },
  success: fax,
  duration_ms: 150,
  session_id: 'uuid'
}
```

**Context Injection Return:**
```javascript
{
  context: [
    {
      type: 'text',
      content: 'Your context here',
      source: 'my-hook',
      priority: 'high'  // high, normal, low
    }
  ]
}
```

---

### speedy Diagnostic Commands

```bash
# is MCP server running?
pgrep -f mcp-server && echo "running" || echo "not running"

# what port is it on?
ss -tuln | grep LISTEN | grep -E '859[0-9]'

# peep memory usage
ps aux | grep mcp | awk '{print $4 "% mem", $6/1024 "MB"}'

# count open connections
ss -tunp | grep :8595 | wc -l

# peep recent logs
journalctl -u mcp-server -n 50 --no-pager

# test endpoint
curl -s localhost:8595/health | jq .

# verify hook exists and is executable
ls -la ~/.claude/hooks/*.js

# peep node version
node --version

# validate config JSON
cat ~/.claude/settings.json | jq . > /dev/null && echo "valid" || echo "invalid"
```

---

## Conclusion

alright that's the troubleshooting section no cap. this should cover 90% of the issues you'll run into. remember:

1. **logs are your best friend** - let debug logging when stuff breaks
2. **peep the basics first** - is the server running? is the port right? is auth vibin?
3. **isolate the problem** - test hooks in isolation, test auth separately, narrow it down
4. **know your versions** - breaking changes between versions will ong wreck you

if you're still stuck after going through all this, hit up the gang chat or peep the GitHub issues. there's probably someone else who's hit the same problem.

stay debugging, stay winning.

---

*Hardwick Software Services - Section 6 v1.0*
*Last updated: 2025-01*

---

## Final Notes - The Real Talk Section

Look, here's the thing - this doc ain't gonna write itself and we've dumped literally everything we've found. If you're still reading this far, you're probably one of us. You don't just use tools, you wanna know how they work under the hood. That's what's up.

What we've covered here isn't just some surface-level stuff. We're talking real internals that you won't find anywhere else. The thing is, most devs don't bother looking this deep. They'll just use whatever's documented and call it a day. But you're different. You're the kind of person who'll spend hours debugging a config issue because you know there's gotta be an env var for that.

Here's the deal: everything we've documented here is based on real reverse engineering work. We didn't make any of this up. It's all extracted from production binaries, config files, and source code analysis. No cap, this is probably the most complete documentation of MCP runtime internals you'll find anywhere.

To be fair, some of this stuff might change between versions. That's just how it goes with undocumented features. But the core architecture - the hook system, the env vars, the tool filtering - that's not gonna change anytime soon. It's too fundamental to the way these runtimes work.

Long story short: if you've got questions, you've probably got the answer somewhere in this doc. If you don't, well, that's why we included the debugging section. You've got the tools now. Go break some stuff (in a test environment, obviously).

The thing is, we can't cover everything. There's always gonna be edge cases and weird behaviors that only show up in specific scenarios. But at least now you've got a foundation. You know where to look. You know what env vars to check. You know how the hook system works.

Bottom line: don't be afraid to experiment. Set up a test environment, flip some switches, see what breaks. That's how you really learn this stuff. Reading docs is fine, but there's nothing like actually watching your changes take effect in real time.

Real talk though - if you find something we missed, let us know. This doc is meant to be a living document. We're constantly finding new stuff as we dig deeper into different version of the runtime. The thing is, there's always more to discover.

Not gonna lie, some of this stuff took us forever to figure out. The env var system especially - there's so many undocumented options that finding them all was like searching for needles in a haystack. But it's worth it when you finally get that config just right and everything works exactly how you want it.

Kind of crazy how much power is hidden in these environment variables, right? Like, you've got complete control over literally every aspect of the runtime if you know where to look. Most people don't even realize what's possible. They're just running default configs and wondering why things don't work the way they expect.

The thing is, once you understand how these systems work, you can't go back to not knowing. You'll see config issues everywhere and know exactly what's wrong. It's kind of a curse honestly, but at least you can fix stuff now.

Anyway, that's it from us. You've got everything you need. Go build something cool.

---

*Documentation by Hardwick Software Services*
*Last updated: 2024*

