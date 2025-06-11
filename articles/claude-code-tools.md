# Inside Claude Code: The Tools and Architecture That Power Your AI Coding Assistant

I've been using Claude Code since its first release, watching it evolve from an experimental tool into something that's progressively transforming how we write software at Techery. We're in the process of making it our main coding agent, with more developers adopting it every week. After months of intensive use, I want to share what I've learned about how Claude Code actually works under the hood.

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

## What Actually Happens When You Type in Claude Code?

When you fire up Claude Code and ask it to refactor a function or debug an issue, you're not just chatting with an AI—you're activating a sophisticated toolkit designed specifically for software engineering. While our [previous article](./loop.md) explained the agent loop pattern that powers all AI coding assistants, today we're diving deep into Claude Code's specific implementation.

If you've ever wondered why Claude Code feels different from other AI assistants, or why it behaves in specific ways, the answer lies in its toolkit. Let's explore the 15 tools that power every interaction.

## The Tools That Power Claude Code

What sets Claude Code apart isn't sophisticated AI magic—it's a thoughtfully designed set of tools that map directly to what developers actually do. These tools can be organized into logical groups that mirror typical development workflows:

**File Operations** (Read, Write, Edit, MultiEdit) - The foundation of any coding task. These tools handle everything from reading existing code to making surgical edits or batch refactoring.

**Search & Navigation** (Glob, Grep, LS) - Your exploration toolkit. Find files by pattern, search content with regex, and navigate directory structures—all optimized for speed even in massive codebases.

**Execution & Automation** (Bash, Task) - The power tools. Run any shell command or spawn autonomous sub-agents for complex operations. The Task tool is particularly clever—it delegates open-ended searches to keep your main conversation clean.

**Web Integration** (WebFetch, WebSearch) - Your window to the internet. Access documentation, search for current information, and process web content with AI to extract exactly what you need.

**Task Management** (TodoWrite, TodoRead) - The planning layer that makes Claude Code proactive rather than reactive. It automatically creates task lists for complex operations and tracks progress systematically.

**Jupyter Support** (NotebookRead, NotebookEdit) - Specialized tools for data science that understand notebook cell structure and outputs.

**MCP Extensions** - The extensibility layer where any third-party tool becomes a first-class citizen, seamlessly integrated into the agent loop.

## Architectural Patterns

### Task Planning with TodoWrite/TodoRead

One of Claude Code's most distinctive architectural patterns is its built-in task planning system. Unlike other AI coding assistants that might lose track of complex operations, Claude Code uses TodoWrite and TodoRead to maintain systematic progress.

**Important:** Claude Code is specifically instructed in its system prompt to call TodoWrite very frequently throughout conversations. This isn't optional behavior—it's a core requirement that ensures:

- Every multi-step task gets planned before execution
- Progress is tracked in real-time as work proceeds
- Users always have visibility into what's happening
- The assistant maintains context even in long conversations

**The Planning Pattern:**

```typescript
// 1. User requests complex task
'Refactor all API endpoints to use async/await';

// 2. Claude Code immediately creates a plan
TodoWrite([
  { id: '1', content: 'Find all API endpoint files', status: 'pending', priority: 'high' },
  { id: '2', content: 'Analyze current promise patterns', status: 'pending', priority: 'high' },
  { id: '3', content: 'Convert to async/await syntax', status: 'pending', priority: 'high' },
  { id: '4', content: 'Update error handling', status: 'pending', priority: 'medium' },
  { id: '5', content: 'Run tests to verify', status: 'pending', priority: 'high' },
]);

// 3. Systematic execution with status updates
TodoWrite([...todos, { id: '1', status: 'in_progress' }]);
// Execute: Grep("(app|router)\\.(get|post|put|delete)\\(")
TodoWrite([...todos, { id: '1', status: 'completed' }]);

// 4. Claude Code regularly checks progress
TodoRead(); // Returns current state for context awareness
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

**Complex Example - Analyzing Technical Debt:**

```typescript
Task(
  description: "Analyze technical debt in codebase",
  prompt: `Analyze the entire codebase for technical debt indicators:

  1. Find all TODO, FIXME, HACK, and XXX comments
  2. Identify files with cyclomatic complexity > 10
  3. Look for deprecated API usage patterns
  4. Find duplicated code blocks (>20 lines similar)
  5. Check for outdated dependencies in package files
  6. Identify large files (>500 lines) that might need splitting
  7. Find deeply nested code (>4 levels of indentation)

  For each issue found:
  - Note the file path and line numbers
  - Categorize by severity (high/medium/low)
  - Estimate effort to fix (hours)

  Return a structured report with:
  - Executive summary of technical debt level
  - Detailed findings by category
  - Prioritized list of refactoring recommendations
  - Total estimated hours to address all issues`
)
```

**The sub-agent would then:**

1. Use `Grep` to find all code quality markers
2. Use `Glob` to find all source files
3. Use `Read` to analyze file contents
4. Use `Bash` to run complexity analysis tools
5. Process and correlate all findings
6. Generate a comprehensive report

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
