# Inside Claude Code: The Tools and Architecture That Power Your AI Coding Assistant

I've been using Claude Code since its first release, watching it evolve from an experimental tool into something that's progressively transforming how we write software at Techery. We're in the process of making it our main coding agent, with more developers adopting it every week. After months of intensive use, I want to share what I've learned about how Claude Code actually works under the hood.

## Table of Contents

- [A Brief History](#a-brief-history)
- [Under the Hood: The Tech Stack](#under-the-hood-the-tech-stack)
- [What Actually Happens When You Type in Claude Code?](#what-actually-happens-when-you-type-in-claude-code)
- [A Quick Example: Refactoring in Action](#a-quick-example-refactoring-in-action)
- [The Tools That Power Claude Code](#the-tools-that-power-claude-code)
- [Tools in Action: Real-World Workflows](#tools-in-action-real-world-workflows)
- [Architectural Patterns](#architectural-patterns)
  - [Task Planning with TodoWrite/TodoRead](#task-planning-with-todowritetodoread)
  - [The Task Agent Architecture](#the-task-agent-architecture)
- [Conclusion](#conclusion)
- [The Complete Tool Reference](#the-complete-tool-reference)
- [Official Resources](#official-resources)

## A Brief History

Claude Code began as an internal tool at Anthropic, developed to help their own engineers work more efficiently. The tool proved so valuable that Anthropic's developers continue to use it daily for their own development work. After seeing its transformative impact internally, Anthropic decided to share it with the broader developer community.

Claude Code was publicly introduced in February 2025 as their first agentic coding tool, initially launched as a research preview alongside Claude 3.7 Sonnet. What started as an internal productivity tool quickly gained traction among external developers for its ability to handle complex, multi-step coding tasks autonomously.

By May 2025, at Anthropic's inaugural "Code with Claude" developer conference, Claude Code graduated from research preview to general availability. This release coincided with the launch of Claude 4 models (Sonnet and Opus), bringing significant enhancements including GitHub Actions support and native IDE integrations for VS Code and JetBrains.

What makes Claude Code unique isn't just its AI capabilities—it's the thoughtful engineering that transforms an LLM into a practical coding assistant. Built on Claude Opus 4's deep reasoning abilities, Claude Code can edit files, run commands, and orchestrate entire development workflows directly from your terminal or IDE.

## Under the Hood: The Tech Stack

One thing that surprised me when I first explored Claude Code's internals is how straightforward the implementation is. This isn't some massive, complex system—it's a relatively simple CLI application built with modern, familiar technologies:

- **TypeScript** - The entire codebase is written in TypeScript, providing type safety and excellent developer experience
- **React with Ink** - Yes, React in the terminal! Claude Code uses [Ink](https://github.com/vadimdemedes/ink) to render its TUI (Terminal User Interface), making the interface reactive and componentized
- **Anthropic SDK** - Uses the official TypeScript SDK to communicate with Claude's API
- **Node.js** - Runs on Node.js, making it cross-platform and easy to install via npm

What's remarkable is how straightforward it would be to implement a similar tool yourself. At its core, you need: a CLI that sends messages to Claude's API, a set of tool definitions that map to file system operations, and a loop that processes Claude's responses and executes the requested tools. Any developer comfortable with Node.js could build a basic version in a weekend - the real value comes from the thoughtful design decisions, safety mechanisms, and the polished user experience that Anthropic has refined through months of internal use.

## From Simple Tools to Sophisticated Workflows

Now that we've seen the surprisingly simple tech stack, you might wonder: how does this straightforward Node.js application become such a powerful coding assistant? The answer lies in how Claude Code orchestrates its tools to mirror real developer workflows. Let's look at what happens when you start a coding session.

## What Actually Happens When You Type in Claude Code?

When you fire up Claude Code and ask it to refactor a function or debug an issue, you're not just chatting with an AI—you're activating a sophisticated toolkit designed specifically for software engineering. While our [previous article](./loop.md) explained the agent loop pattern that powers all AI coding assistants, today we're diving deep into Claude Code's specific implementation.

If you've ever wondered why Claude Code feels different from other AI assistants, or why it behaves in specific ways, the answer lies in its toolkit. Let's explore the 15 tools that power every interaction.

## A Quick Example: Refactoring in Action

Before diving into the tools, let's see Claude Code in action. Here's what happens when you ask it to refactor a function:

```
You: "Refactor the getUserData function to use async/await instead of callbacks"

Claude Code:
1. [Grep] Searches for 'getUserData' across your codebase
2. [Read] Reads the file containing the function
3. [TodoWrite] Creates a plan for the refactoring
4. [Edit] Makes the necessary changes
5. [Bash] Runs your tests to ensure nothing broke
6. [TodoWrite] Marks the task as complete
```

This seamless workflow is powered by 15 specialized tools working in concert. Let's understand how they're organized.

## The Tools That Power Claude Code

What sets Claude Code apart isn't sophisticated AI magic—it's a thoughtfully designed set of tools that map directly to what developers actually do. These tools can be organized into logical groups that mirror typical development workflows (see the [complete tool reference](#the-complete-tool-reference) for detailed specifications):

**File Operations** (Read, Write, Edit, MultiEdit) - The foundation of any coding task. These tools handle everything from reading existing code to making surgical edits or batch refactoring.

**Search & Navigation** (Glob, Grep, LS) - Your exploration toolkit. Find files by pattern, search content with regex, and navigate directory structures—all optimized for speed even in massive codebases.

**Execution & Automation** (Bash, Task) - The power tools. Run any shell command or spawn autonomous sub-agents for complex operations. The Task tool is particularly clever—it delegates open-ended searches to keep your main conversation clean.

**Web Integration** (WebFetch, WebSearch) - Your window to the internet. Access documentation, search for current information, and process web content with AI to extract exactly what you need.

**Task Management** (TodoWrite, TodoRead) - The planning layer that makes Claude Code proactive rather than reactive. It automatically creates task lists for complex operations and tracks progress systematically.

**Jupyter Support** (NotebookRead, NotebookEdit) - Specialized tools for data science that understand notebook cell structure and outputs.

**MCP Extensions** - The extensibility layer where any third-party tool becomes a first-class citizen, seamlessly integrated into the agent loop.

## Tools in Action: Real-World Workflows

Now that we understand the tool categories, let's see how they work together in practice. These workflows demonstrate the power of Claude Code's orchestration.

### Workflow 1: Finding and Fixing a Bug

When you report a bug, here's how Claude Code combines tools to investigate and fix it:

```typescript
// You: "The API is returning 500 errors for user endpoints"

// Claude Code's workflow:
1. Grep("error.*500|500.*error", include: "*.log")     // Find error logs
2. Read("/var/logs/api.log", offset: 1000)             // Read recent logs
3. Grep("user.*endpoint|/api/user", include: "*.js")   // Find user endpoints
4. Read("src/api/userController.js")                   // Examine the code
5. TodoWrite([{ content: "Fix null check in getUserById" }])
6. Edit("src/api/userController.js", ...)              // Fix the bug
7. Bash("npm test -- userController.test.js")          // Verify the fix
```

### Workflow 2: Large-Scale Refactoring

Here's how Claude Code handles a complex refactoring across multiple files:

```typescript
// You: "Convert all Promise-based code to async/await"

// Claude Code's approach:
1. TodoWrite([                                          // Plan the work
    { content: "Find all Promise-based files", status: "pending" },
    { content: "Analyze patterns used", status: "pending" },
    { content: "Convert file by file", status: "pending" },
    { content: "Run tests after each conversion", status: "pending" }
])

2. Task(                                                // Delegate search
    description: "Find Promise patterns",
    prompt: "Search for .then(), .catch(), new Promise..."
)

3. MultiEdit("src/services/api.js", [                  // Batch updates
    { old_string: ".then(data =>", new_string: "const data = await" },
    { old_string: ".catch(err =>", new_string: "} catch(err) {" }
])

4. Bash("npm test")                                    // Verify changes
```

### Workflow 3: Creating New Features

When building something new, Claude Code follows structured patterns:

```typescript
// You: "Add a user authentication middleware"

// Tool orchestration:
1. Glob("**/middleware/*.js")                          // Find existing patterns
2. Read("src/middleware/logger.js")                    // Study conventions
3. TodoWrite([                                         // Plan implementation
    { content: "Create auth middleware file" },
    { content: "Add JWT verification logic" },
    { content: "Write unit tests" },
    { content: "Update route handlers" }
])
4. Write("src/middleware/auth.js", authCode)          // Create new file
5. Grep("app\\.(get|post|put|delete)", "src/routes")  // Find routes to update
6. MultiEdit(...)                                      // Add middleware to routes
7. Write("tests/middleware/auth.test.js", testCode)   // Add tests
8. Bash("npm test -- auth.test.js")                   // Run new tests
```

These workflows show how Claude Code's tools aren't just individual utilities—they're parts of an integrated system designed to handle real development tasks efficiently.

## Beyond Individual Tools: Architectural Patterns

While understanding individual tools is important, Claude Code's real power emerges from how these tools work together through sophisticated architectural patterns. These patterns transform a collection of utilities into an intelligent coding assistant that can handle complex, multi-step operations with minimal guidance.

Let's explore the two key patterns that make Claude Code uniquely effective:

## Architectural Patterns

### Task Planning with TodoWrite/TodoRead

One of Claude Code's most distinctive architectural patterns is its built-in task planning system. Unlike other AI coding assistants that might lose track of complex operations, Claude Code uses TodoWrite and TodoRead to maintain systematic progress.

**Important:** Claude Code is specifically instructed in its system prompt to call TodoWrite very frequently throughout conversations. This isn't optional behavior—it's a core requirement that ensures:

- Every multi-step task gets planned before execution
- Progress is tracked in real-time as work proceeds
- Users always have visibility into what's happening
- The assistant maintains context even in long conversations

**The Planning Pattern in Practice:**

Let's look at a real example. When you ask Claude Code to migrate a codebase from CommonJS to ES modules:

```typescript
// You: "Convert our Node.js project from CommonJS to ES modules"

// Claude Code's response:
"I'll help you convert your project to ES modules. Let me start by analyzing the current structure."

// 1. Creates comprehensive plan
TodoWrite([
  { id: '1', content: 'Analyze package.json and current module system', status: 'pending', priority: 'high' },
  { id: '2', content: 'Find all require() statements', status: 'pending', priority: 'high' },
  { id: '3', content: 'Find all module.exports patterns', status: 'pending', priority: 'high' },
  { id: '4', content: 'Update package.json with "type": "module"', status: 'pending', priority: 'high' },
  { id: '5', content: 'Convert require() to import statements', status: 'pending', priority: 'high' },
  { id: '6', content: 'Convert module.exports to export statements', status: 'pending', priority: 'high' },
  { id: '7', content: 'Update file extensions if needed', status: 'pending', priority: 'medium' },
  { id: '8', content: 'Run tests to verify everything works', status: 'pending', priority: 'high' },
]);

// 2. Systematic execution with real-time updates
TodoWrite([...todos, { id: '1', status: 'in_progress' }]);
Read("package.json");  // Examines current configuration

// Output shows current state
"Checking package.json... Currently using CommonJS (no 'type' field found)"

TodoWrite([...todos, { id: '1', status: 'completed' }, { id: '2', status: 'in_progress' }]);
Grep("require\\(", include: "*.js");  // Finds 47 files with require statements

// 3. Handles discoveries dynamically
"Found 47 files using require(). Also discovered some files using dynamic imports that need special handling."

TodoWrite([...todos, 
  { id: '9', content: 'Handle dynamic require() patterns separately', status: 'pending', priority: 'high' }
]);

// 4. Progress tracking keeps user informed
TodoRead(); 
// Shows: 1 completed, 1 in progress, 8 pending
```

**Why This Matters:**

- **No forgotten steps**: Every part of a complex task is tracked
- **Clear progress visibility**: Users see exactly what's being done
- **Interruption recovery**: Can resume after errors or user intervention
- **Systematic approach**: Forces decomposition of complex tasks
- **Context preservation**: Maintains awareness across long operations

**When Claude Code Uses TodoWrite:**

- Any task with 3 or more steps
- Before starting any complex operation
- After completing each subtask (status updates)
- When discovering new required steps during execution
- After errors to track recovery steps

The system prompt emphasizes that Claude Code should be "proactive" with task management, creating todos immediately upon receiving complex requests rather than waiting. This architectural decision transforms Claude Code from a reactive tool into a proactive assistant that plans and executes systematically.

### The Task Agent Architecture

The `Task` tool is Claude Code's solution to complex, exploratory operations that would otherwise clutter the main conversation. It spawns a complete sub-agent that operates independently with full tool access.

**How Task Agents Work:**

```
Main Claude Code Session
    │
    ├─> Task: Complex search/analysis request
    │     │
    │     └─> Sub-agent (stateless, autonomous)
    │           ├─> Has access to all 15+ tools
    │           ├─> Executes complete task independently
    │           ├─> No back-and-forth with main session
    │           └─> Returns consolidated final results
    │
    └─> Continues with clean context
```

**Real Example - Database Migration Analysis:**

Here's an actual Task agent in action, analyzing a codebase for database migration requirements:

```typescript
// You: "We need to migrate from MongoDB to PostgreSQL. Can you analyze what needs to be changed?"

// Claude Code spawns a task agent:
Task(
  description: "Analyze MongoDB to PostgreSQL migration",
  prompt: `Analyze the codebase to understand the scope of migrating from MongoDB to PostgreSQL:

  1. Find all MongoDB-specific code:
     - Mongoose models and schemas
     - Direct MongoDB driver usage
     - MongoDB-specific queries (aggregations, $lookup, etc.)
     - Connection and configuration code
  
  2. Identify data access patterns:
     - CRUD operations
     - Complex queries and aggregations
     - Transaction usage
     - Index definitions
  
  3. Analyze current data models:
     - Document structures that need normalization
     - Embedded documents vs references
     - Array fields and nested objects
  
  4. Find all affected files and estimate complexity:
     - Simple changes (connection strings)
     - Medium changes (query rewrites)
     - Complex changes (data model restructuring)

  Return a migration plan with:
  - Summary of MongoDB usage across the codebase
  - List of files requiring changes (grouped by complexity)
  - Data model transformation requirements
  - Suggested PostgreSQL schema based on current MongoDB structure
  - Estimated effort for each component`
)

// The sub-agent executes autonomously:
"Starting MongoDB to PostgreSQL migration analysis..."

// Sub-agent's internal workflow:
1. Grep("mongoose|mongodb", include: "*.js")         // Found in 73 files
2. Glob("**/models/*.js")                           // Located 15 model files
3. Read("src/models/User.js")                       // Analyzing schema structure
4. Grep("\\$lookup|\\$aggregate|\\$match")          // Finding complex queries
5. Task("Analyze User model", "Extract schema...")  // Nested analysis
6. Read("config/database.js")                       // Understanding connections

// Sub-agent returns comprehensive report:
"Migration Analysis Complete:

 SUMMARY:
 - 73 files use MongoDB (15 models, 28 controllers, 30 utilities)
 - 234 queries need rewriting (167 simple, 52 medium, 15 complex)
 - 5 models use heavy embedding requiring normalization
 
 CRITICAL CHANGES:
 1. User model: Embedded 'preferences' needs separate table
 2. Order model: Array of 'items' needs order_items junction table
 3. Analytics: MongoDB aggregation pipeline needs SQL rewrite
 
 SUGGESTED POSTGRESQL SCHEMA:
 [Detailed schema definitions...]
 
 EFFORT ESTIMATE: 120-150 hours
 - Phase 1: Schema design and setup (16-20 hours)
 - Phase 2: Model migration (40-50 hours)
 - Phase 3: Query rewriting (48-60 hours)
 - Phase 4: Testing and validation (16-20 hours)"
```

**What Makes This Powerful:**

The sub-agent handled hundreds of file operations, performed complex analysis, and even spawned its own sub-tasks—all without cluttering the main conversation. The user receives a clean, actionable report.

**User Control Over Agent Usage:**

Claude Code intelligently decides when to use sub-agents, but users can influence this behavior:

```typescript
// Force agent use with explicit language
'Please use an agent to search for all database queries across the codebase';

// Discourage agent use by being specific
'Check the src/api/users.js file for SQL queries'; // Direct search, no agent needed

// Request no agents
'Without using agents, show me the contents of package.json';
```

**When Claude Code Uses Task Agents:**

- Open-ended searches ("find all instances of X pattern")
- Multi-faceted analysis requiring many tool calls
- Exploratory tasks with uncertain scope
- Operations that would generate too much output for main context
- When the user explicitly requests it

**Benefits of the Task Agent Pattern:**

- **Context preservation**: Main conversation stays clean
- **Parallel exploration**: Can spawn multiple agents for different searches
- **Efficiency**: Reduces token usage in main context
- **Completeness**: Agent can be thorough without overwhelming the user
- **Specialization**: Each agent focuses on one specific task

This architectural pattern is particularly powerful for large codebases where exhaustive searches might require hundreds of tool calls.

## Conclusion

Claude Code's architecture represents a thoughtful balance of power and safety, designed specifically for software engineering workflows. Its 15 tools aren't arbitrary—they map directly to common developer actions. The restrictions aren't limitations—they're guardrails that ensure predictable, safe operations.

Understanding this architecture transforms Claude Code from a black box into a transparent system you can reason about. You know why it requires absolute paths (safety), why it batches operations (performance), why it delegates complex searches (efficiency), and why it maintains strict tool boundaries (predictability).

The streaming responses, session persistence, parallel execution, and extensible tool registry all work together to create a system that feels responsive and capable while maintaining the safety necessary for production code manipulation.

Whether you're using Claude Code for quick fixes or complex refactoring, understanding its architecture helps you work with the system rather than against it. The next time you see Claude Code spawn a sub-agent for a search or insist on reading before writing, you'll know these aren't quirks—they're features designed to make your coding experience both powerful and safe.

---

## The Complete Tool Reference

Claude Code's power comes from its carefully designed toolkit. Each tool serves a specific purpose in the software development workflow.

---

## 📁 File Operations

### 1. Read

**Purpose:** Read file contents with line numbers

**Parameters:**

- `file_path` (required) - Absolute path to file
- `offset` - Line number to start reading from
- `limit` - Number of lines to read (default: 2000)

**Key features:**

- Returns content in `cat -n` format
- Can display images visually
- Lines >2000 chars are truncated
- Required before editing existing files

### 2. Write

**Purpose:** Create new files or overwrite existing ones

**Parameters:**

- `file_path` (required) - Absolute path to file
- `content` (required) - Complete file content

**Key features:**

- Complete file replacement
- Creates directories if needed
- Must read existing files first (safety)

### 3. Edit

**Purpose:** Exact string replacement in files

**Parameters:**

- `file_path` (required) - Absolute path to file
- `old_string` (required) - Exact string to replace
- `new_string` (required) - Replacement string
- `replace_all` - Replace all occurrences (default: false)

**Key features:**

- Requires exact match including whitespace
- No regex support
- Fails if old_string not unique (unless replace_all=true)
- File must be read first

### 4. MultiEdit

**Purpose:** Multiple edits in one atomic operation

**Parameters:**

- `file_path` (required) - Absolute path to file
- `edits` (required) - Array of edit operations:
  - `old_string` - Text to replace
  - `new_string` - Replacement text
  - `replace_all` - Replace all occurrences

**Key features:**

- Sequential execution (order matters)
- All succeed or all fail
- More efficient than multiple Edit calls

---

## 🔍 Search and Navigation

### 5. Glob

**Purpose:** Find files by name pattern

**Parameters:**

- `pattern` (required) - Glob pattern to match
- `path` - Directory to search in (default: cwd)

**Key features:**

- Standard glob syntax (\*, \*\*, ?, [])
- Sorted by modification time (newest first)
- Respects .gitignore
- Fast on large codebases

### 6. Grep

**Purpose:** Search file contents with regex

**Parameters:**

- `pattern` (required) - Regex pattern to search
- `path` - Directory to search in (default: cwd)
- `include` - File pattern filter (e.g., "\*.js")

**Key features:**

- Full regex support (PCRE syntax)
- Returns file paths containing matches
- Skips binary files
- Use `rg` in Bash for detailed matches

### 7. LS

**Purpose:** List directory contents

**Parameters:**

- `path` (required) - Absolute directory path
- `ignore` - Array of glob patterns to ignore

**Key features:**

- Non-recursive (immediate children only)
- Shows hidden files
- Distinguishes files from directories
- Returns error if path doesn't exist

---

## ⚡ Execution and Automation

### 8. Bash

**Purpose:** Execute shell commands

**Parameters:**

- `command` (required) - Shell command to execute
- `timeout` - Timeout in ms (max: 600000)
- `description` - 5-10 word description

**Key features:**

- Persistent session state
- 2-minute default timeout (10-minute max)
- Output truncated at 30,000 chars
- No interactive commands

### 9. Task

**Purpose:** Spawn sub-agent for complex operations

**Parameters:**

- `description` (required) - Short task description
- `prompt` (required) - Detailed instructions

**Key features:**

- Autonomous execution
- Full tool access
- One-shot operation (stateless)
- Great for open-ended searches

---

## 🌐 Web Integration

### 10. WebFetch

**Purpose:** Fetch and AI-process web content

**Parameters:**

- `url` (required) - URL to fetch
- `prompt` (required) - Processing instruction

**Key features:**

- Converts HTML to markdown
- Processes with AI to extract info
- 15-minute cache
- Auto-upgrades HTTP to HTTPS

### 11. WebSearch

**Purpose:** Search the internet

**Parameters:**

- `query` (required) - Search query
- `allowed_domains` - Domain whitelist
- `blocked_domains` - Domain blacklist

**Key features:**

- Returns search snippets
- Domain filtering options
- Access to current information
- US-only availability

---

## ✅ Task Management

### 12. TodoWrite

**Purpose:** Create and update task lists

**Parameters:**

- `todos` (required) - Array of todo items:
  - `id` - Unique identifier
  - `content` - Task description
  - `status` - pending/in_progress/completed
  - `priority` - high/medium/low

**Key features:**

- One task in_progress at a time
- Used automatically for 3+ step tasks
- Persists across conversation

### 13. TodoRead

**Purpose:** View current task list

**Parameters:** None

**Key features:**

- Returns complete task state
- Called frequently for context
- Empty array if no todos

---

## 📓 Jupyter Notebook Support

### 14. NotebookRead

**Purpose:** Read Jupyter notebook contents

**Parameters:**

- `notebook_path` (required) - Absolute path to .ipynb

**Key features:**

- Returns all cells with outputs
- Preserves cell types and metadata
- Maintains execution order

### 15. NotebookEdit

**Purpose:** Modify notebook cells

**Parameters:**

- `notebook_path` (required) - Absolute path to .ipynb
- `cell_number` (required) - 0-based cell index
- `new_source` (required) - New cell content
- `cell_type` - 'code' or 'markdown' (for insert mode)
- `edit_mode` - 'replace'/'insert'/'delete'

**Key features:**

- Replace, insert, or delete cells
- 0-based cell indexing
- Preserves notebook integrity

---

## Official Resources

- **[Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)** - Official documentation and getting started guide
- **[CLI Usage Guide](https://docs.anthropic.com/en/docs/claude-code/cli-usage)** - Command line interface reference
- **[Settings & Configuration](https://docs.anthropic.com/en/docs/claude-code/settings)** - Customization options
- **[Security & Permissions](https://docs.anthropic.com/en/docs/claude-code/security)** - Understanding tool permissions
- **[Tutorials](https://docs.anthropic.com/en/docs/claude-code/tutorials)** - Step-by-step guides and workflows
- **[GitHub Repository](https://github.com/anthropics/claude-code)** - Report issues and contribute
