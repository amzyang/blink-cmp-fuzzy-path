# Set Search Directory Feature Plan

## Overview
Add functionality to dynamically set the directory path where fuzzy file search should be performed. This will allow users to:
1. Programmatically change the search directory via `set_path()` function
2. Use a Neovim command `:FuzzySearchPath` to interactively set the search path

---

## Current State Analysis

Currently, the plugin searches for files using the working directory (`cwd`) as the search root:
- `file_search.lua` uses `fd` or `rg` commands without specifying a directory, defaulting to `cwd`
- No mechanism exists to constrain the search to a specific directory
- File paths are made relative to the current buffer's directory, but search is global

---

## Design Goals

1. **Flexibility**: Allow search directory to be set per-session or per-buffer
2. **Programmatic Control**: Expose a public API function for setting the path
3. **Interactive Command**: Provide a user-facing command for manual path setting
4. **Backwards Compatibility**: Default behavior should remain unchanged (search from `cwd`)
5. **Validation**: Ensure provided paths exist and are valid directories

---

## Implementation Plan

### 1. Configuration Changes (`config.lua`)

Add new configuration option for the search directory:

```lua
local defaults = {
  filetypes = { "markdown", "json" },
  trigger_char = "@",
  max_results = 5,
  search_tool = "fd",
  search_hidden = false,
  search_gitignore = true,
  relative_paths = true,
  search_dir = nil,  -- NEW: nil = use cwd, string = specific directory
}
```

**Validation**:
- `search_dir` must be nil or a string
- If provided as a string, validate that the path exists and is a directory

---

### 2. File Search Module Changes (`file_search.lua`)

#### 2.1 Update `search_with_fd_async()` function

Modify to accept and use the search directory:

```lua
local function search_with_fd_async(query, config, callback)
  -- ... existing validation ...

  local args = { "--type", "f" }

  -- ... existing args setup ...

  -- Add search directory if specified
  local search_path = "."
  if config.search_dir and config.search_dir ~= "" then
    search_path = config.search_dir
  end

  handle, pid = vim.loop.spawn(
    "fd",
    {
      args = args,
      stdio = { nil, stdout, nil },
      cwd = search_path,  -- NEW: Set working directory for fd
    },
    -- ... rest of the function ...
  )
end
```

#### 2.2 Update `search_with_rg_async()` function

Apply same changes to `rg` implementation:

```lua
local function search_with_rg_async(query, config, callback)
  -- ... existing validation ...

  local args = { "--files" }

  -- ... existing args setup ...

  -- Add search directory if specified
  local search_path = "."
  if config.search_dir and config.search_dir ~= "" then
    search_path = config.search_dir
  end

  handle, pid = vim.loop.spawn(
    "rg",
    {
      args = args,
      stdio = { nil, stdout, nil },
      cwd = search_path,  -- NEW: Set working directory for rg
    },
    -- ... rest of the function ...
  )
end
```

---

### 3. Public API Function (`init.lua`)

#### 3.1 Add `set_path()` function to Source

```lua
-- Set the search directory for this source instance
function Source:set_path(path)
  -- Validate that path exists and is a directory
  if not path or path == "" then
    self.config.search_dir = nil
    return true
  end

  -- Expand path (handle ~, ., .., etc.)
  local expanded_path = vim.fn.expand(path)
  local abs_path = vim.fn.fnamemodify(expanded_path, ":p")

  -- Check if path exists
  if vim.fn.isdirectory(abs_path) ~= 1 then
    vim.notify(
      string.format("fuzzy-path: '%s' is not a valid directory", path),
      vim.log.levels.ERROR
    )
    return false
  end

  -- Update config
  self.config.search_dir = abs_path

  vim.notify(
    string.format("fuzzy-path: Search directory set to '%s'", abs_path),
    vim.log.levels.INFO
  )

  return true
end

-- Get current search directory
function Source:get_path()
  return self.config.search_dir or vim.fn.getcwd()
end
```

#### 3.2 Expose API via module exports

```lua
-- Keep track of the global source instance for command access
local _global_source = nil

-- Setup function for lazy.nvim
local function setup(user_config)
  _global_source = Source:new(user_config)
  return _global_source
end

-- Public API functions
local function set_path(path)
  if not _global_source then
    vim.notify(
      "fuzzy-path: Plugin not initialized. Call setup() first.",
      vim.log.levels.ERROR
    )
    return false
  end
  return _global_source:set_path(path)
end

local function get_path()
  if not _global_source then
    return vim.fn.getcwd()
  end
  return _global_source:get_path()
end

-- Export
return {
  new = function(opts)
    return Source:new(opts)
  end,
  setup = setup,
  set_path = set_path,        -- NEW: Public API
  get_path = get_path,        -- NEW: Public API
}
```

---

### 4. Neovim Command (`init.lua`)

#### 4.1 Create `:FuzzySearchPath` command

Add command registration in the setup function:

