---
name: data-model
description: Pure reference for the HackingTool/HackingToolsCollection "schema" this repo is built on — the real class fields, how a tool entry relates to hackingtool.py's registries and the generated README, what validation actually exists (vs. doesn't), how the menu queries tools, and what a new tool entry mutation looks like. Consult this to look up a field name/type or a relationship, not for review guidance (see hackingtool-menu-review for that).
---

# HackingTool data model

## Primary entities

**`HackingTool`** (`core.py`) — one third-party tool. Class attributes, all with defaults
(instances mutate `OPTIONS` in `__init__`, everything else stays class-level unless overridden):
`TITLE: str`, `DESCRIPTION: str`, `INSTALL_COMMANDS: list[str]`, `UNINSTALL_COMMANDS: list[str]`,
`RUN_COMMANDS: list[str]`, `OPTIONS: list[tuple[str, Callable]]`, `PROJECT_URL: str`,
`SUPPORTED_OS: list[str]` (default `["linux", "macos"]`), `REQUIRES_ROOT/WIFI/GO/RUBY/JAVA/DOCKER: bool`,
`TAGS: list[str]` (default `[]` — no tool in `tools/` currently sets this), `ARCHIVED: bool`,
`ARCHIVED_REASON: str`. `__init__(options, installable, runnable)` builds `OPTIONS` from
Install/Run (conditionally) + always Update + Open Folder + any extra `options`.

**`HackingToolsCollection`** (`core.py`) — one menu category. Fields: `TITLE: str`,
`DESCRIPTION: str`, `TOOLS: list[HackingTool]`.

## Relationships

- `hackingtool.py` holds two **index-parallel** top-level lists: `tool_definitions` (tuples of
  `(full_title, icon, menu_label)`) and `all_tools` (the matching `HackingToolsCollection()`
  instances). Position `i` in one must describe position `i` in the other.
- `AllTools` (`hackingtool.py`) wraps `all_tools` for `generate_readme.py`, which walks it via
  `get_toc`/`get_tools_toc` to fill `README_template.md`'s `{{toc}}`/`{{tools}}` placeholders and
  writes `README.md`. A tool's `TITLE`/`PROJECT_URL` become its README bullet.

## Validation rules

There is no schema enforcement — no dataclass validation, no `__post_init__` checks, no CI
step that type-checks a `HackingTool` subclass. The only "validation" is structural/implicit:
`install()`/`run()`/`uninstall()` assume `INSTALL_COMMANDS`/`RUN_COMMANDS`/`UNINSTALL_COMMANDS`
are lists of strings (`isinstance(..., (list, tuple))` guarded, but wrong element types fail at
`os.system()` time, not at load time). `is_installed`/`_get_tool_dir` degrade silently (return
`False`/`None`) rather than raise on malformed `INSTALL_COMMANDS`/`RUN_COMMANDS` strings.

## Query patterns

`HackingToolsCollection._active_tools()` filters `TOOLS` by `not ARCHIVED and CURRENT_OS.system
in SUPPORTED_OS` (`os_detect.CURRENT_OS`); `_archived_tools()`/`_incompatible_tools()` are the
complementary filters. `hackingtool.py`'s `_get_all_tags()` builds a tag→tools index by unioning
each tool's `TAGS` with tags auto-derived by regex over `f"{TITLE} {DESCRIPTION}"`; `search_tools()`
does a plain substring match over title/description/tags.

## Mutations

Adding a tool = append a `HackingTool` subclass instance to an existing category's
`TOOLS` list. Adding a *category* = append to both `tool_definitions` and `all_tools` at the
same index, with `ToolManager`/its definition staying last in both.
