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

This tech stack choice is brilliant in its simplicity. By using React with Ink, the team could leverage familiar web development patterns for building a terminal interface. The TypeScript foundation ensures reliability and maintainability. And the standard Node.js runtime means it works everywhere developers need it.

The result is a tool that feels sophisticated but is built on a foundation any JavaScript developer would find familiar. While Claude Code isn't open source yet, understanding its tech stack helps demystify how it works.

## What Actually Happens When You Type in Claude Code?

When you fire up Claude Code and ask it to refactor a function or debug an issue, you're not just chatting with an AI—you're activating a sophisticated toolkit designed specifically for software engineering. While our [previous article](./loop.md) explained the agent loop pattern that powers all AI coding assistants, today we're diving deep into **Claude Code itself**: what tools it uses, how they work together, and why certain design decisions were made.

If you've ever wondered why Claude Code feels different from other AI assistants, or why it behaves in specific ways, this article will reveal the engineering decisions behind those behaviors.

## The 15 Tools That Power Claude Code

Claude Code operates with just 15 core tools, organized into logical groups:

**File Operations** (Read, Write, Edit, MultiEdit) - The basics of any coding task. Read files to understand code, Write to create new ones, Edit for precise changes, and MultiEdit for batch refactoring.

**Search & Navigation** (Glob, Grep, LS) - Find files by pattern, search content with regex, and explore directory structures. These tools are optimized for developer workflows, like sorting results by modification time.

**Execution & Automation** (Bash, Task) - Run shell commands and delegate complex operations. The Task tool is particularly powerful—it spawns sub-agents to handle extensive searches without cluttering your main conversation.

**Web Integration** (WebFetch, WebSearch) - Access documentation and current information. WebFetch is clever—it processes web content with AI to extract what you need.

**Task Management** (TodoWrite, TodoRead) - Built-in planning that transforms Claude Code from reactive to proactive. It creates task lists for complex operations and tracks progress systematically.

**Jupyter Support** (NotebookRead, NotebookEdit) - Specialized tools for data science workflows that understand notebook structure.

**MCP Extensions** - The extensibility layer. Any MCP server tool becomes a first-class citizen, like `mcp__github__createPR`, working seamlessly within the agent loop.

## The Complete Tool Reference: 15 Tools Explained

Claude Code's power comes from its carefully designed toolkit. Each tool serves a specific purpose in the software development workflow. Let's explore each one in detail.

---

## 📁 File Operations

These four tools form the core of Claude Code's file manipulation capabilities. They're designed to be safe, predictable, and traceable.

### 1. Read - File Content Viewer

```typescript
interface Read {
  file_path: string; // Absolute path to file (required)
  offset?: number; // Line number to start reading from
  limit?: number; // Number of lines to read (default: 2000)
}
```

**What it does:** Reads file contents and returns them with line numbers in `cat -n` format.

**How it works:**

- Opens the specified file and reads its contents
- Applies line numbering starting from 1
- Supports pagination for large files via offset/limit
- Handles various encodings (UTF-8 by default)
- Returns an error for binary files or if file doesn't exist

**Key behaviors:**

- Maximum 2000 lines by default to prevent token overflow
- Lines longer than 2000 characters are truncated
- Can read images and display them visually (PNG, JPG, etc.)
- Required before editing existing files

**Example usage:**

```typescript
// Read entire file (up to 2000 lines)
Read("/Users/dev/project/src/index.js")

// Read lines 100-150 of a large file
Read("/Users/dev/project/large-log.txt", offset: 100, limit: 50)
```

### 2. Write - File Creator/Overwriter

```typescript
interface Write {
  file_path: string; // Absolute path to file (required)
  content: string; // Complete file content (required)
}
```

**What it does:** Creates new files or completely overwrites existing files.

**How it works:**

- Creates directories if they don't exist
- Writes content atomically to prevent corruption
- For existing files, requires prior Read to ensure awareness

**Key behaviors:**

- Complete replacement - no merging or appending
- Must read existing files first (safety mechanism)
- Can create files in non-existent directories
- No size limits but subject to system constraints

**Example usage:**

```typescript
// Create new file
Write('/Users/dev/project/new-component.js', 'export const Component = () => {...}');

// Overwrite existing (after reading)
Write('/Users/dev/project/config.json', updatedConfigContent);
```

### 3. Edit - Precision String Replacer

