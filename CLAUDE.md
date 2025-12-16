# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Latest Updates (v0.3.1)
- **NEW:** Added `settingSources` parameter to control MCP server and settings loading
  - Enables loading user-level MCP servers from `~/.claude.json`
  - Supports project-level and local settings
  - Default: loads all sources (`user`, `project`, `local`)
- Updated to claude-agent-sdk ^0.1.70
- Added support for Claude 4-5 models:
  - Sonnet 4.5 (recommended for most tasks)
  - Opus 4.5 (most capable for complex tasks)
  - Haiku 4 (fastest for simple tasks)

## Common Commands

### Development
- `npm run dev` - Run TypeScript compiler in watch mode for development
- `npm run build` - Build the project (clean dist, compile TypeScript, copy icons)
- `npm run format` - Format code using Prettier
- `npm run lint` - Run ESLint to check code quality
- `npm run lintfix` - Auto-fix linting issues where possible

### n8n Integration
- Install locally: `npm link` then `n8n start` to test the node
- The node appears in n8n UI under "Claude Code" category
- Debug output available when Debug option is enabled in node parameters

## Architecture Overview

This is an n8n community node that integrates Claude Code SDK into n8n workflows. The architecture consists of:

1. **Main Node Implementation** (`nodes/ClaudeCode/ClaudeCode.node.ts`)
   - Implements `INodeType` interface from n8n
   - Provides Query and Continue operations
   - Handles Claude Code SDK initialization and message processing
   - Manages tool availability and project path configuration

2. **Tool System**
   - Dynamic tool enabling/disabling based on user configuration
   - Supports: Bash, Edit/MultiEdit, Read/Write, Web operations, Todo management
   - MCP servers supported via `settingSources` parameter:
     - **User-level**: Loads MCP servers from `~/.claude.json` and settings from `~/.claude/settings.json`
     - **Project-level**: Loads from `.mcp.json` and `.claude/settings.json` in project directory
     - **Local**: Loads from `.claude/settings.local.json` (gitignored personal settings)

3. **Output Handling**
   - Multiple output formats: structured JSON, messages array, or plain text
   - Streaming support with abort signal handling
   - Debug mode for troubleshooting

4. **Project Path Support** (v0.2.0+)
   - Configure working directory via `projectPath` parameter
   - Allows Claude Code to run in specific project directories
   - Enables access to code repositories without changing n8n's working directory

## Key Development Patterns

### n8n Node Structure
- All node logic resides in `ClaudeCode.node.ts`
- Parameters defined using n8n's declarative schema
- Error handling follows n8n patterns with `NodeOperationError`
- Supports both single execution and streaming responses

### TypeScript Configuration
- Strict mode enabled for type safety
- Target ES2019 with CommonJS modules (n8n requirement)
- Source maps generated for debugging
- Output to `dist/` directory

### Code Style
- Uses tabs with width 2 (n8n standard)
- Single quotes for strings
- Semicolons required
- Maximum line width: 100 characters
- ESLint configured with n8n-nodes-base rules


## Testing Approach

No automated tests are configured (typical for n8n community nodes). Testing involves:
1. Building the node: `npm run build`
2. Linking locally: `npm link`
3. Starting n8n: `n8n start`
4. Creating test workflows with various parameter combinations
5. Using Debug mode to inspect Claude Code interactions

## Configuration Examples

The `examples/` directory contains sample configurations:
- **simple-project/**: Basic setup without MCP servers
- **project-with-mcp/**: Full MCP server configuration example

Key configuration files:
- `.mcp.json`: Defines available MCP servers (project root)
- `.claude/settings.json`: Team-shared settings
- `.claude/settings.local.json`: Personal settings (gitignored)
- `~/.claude.json`: User-level MCP servers and global settings

### MCP Server Loading via settingSources

The `settingSources` parameter controls which configuration sources the SDK loads:

- **`['user']`**: Loads only user-level configs (`~/.claude.json`, `~/.claude/settings.json`)
- **`['project']`**: Loads only project configs (`.mcp.json`, `.claude/settings.json`)
- **`['local']`**: Loads only local configs (`.claude/settings.local.json`)
- **`['user', 'project', 'local']`**: Default - loads all configs (recommended)
- **`[]`**: Isolation mode - no automatic config loading

**Example:** To use your user-level MCP servers from `~/.claude.json`:
1. Set `settingSources` to include `'user'` (enabled by default)
2. The SDK will automatically load all MCP servers defined in `~/.claude.json`
3. Use MCP tools via the `allowedTools` parameter (e.g., `mcp__filesystem__list_files`)

When using Project Path, Claude Code automatically loads these configurations from the specified directory.