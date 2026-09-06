---
name: hackingtool-install-all-review
description: Reviews hackingtool's bulk-install path — HackingToolsCollection.show_options's "Install all" (menu option 97) in core.py — for whether a user can trigger many third-party tools' INSTALL_COMMANDS (sudo, curl|sh, git clone + pip/gem/go installs) in a single keypress without ever seeing the actual commands or being asked to confirm. Distinct from hackingtool-menu-review, which checks whether a tool's INSTALL_COMMANDS are well-formed and from a legitimate source — this is about whether the user consented to what's about to run at all. Use this whenever a diff touches HackingToolsCollection.show_options's choice==97 branch, HackingTool.install()/run()/update(), or adds a new tool/category that grows what "Install all" silently executes.
---

# "Install all" executes N tools' install commands with no preview and no confirmation

`HackingToolsCollection.show_options` (`core.py`) has a documented, generic "Install all" action
available in every category (`_show_inline_help` even advertises `97  install all (in category)`).
The implementation is:

```python
elif choice == 97 and not_installed:
    console.print(Panel(f"[bold]Installing {len(not_installed)} tools...[/bold]", ...))
    for i, tool in enumerate(not_installed, start=1):
        console.print(f"\n[bold cyan]({i}/{len(not_installed)})[/bold cyan] {tool.TITLE}")
        try:
            tool.install()
        except Exception:
            console.print(f"[error]✘ Failed: {tool.TITLE}[/error]")
    Prompt.ask("\n[dim]Press Enter to continue[/dim]", default="")
```

There is no gate before this loop starts — no `Confirm.ask`, no listing of the actual
`INSTALL_COMMANDS` about to run. The only thing the user has seen beforehand is the category's
tool table, which shows each tool's **title and first line of description** (`show_options`'s
`table.add_row(str(index), status, tool.TITLE, desc)`), never the commands themselves. A single
keypress of `97` then calls `tool.install()` back-to-back for every not-installed tool in that
category, and `install()` only prints a command (`console.print(f"[warning]→ {cmd}[/warning]")`)
*immediately before* handing it to `os.system(cmd)` — i.e. the user learns what's running at the
same instant it starts running, once per tool, with no way to review the full batch first and no
per-tool or per-batch "are you sure" step.

## Why this is a real, not hypothetical, blast radius

Pick almost any category and the commands swept into one `97` keypress are already varied and
sometimes destructive. `tools/xss_attack.py`'s `XSSAttackTools` (9 tools) alone contains, across
its members' `INSTALL_COMMANDS`:
- `sudo apt-get install -y golang` (Dalfox) and `sudo apt install librust-openssl-dev` (RVuln)
- `sudo pip3 install -r requirements.txt` (XSSPayloadGenerator, XSSFreak)
- `sudo rm -rf XSStrike` unconditionally, before re-cloning it (XSSStrike)
- `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` (RVuln) — a legitimate official
  installer per `hackingtool-menu-review` section 5, but still a third-party network script
  execution the user never explicitly opted into
- a bare `sudo su` embedded mid-command-string (RVuln) — which, run non-interactively via
  `os.system` inside an unattended "install all" loop, is exactly the kind of command a user would
  want a chance to see and reject before it fires

`tools/information_gathering.py`'s `InformationGatheringTools` category is larger still — 26 tools,
9 separate `sudo` invocations across their `INSTALL_COMMANDS` — all reachable via the same single
`97` keypress from that category's menu.

## The codebase already has, and uses, the fix pattern — just not here

`tools/tool_manager.py`'s `UninstallTool` (hackingtool's own self-uninstall) gates its action with
exactly the pattern this path lacks:
```python
if not Confirm.ask("Continue?", default=False):
    ...
if Confirm.ask(f"Also remove user data at {USER_CONFIG_DIR}?", default=False):
    ...
```
So the "ask before an irreversible/bulk action, default to no" idiom is established precedent in
this repo (`rich.prompt.Confirm`, imported in both `hackingtool.py` and `tool_manager.py`) — it is
simply absent from the one path that fans out to the largest number of `sudo`/network-fetching
commands per keypress anywhere in the app.

## What to check on a relevant diff

- Any diff touching the `choice == 97` branch: does it still start the install loop with zero
  confirmation? A fix should show the actual `INSTALL_COMMANDS` (or at least which not-installed
  tools carry `sudo`/root-requiring/network-script commands) and gate with `Confirm.ask(...,
  default=False)`, not just print a tool-title progress panel after the fact.
- Any diff adding a tool to an existing category's `TOOLS` list: that tool's `INSTALL_COMMANDS`
  become reachable through that category's `97` with no additional review step of their own — if
  the new tool adds `sudo`, a piped-script installer, or a destructive command, treat that as
  raising the stakes of the existing `97` gap, not as a new isolated concern.
- Don't accept a fix that only reduces the tool-title panel's noise (e.g. collapsing progress
  output) as addressing this — the gap is the missing **pre-execution** preview/confirmation, not
  the in-progress logging.
- This is unrelated to `hackingtool-menu-review` section 5 (whether a `curl|sh` install is from a
  legitimate upstream source) — a command can be perfectly legitimate and still be something the
  user never agreed to run *right now, as part of a batch of unrelated tools*. Review both
  independently; don't treat one as covering the other.