```typescript
interface Edit {
  file_path: string; // Absolute path to file (required)
  old_string: string; // Exact string to replace (required)
  new_string: string; // Replacement string (required)
  replace_all?: boolean; // Replace all occurrences (default: false)
}
```

**What it does:** Performs exact string replacements within files.

**How it works:**

- Searches for exact match of old_string
- Replaces with new_string
- Preserves all formatting and whitespace
- Maintains file encoding

**Key behaviors:**

- Fails if old_string is not unique (unless replace_all=true)
- Must match exactly including whitespace
- Cannot use regex patterns
- Requires prior file read

**Example usage:**

```typescript
// Single replacement
Edit("/Users/dev/file.js", "const OLD_NAME = 5", "const NEW_NAME = 5")

// Replace all occurrences
Edit("/Users/dev/file.js", "console.log", "logger.debug", replace_all: true)
```

### 4. MultiEdit - Batch Editor

```typescript
interface MultiEdit {
  file_path: string; // Absolute path to file (required)
  edits: Array<{
    // Sequential edits to apply (required)
    old_string: string;
    new_string: string;
    replace_all?: boolean;
  }>;
}
```

**What it does:** Applies multiple edits to a single file in one atomic operation.

**How it works:**

- Applies edits sequentially in order provided
- Each edit operates on the result of previous edits
- All succeed or all fail (atomic operation)

**Key behaviors:**

- More efficient than multiple Edit calls
- Order matters - plan carefully
- Same restrictions as Edit for each operation
- Great for refactoring

**Example usage:**

```typescript
MultiEdit('/Users/dev/api.js', [
  { old_string: 'getUserById', new_string: 'findUserById', replace_all: true },
  { old_string: '// TODO: Add error handling', new_string: '// Error handling added' },
  { old_string: "status: 'active'", new_string: "status: 'ACTIVE'" },
]);
```

---

## 🔍 Search and Navigation

These tools help you explore and search through codebases efficiently, from finding files to searching content.

### 5. Glob - Pattern-Based File Finder

```typescript
interface Glob {
  pattern: string; // Glob pattern to match (required)
  path?: string; // Directory to search in (default: cwd)
}
```

**What it does:** Finds files matching glob patterns, sorted by modification time.

**How it works:**

- Uses standard glob syntax (\*, \*\*, ?, [])
- Searches recursively when using \*\*
- Returns paths relative to search directory
- Sorts by modification time (newest first)

**Key behaviors:**

- Respects .gitignore patterns
- Case-sensitive by default
- Returns empty array if no matches
- Very fast even on large codebases

**Example patterns:**

```typescript
// Find all TypeScript files
Glob("**/*.ts")

// Find test files in specific directory
Glob("**/*.test.js", path: "/Users/dev/project/src")

// Find all JSON configs
Glob("**/config*.json")
```

### 6. Grep - Content Search Engine

```typescript
interface Grep {
  pattern: string; // Regex pattern to search (required)
  path?: string; // Directory to search in (default: cwd)
  include?: string; // File pattern filter (e.g., "*.js")
}
```

**What it does:** Searches file contents using regular expressions.

**How it works:**

- Full regex support (PCRE syntax)
- Searches file contents, not names
- Returns list of files containing matches
- Can filter by file type

**Key behaviors:**

- Returns file paths, not match contents
- Case-sensitive by default
- Skips binary files
- Use ripgrep (`rg`) in Bash for detailed matches

**Example usage:**

```typescript
// Find files containing function definitions
Grep("function\\s+\\w+\\s*\\(")

// Search only in JavaScript files
Grep("TODO|FIXME", include: "*.js")

// Find class definitions
Grep("class\\s+[A-Z]\\w*\\s*(extends|{)")
```

### 7. LS - Directory Lister

```typescript
interface LS {
  path: string; // Absolute directory path (required)
  ignore?: string[]; // Glob patterns to ignore
}
```

**What it does:** Lists files and directories with optional filtering.

**How it works:**

- Returns immediate children only (not recursive)
- Distinguishes files from directories
- Applies ignore patterns if provided
- Formats output hierarchically

**Key behaviors:**

- Requires absolute paths
- Shows hidden files (starting with .)
- Returns error if path doesn't exist
- Useful for exploring project structure

**Example usage:**

```typescript
// List project root
LS("/Users/dev/project")

// List source directory, ignoring tests
LS("/Users/dev/project/src", ignore: ["*.test.js", "*.spec.js"])
```

---

## ⚡ Execution and Automation

These powerful tools enable Claude Code to run commands and delegate complex tasks to sub-agents.