```lua
local function setup(user_config)
  _global_source = Source:new(user_config)

  -- Register command
  vim.api.nvim_create_user_command("FuzzySearchPath", function(opts)
    local path = opts.args

    -- If no argument provided, show current path
    if not path or path == "" then
      local current_path = get_path()
      vim.notify(
        string.format("Current fuzzy search directory: %s", current_path),
        vim.log.levels.INFO
      )
      return
    end

    -- Set the new path
    set_path(path)
  end, {
    nargs = "?",  -- Optional argument
    complete = "dir",  -- Directory completion
    desc = "Set or display the fuzzy file search directory",
  })

  return _global_source
end
```

#### 4.2 Command cleanup on plugin unload (optional)

```lua
-- Optional: Provide cleanup function
local function cleanup()
  if vim.api.nvim_del_user_command then
    pcall(vim.api.nvim_del_user_command, "FuzzySearchPath")
  end
end
```

---

### 5. Configuration Validation (`config.lua`)

Update validation to handle the new `search_dir` option:

```lua
local function validate_config(config)
  vim.validate({
    filetypes = { config.filetypes, "table" },
    trigger_char = { config.trigger_char, "string" },
    max_results = { config.max_results, "number" },
    search_tool = { config.search_tool, "string" },
    search_hidden = { config.search_hidden, "boolean" },
    search_gitignore = { config.search_gitignore, "boolean" },
    relative_paths = { config.relative_paths, "boolean" },
    search_dir = { config.search_dir, function(v)
      return v == nil or type(v) == "string"
    end, "nil or string" },  -- NEW
  })

  -- ... existing validations ...

  -- Validate search_dir if provided
  if config.search_dir ~= nil and config.search_dir ~= "" then
    local expanded = vim.fn.expand(config.search_dir)
    local abs_path = vim.fn.fnamemodify(expanded, ":p")

    if vim.fn.isdirectory(abs_path) ~= 1 then
      error(string.format("search_dir '%s' is not a valid directory", config.search_dir))
    end

    -- Normalize to absolute path
    config.search_dir = abs_path
  end
end
```

---

## Usage Examples

### 1. Programmatic Usage

```lua
-- In your Neovim config
local fuzzy_path = require("blink-cmp-fuzzy-path")

-- Set search to a specific project directory
fuzzy_path.set_path("~/projects/my-app")

-- Reset to default (cwd)
fuzzy_path.set_path(nil)
-- or
fuzzy_path.set_path("")

-- Get current search directory
local current = fuzzy_path.get_path()
print("Searching in: " .. current)
```

### 2. Command Usage

```vim
" Show current search directory
:FuzzySearchPath

" Set to specific directory
:FuzzySearchPath ~/projects/my-app

" Set to current buffer's directory
:FuzzySearchPath %:h

" Set to parent directory
:FuzzySearchPath ..

" Reset to cwd (pass empty string via Lua)
:lua require("blink-cmp-fuzzy-path").set_path("")
```

### 3. Configuration at Setup

```lua
{
  'newtoallofthis123/blink-cmp-fuzzy-path',
  dependencies = { 'saghen/blink.cmp' },
  opts = {
    filetypes = { "markdown", "json" },
    trigger_char = "@",
    search_dir = "~/notes",  -- Default search directory
  }
}
```

### 4. Keybindings

Users can create keybindings for common workflows:

```lua
-- Set to current buffer's directory
vim.keymap.set('n', '<leader>fp', function()
  local buf_dir = vim.fn.expand('%:h')
  require('blink-cmp-fuzzy-path').set_path(buf_dir)
end, { desc = "Fuzzy path: set to buffer dir" })

-- Set to project root (git root)
vim.keymap.set('n', '<leader>fP', function()
  local git_root = vim.fn.systemlist('git rev-parse --show-toplevel')[1]
  if git_root and git_root ~= "" then
    require('blink-cmp-fuzzy-path').set_path(git_root)
  end
end, { desc = "Fuzzy path: set to git root" })

-- Reset to cwd
vim.keymap.set('n', '<leader>fr', function()
  require('blink-cmp-fuzzy-path').set_path(nil)
end, { desc = "Fuzzy path: reset to cwd" })
```

---

## Testing Checklist

### Manual Testing

- [ ] **Setup with `search_dir` config**: Plugin loads with initial search directory
- [ ] **`set_path()` with valid directory**: Search results come from specified directory
- [ ] **`set_path()` with invalid directory**: Error message shown, search_dir unchanged
- [ ] **`set_path(nil)` or `set_path("")`**: Reset to default (cwd)
- [ ] **`get_path()`**: Returns current search directory correctly
- [ ] **`:FuzzySearchPath` with no args**: Displays current search directory
- [ ] **`:FuzzySearchPath ~/some/path`**: Sets new search directory
- [ ] **`:FuzzySearchPath` with tab completion**: Directory completion works
- [ ] **`:FuzzySearchPath` with relative path**: Path expanded correctly (e.g., `..`, `.`)
- [ ] **`:FuzzySearchPath` with `~` expansion**: Home directory expanded correctly
- [ ] **`:FuzzySearchPath` with `%:h`**: Current buffer directory set correctly
- [ ] **File search respects `search_dir`**: Only files within search_dir appear
- [ ] **Relative paths still work**: Files shown relative to buffer directory
- [ ] **Both `fd` and `rg` respect `search_dir`**: Test with both tools
- [ ] **Multiple source instances**: Each instance can have its own search_dir

