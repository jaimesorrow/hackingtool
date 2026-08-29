---
name: hackingtool-menu-review
description: Reviews diffs in this repo (hackingtool, a Python/Rich menu-driven launcher that installs and runs 185+ third-party security tools) against its actual launcher-plumbing invariants — how core.py infers a tool's install directory from its own INSTALL_COMMANDS string, how menu registration and README generation stay in sync, and how user input reaches the shell. Use this instead of a generic code review for any change to core.py, hackingtool.py, os_detect.py, config.py, constants.py, generate_readme.py, or any file under tools/ (including tools/others/) — especially PRs that add or edit a HackingTool tool entry.
---

# Hackingtool launcher review

This is a menu/launcher aggregator, not a tool itself: `core.py` defines the generic
`HackingTool` / `HackingToolsCollection` engine, and every file under `tools/` (plus
`tools/others/`) is a list of small `HackingTool` subclasses that each wrap one third-party
security tool's install/run commands. Review changes here for launcher correctness and
safety — not for the offensive capability of the wrapped tools themselves.

## 1. `INSTALL_COMMANDS`/`RUN_COMMANDS` must agree with `core.py`'s string-parsing

`core.py` (`is_installed`, `update`, `_get_tool_dir`, `open_folder`) does **not** track a
tool's directory explicitly — it re-derives it every time by parsing the tool's own class
attributes:
- the clone directory is inferred from the last path segment of the first `http...` URL
  found in an `INSTALL_COMMANDS` entry containing `"git clone"` (`.git` stripped);
- `update()` re-dispatches based on substring matches (`"git clone"` → `git pull`,
  `"pip install"` → `pip install --upgrade`, `"go install"` → re-run, `"gem install"` →
  `gem update`);
- `is_installed`/`_get_tool_dir` also try to resolve a binary from `RUN_COMMANDS[0]`,
  stripping a leading `cd ... &&` or `sudo `.

For any new or edited `HackingTool` subclass, check that:
- if `RUN_COMMANDS` starts with `cd <dir>;` or `cd <dir> &&`, `<dir>` actually matches the
  directory name the `git clone` URL in `INSTALL_COMMANDS` will produce (or the custom
  target-dir argument after the URL, if one is given) — a mismatch silently breaks
  install-status detection, "Update", and "Open Folder" for that tool.
- `INSTALL_COMMANDS` only contains one `git clone` (or, if multiple, the first one is the
  tool's own repo) — the parser only looks at the first match.
- if the tool has no natural binary and no clone dir (e.g. a pure-webbrowser helper like
  `IsItDown`), it's constructed with `installable=False`/`runnable=False` rather than left
  to silently report "not installed" forever.

## 2. Menu registration must stay two-sided and index-parallel

A new **category** (not just a new tool inside an existing category) must be added in
*both* parallel lists in `hackingtool.py`, at the same position:
- `tool_definitions` — `(full_title, icon, menu_label)`
- `all_tools` — the matching `HackingToolsCollection()` instance

`interact_menu()` indexes into `all_tools` using the number the user typed against
`tool_definitions[choice - 1]` for display — these two lists silently going out of index-sync
is a correctness bug, not just cosmetic. `ToolManager`/"Update or Uninstall" must stay last in
both lists, since `generate_readme.py` explicitly slices `all_tools[:-1]` to drop it before
building the tool table-of-contents/README body.

A new **tool** inside an existing category just needs adding to that category's
`HackingToolsCollection.TOOLS` list (e.g. `tools/information_gathering.py`'s
`InformationGatheringTools.TOOLS`) — verify it was actually appended there, not just defined
and left unreferenced.

## 3. README.md is generated, not hand-written, for the tool listing

`generate_readme.py` builds the table of contents and per-category tool list from `all_tools`
and writes over `README.md` using `README_template.md` as the source (`{{toc}}`/`{{tools}}`
placeholders). If a diff hand-edits the tool-listing portions of `README.md` directly instead
of updating `README_template.md` and/or the `all_tools`/category data and regenerating, the
next regeneration will silently discard those edits — flag it.

## 4. Don't let user input reach a shell string

The existing code is deliberate about this — e.g. `Striker.run`/`HatCloud.run` use
`subprocess.run([...], cwd=...)` with argument lists specifically to avoid the injection risk
of interpolating a user-supplied site/host into a shell string (see the "Bug 3 fix" comments
already in `tools/information_gathering.py` and `tools/other_tools.py` noting `os.chdir()` was
replaced with `cwd=`). For any new/changed `run()` method that takes user input via
`Prompt.ask` (a hostname, IP, filename, etc.) and shells out:
- it must use the list form of `subprocess.run`, never `os.system(f"...")` or
  `subprocess.run(str, shell=True)`, when the interpolated value came from the user;
- `os.system`/`shell=True` remain fine only for the class's own static, hardcoded
  `INSTALL_COMMANDS`/`RUN_COMMANDS`/`UNINSTALL_COMMANDS` strings (that's the existing pattern
  throughout `core.py`, `os_detect.py`, `tool_manager.py`), not for anything built from a
  prompt.

## 5. Remote script execution in `INSTALL_COMMANDS` needs to be the tool's own official installer

Piping `curl`/`wget` straight into `sh`/`bash` already appears for a couple of tools (Trivy,
rustup) — that's acceptable *only* when it's the upstream project's own documented installer
URL (official repo/domain, HTTPS, matches what that tool's own README tells users to run).
Flag any new `INSTALL_COMMANDS` that:
- pipes a script from a personal pastebin/gist/URL-shortener into a shell (there's precedent
  for removing exactly this — see the "Bug 29 fix" comment in `tools/wireless_attack.py`
  removing a `pastebin.com` download);
- downloads over plain `http://` where `https://` is available;
- runs a downloaded script with elevated privileges without that being the tool's documented
  install path.

## 6. OS/capability metadata should be accurate, since it's load-bearing

`HackingToolsCollection._active_tools()` hides a tool from the menu entirely when
`CURRENT_OS.system` isn't in its `SUPPORTED_OS`. For a new tool, check `SUPPORTED_OS`
(default `["linux", "macos"]`) is actually correct — e.g. anything using `apt-get`/`dpkg`
directly, Bluetooth/Wi-Fi monitor-mode tooling, or Linux-only kernel features should declare
`SUPPORTED_OS = ["linux"]`; and the `REQUIRES_WIFI`/`REQUIRES_ROOT`/`REQUIRES_GO`/
`REQUIRES_RUBY`/`REQUIRES_JAVA`/`REQUIRES_DOCKER` flags should reflect what the tool actually
needs, since these are the only place that information is recorded.

## 7. No hardcoded user paths

`constants.py` is explicit about this (`USER_CONFIG_DIR = Path.home() / ...`, with a comment
against hardcoding `/home/username`). Any new code that needs a user-writable path should go
through `Path.home()` / `config.get_tools_dir()` / the existing `constants.py` path constants
— never a literal `/home/<user>/...` or `/root/...`.

## 8. Ordinary correctness, on top of the above

Also check for the usual things given this file's own patterns: `TITLE`/`DESCRIPTION` set
(the description's first line is what shows in menu tables); `PROJECT_URL` present and
matching the actual clone URL (`show_project_page`/option `98` open it); `OPTIONS` tuples
follow the existing `(label, bound_method)` shape; and that `RUN_COMMANDS`/`INSTALL_COMMANDS`
are lists of strings (not a bare string), since `install()`/`run()`/`uninstall()` iterate them
assuming that shape.
