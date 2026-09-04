---
name: hackingtool-conventions
description: File-structure and naming reference for where a new HackingTool entry's code actually lives and how it's named — distinct from hackingtool-menu-review's review checklist. Consult this when adding a new tools/*.py module or class, or to find where os_detect.py/constants.py values come from, before writing the code (review of the resulting diff still belongs to hackingtool-menu-review).
---

# File structure & naming conventions

This is a "where does it go / what do I call it" reference. For invariant/correctness checks on
the resulting code, use `hackingtool-menu-review` instead — this file doesn't duplicate it.

## Where tool files live

- One module per **menu category** directly under `tools/`, named after the category in
  `snake_case` (e.g. `tools/wireless_attack.py`, `tools/sql_injection.py`, `tools/active_directory.py`).
  Each module defines one or more `HackingTool` subclasses plus exactly one
  `HackingToolsCollection` subclass that lists them in `TOOLS`, and that collection class is what
  gets imported into `hackingtool.py`'s `all_tools`.
- `tools/others/` is a *second*, nested package for tools that don't merit their own top-level
  category — `OtherTools` (`tools/other_tools.py`) imports from here (e.g.
  `tools/others/socialmedia.py`, `tools/others/hash_crack.py`, `tools/others/wifi_jamming.py`) and
  is itself just another `HackingToolsCollection` wired into `all_tools` like any other category.
  Put a genuinely new, narrow tool here rather than inventing a new top-level category for one tool.
- `tools/tool_manager.py` (`ToolManager`/`UpdateTool`) is the one category that isn't
  install/run-shaped (self-update); don't use it as a template for an ordinary tool entry.
- Both `tools/__init__.py` and `tools/others/__init__.py` are empty — they exist only to make the
  directories importable packages, not for shared code.

## Naming pattern

- `HackingTool` subclasses: `PascalCase`, named after the wrapped tool itself, not what it does
  (`NMAP`, `Dracnmap`, `XeroSploit`, `Striker` — not `NetworkScanner1`). Where a class isn't a
  wrapped external tool but a small built-in action, it's still `PascalCase` and named for the
  action (`PortScan`, `Host2IP`).
- `HackingToolsCollection` subclasses: `PascalCase` + `Tools` suffix, matching the module/category
  (`InformationGatheringTools`, `WirelessAttackTools`, `MobileSecurityTools`).
- Module-level `console`/`_theme` redefinitions seen in some `tools/*.py` files (e.g.
  `information_gathering.py`) are leftover duplicates of `core.py`'s shared `console` — new code
  should just `from core import console` rather than adding another one.

## How `os_detect.py`/`constants.py` fit in

- `os_detect.py` exposes a single computed-once singleton, `CURRENT_OS` (an `OSInfo` dataclass:
  `system`, `distro_id`, `pkg_manager`, `is_root`, `home_dir`, `is_wsl`, `arch`), plus
  `PACKAGE_INSTALL_CMDS`/`PACKAGE_UPDATE_CMDS`/`REQUIRED_PACKAGES` dicts keyed by package manager,
  and `install_packages()`. `HackingToolsCollection._active_tools()` is the only place `CURRENT_OS`
  gates the menu; use it (not `platform.system()` directly) for any new OS-conditional logic.
- `constants.py` holds repo identity (`REPO_OWNER`/`REPO_URL`), version, path constants built from
  `Path.home()` (never hardcode `/home/<user>` — see its own comment), the Rich theme color
  constants (`THEME_*`, imported into `core.py`'s `_theme`), `DEFAULT_CONFIG`, and `PRIV_CMD`
  (`doas` if present, else `sudo`). New user-facing paths or theme colors belong here, not inlined
  in a `tools/*.py` file.