### Edge Cases

- [ ] **search_dir doesn't exist**: Proper error handling
- [ ] **search_dir is a file, not a directory**: Proper error handling
- [ ] **search_dir has no read permissions**: Graceful failure
- [ ] **search_dir is empty directory**: No results, no errors
- [ ] **search_dir is very large**: Performance acceptable (search doesn't hang)
- [ ] **Changing search_dir mid-typing**: Next completion uses new directory
- [ ] **Rapid search_dir changes**: No race conditions or crashes

---

## Implementation Steps

### Phase 1: Core Functionality

1. **Update `config.lua`**:
   - Add `search_dir` to defaults
   - Add validation for `search_dir`
   - Expand and normalize paths

2. **Update `file_search.lua`**:
   - Modify `search_with_fd_async()` to use `config.search_dir`
   - Modify `search_with_rg_async()` to use `config.search_dir`
   - Test that searches are constrained to specified directory

3. **Update `init.lua`**:
   - Add `set_path()` method to Source
   - Add `get_path()` method to Source
   - Track global source instance
   - Export public API functions

4. **Test programmatic API**:
   - Verify `set_path()` works
   - Verify `get_path()` works
   - Verify searches respect new directory

### Phase 2: Command Interface

5. **Add Neovim command**:
   - Register `:FuzzySearchPath` command in `setup()`
   - Handle no-argument case (display current path)
   - Handle argument case (set new path)
   - Add directory completion

6. **Test command interface**:
   - Verify command works with various path formats
   - Verify tab completion works
   - Verify error messages are clear

### Phase 3: Documentation & Polish

7. **Update README**:
   - Document `set_path()` and `get_path()` functions
   - Document `:FuzzySearchPath` command
   - Add usage examples
   - Add keybinding suggestions

8. **Add health check** (optional):
   - Check if search_dir is valid
   - Warn if search_dir is inaccessible

---

## API Reference

### Functions

#### `require("blink-cmp-fuzzy-path").set_path(path)`

Set the directory where fuzzy file search should be performed.

**Parameters**:
- `path` (string|nil): Directory path. Can be:
  - Absolute path: `/home/user/projects/myapp`
  - Relative path: `..`, `./subdir`
  - Home directory: `~/documents`
  - Buffer expansion: `%:h` (current buffer's directory)
  - `nil` or `""`: Reset to default (cwd)

**Returns**:
- `boolean`: `true` if path was set successfully, `false` otherwise

**Example**:
```lua
require("blink-cmp-fuzzy-path").set_path("~/projects/notes")
```

---

#### `require("blink-cmp-fuzzy-path").get_path()`

Get the current search directory.

**Returns**:
- `string`: Current search directory (absolute path)

**Example**:
```lua
local current_dir = require("blink-cmp-fuzzy-path").get_path()
print("Searching in: " .. current_dir)
```

---

### Commands

#### `:FuzzySearchPath [path]`

Set or display the fuzzy file search directory.

**Arguments**:
- `[path]` (optional): Directory path to set. Supports tab completion.

**Behavior**:
- No argument: Display current search directory
- With argument: Set new search directory

**Examples**:
```vim
:FuzzySearchPath                    " Show current directory
:FuzzySearchPath ~/projects/myapp   " Set to specific directory
:FuzzySearchPath %:h                " Set to current buffer's directory
:FuzzySearchPath ..                 " Set to parent directory
```

---

### Configuration

#### `search_dir` (string|nil)

Default search directory. If not set, defaults to current working directory.

**Example**:
```lua
{
  'newtoallofthis123/blink-cmp-fuzzy-path',
  opts = {
    search_dir = "~/my-notes",  -- Default search directory
  }
}
```

---

## Benefits

1. **Project-focused searches**: Constrain searches to relevant directories
2. **Multi-project workflows**: Switch search scope without changing cwd
3. **Performance**: Faster searches in large repositories by limiting scope
4. **Flexibility**: Both programmatic and interactive control
5. **Intuitive**: Command with directory completion makes it easy to use

---

## Future Enhancements

- **Buffer-local search directories**: Different directories per buffer
- **Auto-detect project root**: Automatically set to git root or similar
- **Search directory history**: Remember recent search directories
- **Multiple search directories**: Search in multiple directories simultaneously
- **Workspace-aware**: Integration with Neovim workspace/session management

---

## Notes

- The `search_dir` is stored in the config and affects all subsequent searches
- Setting `search_dir` to `nil` or empty string restores default behavior (search from cwd)
- Path expansion happens during validation and when calling `set_path()`
- The command provides directory tab completion for better UX
- Error messages are shown via `vim.notify()` for visibility
- The API is designed to work with both single and multiple source instances