### 8. Bash - Command Executor

```typescript
interface Bash {
  command: string; // Shell command to execute (required)
  timeout?: number; // Timeout in ms (max: 600000)
  description?: string; // 5-10 word description
}
```

**What it does:** Executes shell commands in a persistent session.

**How it works:**

- Maintains shell state across invocations
- Captures stdout and stderr
- Enforces timeout limits
- Preserves working directory

**Key behaviors:**

- No interactive commands allowed
- Output truncated at 30,000 characters
- 2-minute default timeout, 10-minute max
- Use specialized tools instead of file commands

**Example usage:**

```typescript
// Run tests
Bash("npm test", description: "Run project test suite")

// Check git status
Bash("git status --porcelain", description: "Check git working tree")

// Build project
Bash("npm run build", timeout: 300000, description: "Build production bundle")
```

### 9. Task - Sub-Agent Spawner

```typescript
interface Task {
  description: string; // Short task description (required)
  prompt: string; // Detailed instructions (required)
}
```

**What it does:** Launches an autonomous sub-agent for complex tasks.

**How it works:**

- Creates new agent with full tool access
- Executes entire task independently
- Returns consolidated results
- No back-and-forth communication

**Key behaviors:**

- Stateless - one-shot execution
- Perfect for open-ended searches
- Reduces main session clutter
- Same tool access as parent

**Example usage:**

```typescript
Task(
  description: "Find all API endpoints",
  prompt: "Search the codebase for all REST API endpoint definitions. Include the HTTP method, path, and which file contains each endpoint."
)
```

---

## 🌐 Web Integration

Claude Code can reach beyond your local filesystem to fetch and search web content.

### 10. WebFetch - Intelligent Web Scraper

```typescript
interface WebFetch {
  url: string; // URL to fetch (required)
  prompt: string; // Processing instruction (required)
}
```

**What it does:** Fetches web content and processes it with AI.

**How it works:**

- Downloads web page content
- Converts HTML to markdown
- Processes with smaller AI model
- Returns extracted information

**Key behaviors:**

- 15-minute result cache
- Auto-upgrades HTTP to HTTPS
- Handles cookies and redirects
- Read-only operation

**Example usage:**

```typescript
WebFetch(
  url: "https://docs.python.org/3/library/asyncio.html",
  prompt: "Extract the main asyncio concepts and code examples"
)
```

### 11. WebSearch - Internet Search Interface

```typescript
interface WebSearch {
  query: string; // Search query (required)
  allowed_domains?: string[]; // Domain whitelist
  blocked_domains?: string[]; // Domain blacklist
}
```

**What it does:** Searches the internet for current information.

**How it works:**

- Performs web search
- Returns formatted results
- Can filter by domain
- US-only availability

**Key behaviors:**

- Returns search result snippets
- Recent information beyond training cutoff
- Domain filtering for focused searches
- No direct page fetching

**Example usage:**

```typescript
// General search
WebSearch("React 19 new features")

// Search specific sites
WebSearch("typescript generics", allowed_domains: ["stackoverflow.com", "github.com"])
```

---

## ✅ Task Management

Built-in task tracking ensures complex operations are completed systematically.

### 12. TodoWrite - Task List Manager

```typescript
interface TodoWrite {
  todos: Array<{
    // Todo list items (required)
    id: string;
    content: string;
    status: 'pending' | 'in_progress' | 'completed';
    priority: 'high' | 'medium' | 'low';
  }>;
}
```

**What it does:** Creates and updates a structured task list.

**How it works:**

- Maintains session-specific todo list
- Tracks status and priority
- Enables systematic task completion
- Persists across conversation

**Key behaviors:**

- Used for tasks with 3+ steps
- Only one task in_progress at a time
- IDs should be unique
- Automatic for complex requests

**Example usage:**

```typescript
TodoWrite([
  { id: '1', content: 'Set up test environment', status: 'completed', priority: 'high' },
  { id: '2', content: 'Write unit tests', status: 'in_progress', priority: 'high' },
  { id: '3', content: 'Add integration tests', status: 'pending', priority: 'medium' },
]);
```

### 13. TodoRead - Task List Viewer

```typescript
interface TodoRead {
  // No parameters - returns current todo list
}
```

**What it does:** Retrieves the current task list.

**How it works:**

- Returns all todos with current state
- No filtering or parameters
- Shows complete task context

**Key behaviors:**

- Called frequently by Claude Code
- Empty array if no todos
- Helps maintain task awareness

---

