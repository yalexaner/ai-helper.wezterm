# WezTerm AI Plugin Cursor Positioning Journey

## Problem Overview

This document chronicles the complete journey of solving cursor positioning issues in a WezTerm AI helper plugin, originally designed for macOS/Linux but being adapted for Windows PowerShell.

### Original Plugin Behavior
- User presses keybinding (Super+I)
- AI prompt input appears
- AI processes request and displays response
- User should be able to continue typing at correct cursor position

### Core Technical Challenge
The plugin uses `inject_output()` to display AI responses, which bypasses PowerShell's PSReadLine module. This creates a **state desynchronization** between:
- **Terminal's visual state** (what user sees)
- **Shell's internal state** (where shell thinks cursor should be)

## Original Issues (Windows-Specific)

### Issue 1: "^U" Symbol and Command Errors
**Symptoms:**
- `^U` symbol appeared at end of responses
- PowerShell error: "The term '' is not recognized as a cmdlet..."

**Root Cause:**
- Plugin used Unix-specific `Ctrl+U` (`\u{15}`) for line clearing
- PowerShell doesn't handle `Ctrl+U` the same way as bash/zsh
- Empty command execution caused cmdlet errors

**Fix Applied:**
- Added OS detection: `is_windows()` function
- OS-specific line clearing:
  - Windows: ANSI escape sequences `\x1b[2K\x1b[0G`
  - Unix: Keep `Ctrl+U` behavior
- OS-specific system prompts and default paths

## Solution Attempts Chronicle

### Attempt 1: Replace inject_output with send_text
**Date Context:** Early in session
**Approach:** Use `send_text()` instead of `inject_output()` for all messages
**Implementation:**
```lua
-- Instead of: pane:inject_output("🤖 AI is thinking...")
pane:send_text("🤖 AI is thinking...\r\n")
```

**Outcome:** ❌ **FAILED**
**Failure Reason:** PowerShell tried to execute emoji characters as commands
**Error Example:**
```
🤖: The term '🤖' is not recognized as a name of a cmdlet, function, script file, or executable program.
```

**Learning:** `send_text()` simulates user typing, so shell interprets content as commands.

### Attempt 2: Use Shell Commands (Write-Host/echo)
**Approach:** Wrap display text in shell-specific commands
**Implementation:**
```lua
if is_windows() then
    pane:send_text("Write-Host '🤖 AI is thinking...'; ")
else
    pane:send_text("echo '🤖 AI is thinking...'; ")
end
```

**Outcome:** ❌ **FAILED**
**Failure Reason:** Poor UX - users saw actual commands being typed before execution
**User Experience:**
```
Write-Host '🤖 AI is thinking...'; Write-Host '💬 Response';
🤖 AI is thinking...
💬 Response
```

**Learning:** Visible command execution creates terrible user experience.

### Attempt 3: inject_output + Relative Cursor Positioning
**Approach:** Use `inject_output()` for display + `\r\n` + `\x1b[0J` + `\r` for positioning
**Implementation:**
```lua
local function scroll_and_clear(pane)
    pane:send_text("\r\n")        -- move to clean line below content
    pane:send_text("\x1b[0J")     -- clear from cursor to end of screen
    pane:send_text("\r")          -- let shell redraw prompt
end
```

**Outcome:** ❌ **FAILED**
**Failure Reasons:**
- Created duplicate prompts
- Multi-line positioning issues
- PowerShell continuation prompts (`>>`)

**Learning:** Relative positioning doesn't work with unknown starting position.

### Attempt 4: inject_output + Absolute Cursor Positioning
**Approach:** Use absolute positioning to move cursor to known location
**Implementation:**
```lua
local function scroll_and_clear(pane)
    pane:send_text("\x1b[999;1H")  -- move to bottom of screen
    pane:send_text("\x1b[0J")      -- clear from cursor to end of screen
    pane:send_text("\r")           -- let shell redraw prompt
end
```

**Outcome:** ❌ **FAILED**
**Failure Reason:** Initial positioning worked, but cursor jumped after first keypress
**Symptoms:**
- Input line appeared correctly at bottom
- First character appeared correctly
- **After first keypress, cursor jumped to "AI is thinking" line**

**Learning:** Visual positioning works, but PowerShell's PSReadLine module has separate internal state.

### Attempt 5: inject_output + Absolute Positioning + Ctrl+C
**Approach:** Use `Ctrl+C` to force PowerShell state reset
**Implementation:**
```lua
local function scroll_and_clear(pane)
    pane:send_text("\x1b[999;1H")  -- position cursor
    pane:send_text("\x1b[0J")      -- clear below
    pane:send_text("\x03")         -- Ctrl+C to reset shell state
end
```

**Outcome:** ❌ **FAILED**
**Failure Reason:** Ctrl+C interrupted user input session
**Symptoms:**
- `^C` appeared in output
- Cursor still jumped to wrong location
- User input session was cancelled, not just state reset

**Learning:** Can't easily fix PSReadLine internal state from outside.

## The Breakthrough: Alternate Screen Buffer Approach

### Gemini's Strategic Analysis
**Key Insight:** Stop trying to fix PowerShell's state; work around it entirely.

**Root Cause Identification:** 
> "The conflict is between two different worlds:
> 1. **Terminal's World** (`inject_output`): 'Dumb' display buffer
> 2. **Shell's World** (PSReadLine): Stateful application with internal cursor tracking"

