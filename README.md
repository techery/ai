# AI Coding Assistant Engineering

A resource hub for understanding modern AI coding assistants. This repository contains in-depth technical articles that demystify how AI coding tools really work, with practical examples and implementation details.

## 📚 Articles

### [Understanding Agent Loops: How AI Assistants Use Tools to Solve Complex Problems](./articles/loop.md)

Discover the elegant architecture behind every modern AI coding assistant. This detailed guide explores:

- **The Simple Pattern**: How Claude Code, Cursor, Windsurf, and other AI assistants all use the same fundamental approach
- **Agent Loop Architecture**: The Think → Act → Observe → Repeat cycle that powers intelligent automation
- **Tool Integration**: How AI systems use file operations, shell commands, and external services to accomplish real work
- **Complete Implementation**: A working TypeScript example using Anthropic's SDK showing exactly how to build your own agent

**Key Takeaways:**
- No magic involved—just clever engineering around giving AI the ability to use tools
- Complete code examples demonstrating the core patterns
- Understanding that empowers you to use existing tools more effectively and build your own

### [Inside Claude Code: The Tools and Architecture That Power Your AI Coding Assistant](./articles/claude-code-tools.md)

A deep dive into Claude Code's specific implementation, revealing the engineering decisions that shape its behavior:

- **The Agent Loop**: How Claude Code's implementation differs from other AI assistants through explicit task planning and nested agent architecture
- **The Toolkit**: Detailed exploration of all 15 tools, from file operations to web search
- **Architectural Patterns**: Understanding TodoWrite/TodoRead for task planning and the Task agent system for complex operations
- **Practical Usage**: How to work more effectively with Claude Code based on its architecture

**Key Takeaways:**
- Two key features that make Claude Code unique: mandatory task planning with TodoWrite and the nested Task agent architecture
- Claude Code's 15 specialized tools and when to use each one
- Why certain operations are restricted or require specific approaches
- How to leverage Claude Code's architectural patterns for maximum productivity

## 🎯 What You'll Learn

This series focuses on practical understanding rather than theoretical concepts:

- **Architecture Patterns**: The fundamental designs that make AI coding assistants work
- **Implementation Details**: Real code examples and working implementations
- **Tool Composition**: How simple tools combine to create powerful coding capabilities
- **Design Trade-offs**: Understanding the decisions behind different AI coding assistant approaches

## 🚀 For Software Engineers

Whether you're:

- **Using** AI coding assistants in your daily work
- **Building** AI-powered coding tools for your team
- **Curious** about how modern AI coding assistants actually work
- **Evaluating** different AI coding solutions

These materials provide the technical depth to understand, evaluate, and leverage AI coding tools effectively.

## 🔧 Practical Applications

Learn how to:

- Use AI coding assistants more effectively by understanding their capabilities and limitations
- Debug issues when AI tools don't work as expected
- Build custom agents for specialized workflows
- Evaluate different AI coding solutions based on their architectural approaches

## 📖 About This Series

This series examines various implementations including Claude Code, Cursor, Aider, Windsurf, and other coding assistants. We focus on understanding the patterns and concepts that make these tools work, using real examples and open-source implementations to look under the hood.

The beauty is that whether you're using Claude Code, Cursor, Windsurf, or any other coding assistant, **they all follow the same fundamental patterns**.

---

_A curated collection of practical insights about AI coding assistant engineering and modern development tools._
