# MCP Server Installation Guide for Windows PowerShell

## 📋 Table of Contents
- [Overview](#overview)
- [The Problem](#the-problem)
- [Investigation Process](#investigation-process)
- [What Didn't Work](#what-didnt-work)
- [The Solution](#the-solution)
- [Working Commands](#working-commands)
- [Key Learnings](#key-learnings)
- [Troubleshooting Tips](#troubleshooting-tips)
- [Example Configurations](#example-configurations)

---

## 🔍 Overview

This guide documents the complete process of troubleshooting and solving MCP (Model Context Protocol) server installation issues on Windows PowerShell. Through trial and error, we discovered the specific requirements for Windows compatibility and developed reliable installation methods.

**Key Achievement:** Successfully installed Perplexity AI and Exa Search MCP servers that connect properly with Claude Code.

---

## ⚠️ The Problem

### Initial Issue
MCP servers were showing connection failures in Claude Code:
```
[ERROR] MCP server "perplexity" Connection failed: MCP error -32000: Connection closed
[ERROR] MCP server "exa" Connection failed: MCP error -32000: Connection closed
```

### Working vs Non-Working Servers
- ✅ **Working:** puppeteer, context7
- ❌ **Failing:** perplexity, exa

### Root Cause Discovery
The failing servers had incorrect command structure compared to working ones:

**❌ Incorrect Structure (Failing):**
```yaml
perplexity:
  Command: cmd /c npx server-perplexity-ask  # All as one command
  Args: (empty)
```

**✅ Correct Structure (Working):**
```yaml
context7:
  Command: cmd                               # cmd only
  Args: /c npx @upstash/context7-mcp        # Rest as arguments
```

---

## 🔎 Investigation Process

### Step 1: Initial Installation Attempt
```bash
# This failed with "missing required argument 'commandOrUrl'" error
claude mcp add perplexity -s user -e -- cmd /c npx server-perplexity-ask
```

### Step 2: Command Syntax Discovery
```bash
# Found correct argument order
claude mcp add perplexity "npx server-perplexity-ask" -s user -e PERPLEXITY_API_KEY=key
```

### Step 3: Environment Variable Testing
```bash
# Direct testing showed the server worked fine
$env:PERPLEXITY_API_KEY="key"; npx server-perplexity-ask --version
# Output: "Perplexity Ask MCP Server running on stdio" ✅
```

### Step 4: Structure Comparison
```bash
# Discovered the critical difference
claude mcp get context7  # Working server
claude mcp get perplexity  # Failing server
```

### Step 5: Windows PowerShell Compatibility Issue
The key insight: **Windows PowerShell requires `cmd` wrapper for `npx` commands in MCP servers.**

---

## ❌ What Didn't Work

### 1. Direct npx Command Format
```bash
# Generated incorrect structure
claude mcp add perplexity "npx server-perplexity-ask" -s user -e API_KEY=key
```
**Result:** All command in one string instead of separated cmd + args

### 2. PowerShell JSON Escaping
```bash
# PowerShell couldn't handle complex JSON escaping
claude mcp add-json perplexity '{"command": "cmd", ...}' -s user
# Error: Invalid configuration: : Invalid input
```

### 3. Complex PowerShell Syntax
```bash
# Backtick escaping didn't work reliably
claude mcp add-json perplexity "{`"command`": `"cmd`", ...}" -s user
```

### 4. File-based JSON Approach
```bash
# Even loading from file failed in PowerShell
$json = (Get-Content config.json -Raw); claude mcp add-json server $json -s user
```

---

## ✅ The Solution

### Core Discovery: cmd.exe Wrapper Required

The working solution uses `cmd.exe` to execute the `claude mcp add-json` command:

```bash
cmd /c 'claude mcp add-json SERVERNAME "JSON_CONFIG" -s user'
```

### Proper JSON Structure
```json
{
  "command": "cmd",
  "args": ["/c", "npx", "package-name"],
  "env": {
    "API_KEY": "value"
  }
}
```

### Complete Working Process

1. **Remove existing server:**
   ```bash
   claude mcp remove SERVERNAME
   ```

2. **Add with correct structure:**
   ```bash
   cmd /c 'claude mcp add-json SERVERNAME "ESCAPED_JSON" -s user'
   ```

---

## 🚀 Working Commands

### Perplexity AI Server
```bash
# Remove if exists
claude mcp remove perplexity

# Install with proper structure
cmd /c 'claude mcp add-json perplexity "{\"command\": \"cmd\", \"args\": [\"/c\", \"npx\", \"server-perplexity-ask\"], \"env\": {\"PERPLEXITY_API_KEY\": \"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\"}}" -s user'
```

### Exa Search Server
```bash
# Remove if exists
claude mcp remove exa

# Install with proper structure
cmd /c 'claude mcp add-json exa "{\"command\": \"cmd\", \"args\": [\"/c\", \"npx\", \"-y\", \"mcp-remote\", \"https://mcp.exa.ai/mcp?exaApiKey=2ff2c1ae-fd6f-4c2d-a00f-70a44a0e9480\"]}" -s user'
```

### Verification
```bash
# List all servers
claude mcp list

# Check specific server structure
claude mcp get SERVERNAME
```

---

## 💡 Key Learnings

### 1. Windows PowerShell MCP Requirements
- ✅ **Must use:** `"command": "cmd"`
- ✅ **Must separate:** Arguments in `"args": ["/c", "npx", ...]`
- ❌ **Don't use:** `"command": "cmd /c npx ..."` (all as one string)

### 2. JSON Escaping in Windows
- ✅ **Works:** `cmd /c 'claude mcp add-json ... "ESCAPED_JSON"'`
- ❌ **Fails:** Direct PowerShell JSON with complex escaping
- 🔧 **Escape pattern:** `\"` for quotes in JSON

### 3. Command Structure Pattern
```json
{
  "command": "cmd",
  "args": [
    "/c",              // Windows cmd flag
    "npx",             // Base command
    "package-name",    // Package to run
    "--flag",          // Optional flags
    "value"            // Optional values
  ],
  "env": {
    "KEY": "VALUE"     // Environment variables
  }
}
```

### 4. Environment Variable Handling
- API keys can be in `env` object OR embedded in URLs (for remote servers)
- Environment variables are properly passed to the MCP server process
- Test with: `$env:KEY="value"; npx package-name`

### 5. Debugging Process
1. Test package directly: `npx package-name --version`
2. Test with env vars: `$env:KEY="val"; npx package-name`
3. Compare working vs failing server structures: `claude mcp get SERVER`
4. Check connection logs in Claude Code debug output

---

## 🛠️ Troubleshooting Tips

### Connection Failures
1. **Check server structure:** `claude mcp get SERVERNAME`
2. **Compare with working servers:** Look for `cmd` + separated args
3. **Test package directly:** Verify the NPM package works
4. **Check API keys:** Ensure environment variables are correct

### PowerShell JSON Issues
1. **Use cmd.exe wrapper:** `cmd /c 'claude mcp add-json ...'`
2. **Escape quotes:** Replace `"` with `\"` in JSON
3. **Avoid complex PowerShell escaping:** Use the cmd approach instead

### Server Won't Start
1. **Verify package exists:** `npm info PACKAGE-NAME`
2. **Check dependencies:** Some packages need global installation
3. **Test environment:** Run server manually with env vars
4. **Review error logs:** Look for specific error messages

---

## 📚 Example Configurations

### Standard NPX Package
```json
{
  "command": "cmd",
  "args": ["/c", "npx", "@modelcontextprotocol/server-puppeteer"],
  "env": {}
}
```

### Package with Environment Variables
```json
{
  "command": "cmd",
  "args": ["/c", "npx", "server-perplexity-ask"],
  "env": {
    "PERPLEXITY_API_KEY": "your-key-here"
  }
}
```

### NPX with Flags
```json
{
  "command": "cmd",
  "args": ["/c", "npx", "-y", "package-name", "--port", "3000"],
  "env": {}
}
```

### Remote MCP Server
```json
{
  "command": "cmd",
  "args": ["/c", "npx", "-y", "mcp-remote", "https://api.example.com/mcp?key=value"],
  "env": {}
}
```

### Custom Python Script
```json
{
  "command": "cmd",
  "args": ["/c", "python", "path/to/script.py", "--arg1", "value1"],
  "env": {
    "PYTHON_PATH": "/custom/path",
    "API_KEY": "secret"
  }
}
```

---

## 🎯 Final Status

After implementing the solution, all MCP servers connect successfully:

```bash
PS> claude mcp list
puppeteer: cmd /c npx @modelcontextprotocol/server-puppeteer      ✅
context7: cmd /c npx @upstash/context7-mcp                       ✅  
perplexity: cmd /c npx server-perplexity-ask                     ✅ FIXED
exa: cmd /c npx -y mcp-remote https://mcp.exa.ai/mcp?exaApiKey=  ✅ FIXED
```

**Result:** All servers now connect without "Connection closed" errors and provide their respective tools in Claude Code.

---

## 🔗 Related Tools

This guide served as the foundation for creating an **MCP Server Installation Generator** - a modern HTML application that:

- ✅ Automatically generates proper Windows PowerShell commands
- ✅ Handles JSON escaping and cmd.exe wrapping
- ✅ Includes presets for common MCP servers
- ✅ Provides a user-friendly interface for complex configurations

The generator eliminates the need to manually handle the complex JSON escaping and Windows-specific requirements documented in this guide.

---

## 📝 Summary

**The Core Issue:** Windows PowerShell requires MCP servers to use `"command": "cmd"` with separated arguments, not combined command strings.

**The Solution:** Use `cmd /c 'claude mcp add-json ...'` with properly escaped JSON that separates `cmd` and arguments.

**Key Success Factors:**
1. Understanding Windows MCP server structure requirements
2. Using cmd.exe to bypass PowerShell JSON limitations  
3. Proper JSON escaping with `\"` for quotes
4. Testing packages directly before MCP integration
5. Comparing working vs failing server configurations

This guide provides the complete knowledge base for successfully installing and troubleshooting MCP servers on Windows PowerShell systems. 