### Strategy 1: Modal Application (Recommended)
**Concept:** Use alternate screen buffer (like vim, fzf, less)
**Technical Approach:**
1. `\x1b[?1049h` - Enter alternate screen buffer
2. Display AI interaction in clean environment
3. Capture user choices (Insert/Copy/Quit)
4. `\x1b[?1049l` - Exit alternate screen, restore original prompt
5. Execute chosen action in restored shell

**Benefits:**
- ✅ Zero state conflicts (PSReadLine paused)
- ✅ Cross-platform compatibility
- ✅ Perfect shell restoration
- ✅ Rich interaction possibilities
- ✅ Familiar UX pattern (fzf-style)

## Current Implementation Status

### ✅ Implemented Functions
```lua
local function enter_ai_mode(pane)
    pane:inject_output("\x1b[?1049h")     -- Enter alternate screen
    pane:inject_output("\x1b[2J\x1b[H")   -- Clear and home cursor
end

local function exit_ai_mode(pane)
    pane:inject_output("\x1b[?1049l")     -- Exit alternate screen
end

local function display_ai_interface(pane, content, command)
    -- Display full AI interface with options
end

local function handle_ai_choice(pane, choice, command)
    -- Process user choice (I/C/Q)
end
```

### ❌ Critical Implementation Error Discovered
**Problem:** Used `send_text()` instead of `inject_output()` in alternate screen functions
**Result:** PowerShell tried to execute all display text as commands (disaster)
**Status:** **IDENTIFIED BUT NOT YET FIXED**

**Correct Implementation Required:**
```lua
-- WRONG (current):
pane:send_text("💬 " .. content)  -- Shell tries to execute this

// CORRECT (needed):
pane:inject_output("💬 " .. content)  -- Terminal displays this
```

## Critical Issues Identified by Expert Review

### 🚨 **Showstopper Issues**
1. **Getting Stuck in Alternate Screen**
   - If Lua error occurs after entering alternate screen, user trapped
   - **Solution:** Protected call (`pcall`) wrapper with guaranteed cleanup

2. **Concurrency Problems**
   - Multiple AI requests can corrupt state
   - **Solution:** Per-pane lock mechanism

3. **Content Overflow**
   - Long AI responses break UI completely
   - **Solution:** Content truncation + clipboard fallback

### ⚠️ **Important Issues**
4. **Terminal Resizing**
   - UI corrupts on window resize
   - **Solution:** Listen for resize events, re-render

5. **Input Handling**
   - Need robust keystroke capture in alternate screen
   - **Solution:** WezTerm modal keymap system

## Recommended Implementation Pattern

```lua
local active_panes = {}  -- Concurrency lock

local function safe_ai_request(window, pane, prompt, config)
    -- 1. Concurrency check
    if active_panes[pane:pane_id()] then
        return
    end
    
    -- 2. Set lock and enter alternate screen
    active_panes[pane:pane_id()] = true
    enter_ai_mode(pane)
    
    -- 3. Protected execution
    local ok, err = pcall(function()
        -- All AI logic here using inject_output()
        -- Handle responses, user input, etc.
    end)
    
    -- 4. ALWAYS cleanup (guaranteed)
    exit_ai_mode(pane)
    active_panes[pane:pane_id()] = nil
    
    -- 5. Error logging
    if not ok then
        wezterm.log_error("AI Helper error: " .. tostring(err))
    end
end
```

## Next Steps (Priority Order)

### 🔴 **Critical (Must Fix)**
1. **Fix send_text → inject_output** in all display functions
2. **Implement protected call wrapper** to prevent stuck screen
3. **Add concurrency lock** to prevent state corruption
4. **Test basic alternate screen functionality**

### 🟡 **Important (Should Fix)**
5. **Implement content truncation** for long responses
6. **Add proper user input capture** via WezTerm events
7. **Handle terminal resize** events
8. **Add comprehensive error handling**

### 🟢 **Enhancement (Nice to Have)**
9. **Implement scrollable pager** for long content
10. **Add keyboard shortcuts** beyond I/C/Q
11. **Improve visual design** of AI interface

## Technical Learnings

### Key Distinctions
- **`send_text()`** = Send to shell input (for commands)
- **`inject_output()`** = Send to terminal display (for UI)
- **Alternate screen buffer** = Separate display mode (like vim)
- **PSReadLine** = PowerShell's advanced line editor with internal state

### Design Principles
1. **Work WITH the shell, not against it**
2. **Separate display from command execution**
3. **Use established terminal patterns** (alternate screen)
4. **Always provide escape mechanisms** (protected calls)
5. **Handle the unhappy path** (errors, edge cases)

### Cross-Platform Considerations
- **ANSI escape sequences** work consistently in modern terminals
- **Shell differences** (PowerShell vs bash) mainly affect command syntax
- **WezTerm provides consistent** terminal emulation across platforms

## Conclusion

The cursor positioning problem led us through multiple failed approaches to a robust solution using alternate screen buffers. The key insight was recognizing that **fighting the shell's internal state is futile**; instead, we should **temporarily bypass it entirely**.

The alternate screen buffer approach is:
- ✅ **Technically sound** (established terminal pattern)
- ✅ **Cross-platform compatible** (standard ANSI sequences)
- ✅ **User-friendly** (familiar from fzf, vim, etc.)
- ✅ **Extensible** (rich interaction possibilities)

**Current Status:** Solution designed and partially implemented, but needs critical fixes before testing.

**Next Session Goal:** Implement the protected call pattern and fix the `send_text`/`inject_output` distinction to get a working prototype.