## 📓 Jupyter Notebook Support

Specialized tools for data science and notebook workflows.

### 14. NotebookRead - Jupyter Notebook Viewer

```typescript
interface NotebookRead {
  notebook_path: string; // Absolute path to .ipynb (required)
}
```

**What it does:** Reads Jupyter notebook structure and contents.

**How it works:**

- Parses .ipynb JSON format
- Returns all cells with outputs
- Preserves cell types and metadata
- Maintains execution order

**Key behaviors:**

- Shows code and markdown cells
- Includes cell outputs
- Preserves notebook structure
- Large outputs may be truncated

### 15. NotebookEdit - Jupyter Notebook Editor

```typescript
interface NotebookEdit {
  notebook_path: string; // Absolute path to .ipynb (required)
  cell_number: number; // 0-based cell index (required)
  new_source: string; // New cell content (required)
  cell_type?: 'code' | 'markdown'; // For insert mode
  edit_mode?: 'replace' | 'insert' | 'delete';
}
```

**What it does:** Modifies Jupyter notebook cells.

**How it works:**

- Can replace, insert, or delete cells
- Preserves notebook integrity
- Updates cell content or type
- Maintains notebook metadata

**Key behaviors:**

- 0-based indexing for cells
- Insert adds at specified index
- Delete removes cell entirely
- Type required for insert mode

**Example usage:**

```typescript
// Replace cell content
NotebookEdit("/path/notebook.ipynb", cell_number: 2, new_source: "import pandas as pd")

// Insert new cell
NotebookEdit("/path/notebook.ipynb", cell_number: 1, new_source: "# Data Analysis",
            cell_type: "markdown", edit_mode: "insert")
```

---

## 🔌 MCP Integration: The Extensibility Layer

### Understanding Model Context Protocol (MCP)

The Model Context Protocol (MCP) is Claude Code's extensibility framework that allows third-party tools to integrate seamlessly into agent's workflow.

### Architectural Design: First-Class Tool Citizens

**The Key Differentiator:** While many AI coding assistants treat MCP as an add-on layer requiring special handling (like `mcp_invoke("tool_name", params)`), Claude Code's architecture treats every MCP tool as a native tool. This means:

```typescript
// Other assistants might require:
mcp_invoke('github', 'createPR', { title: 'Fix bug', body: '...' });

// In Claude Code, it's just:
mcp__github__createPR({ title: 'Fix bug', body: '...' });
```

This architectural decision has profound implications:

1. **Unified Tool Discovery**: It sees all tools—built-in and MCP—in a single registry
2. **Consistent Error Handling**: MCP tool failures are handled identically to built-in tool failures
3. **Natural Integration**: It can use MCP tools in the same agent loops and patterns as native tools

### Common MCP Integrations

#### GitHub Integration

```typescript
// These tools appear when GitHub MCP server is connected:
mcp__github__createPR({
  title: string,
  body: string,
  base?: string,
  head?: string
})
mcp__github__listIssues({
  state?: 'open' | 'closed' | 'all',
  labels?: string[]
})
mcp__github__mergePR({
  number: number,
  merge_method?: 'merge' | 'squash' | 'rebase'
})
```

#### Database Integration

```typescript
// Example database MCP tools:
mcp__postgres__query({ sql: string, params?: any[] })
mcp__postgres__listTables({ schema?: string })
mcp__postgres__describeTable({ table: string })
```

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

The WebSocket streaming, session persistence, parallel execution, and extensible tool registry all work together to create a system that feels responsive and capable while maintaining the safety necessary for production code manipulation.

Whether you're using Claude Code for quick fixes or complex refactoring, understanding its architecture helps you work with the system rather than against it. The next time you see Claude Code spawn a sub-agent for a search or insist on reading before writing, you'll know these aren't quirks—they're features designed to make your coding experience both powerful and safe.

## Official Resources

- **[Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)** - Official documentation and getting started guide
- **[CLI Usage Guide](https://docs.anthropic.com/en/docs/claude-code/cli-usage)** - Command line interface reference
- **[Settings & Configuration](https://docs.anthropic.com/en/docs/claude-code/settings)** - Customization options
- **[Security & Permissions](https://docs.anthropic.com/en/docs/claude-code/security)** - Understanding tool permissions
- **[Tutorials](https://docs.anthropic.com/en/docs/claude-code/tutorials)** - Step-by-step guides and workflows
- **[GitHub Repository](https://github.com/anthropics/claude-code)** - Report issues and contribute
