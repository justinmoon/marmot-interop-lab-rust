# AGENTS.md instructions for /Users/justin/code/marmot-interop-lab-rust

<INSTRUCTIONS>
## Skills
A skill is a set of local instructions to follow that is stored in a `SKILL.md` file. Below is the list of skills that can be used. Each entry includes a name, description, and file path so you can open the source for full instructions when using a specific skill.
### Available skills
- diagram: Render Mermaid diagrams as ASCII art in the terminal or as beautiful themed SVGs. Use when you need to visualize architecture, data flows, state machines, sequences, class hierarchies, or ER models. Supports flowcharts, sequence diagrams, state diagrams, class diagrams, and ER diagrams. (file: /Users/justin/configs/home/skills/diagram/SKILL.md)
- global-agents-md: Add global agents reference to a repo. Use when initializing a repo for AI agent collaboration or when asked to set up AGENTS.md. (file: /Users/justin/configs/home/skills/global-agents-md/SKILL.md)
- google: Access Google services (Gmail, Calendar, Drive, Docs, Sheets, Contacts, Tasks) via gogcli. Use for reading/sending email, managing calendar events, accessing Drive files, and other Google Workspace operations. (file: /Users/justin/configs/home/skills/google/SKILL.md)
- init-ci: Initialize CI/CD for a project using Nix + GitHub Actions + Blacksmith runners. Use when setting up continuous integration, adding pre-merge checks, post-merge deploys, or nightly jobs. Creates GitHub Actions workflows that delegate to justfile recipes running inside nix develop. (file: /Users/justin/configs/home/skills/init-ci/SKILL.md)
- nix: Nix flakes, devShells, NixOS modules, and build patterns. Use when working with flake.nix, NixOS configs, derivations, or debugging nix issues. Covers Crane (Rust), uv2nix (Python), fixed-output derivations (Bun/npm). (file: /Users/justin/configs/home/skills/nix/SKILL.md)
- worktree: Build features in git worktrees. Creates a worktree, opens a new tmux window, and launches a SEPARATE AI agent with the prompt. Use when planning is complete and implementation should happen in isolation. IMPORTANT - you must launch a new agent, not implement yourself. (file: /Users/justin/configs/home/skills/worktree/SKILL.md)
- skill-creator: Guide for creating effective skills. This skill should be used when users want to create a new skill (or update an existing skill) that extends Codex's capabilities with specialized knowledge, workflows, or tool integrations. (file: /Users/justin/configs/home/skills/.system/skill-creator/SKILL.md)
- skill-installer: Install Codex skills into $CODEX_HOME/skills from a curated list or a GitHub repo path. Use when a user asks to list installable skills, install a curated skill, or install a skill from another repo (including private repos). (file: /Users/justin/configs/home/skills/.system/skill-installer/SKILL.md)
### How to use skills
- Discovery: The list above is the skills available in this session (name + description + file path). Skill bodies live on disk at the listed paths.
- Trigger rules: If the user names a skill (with `$SkillName` or plain text) OR the task clearly matches a skill's description shown above, you must use that skill for that turn. Multiple mentions mean use them all. Do not carry skills across turns unless re-mentioned.
- Missing/blocked: If a named skill isn't in the list or the path can't be read, say so briefly and continue with the best fallback.
- How to use a skill (progressive disclosure):
  1) After deciding to use a skill, open its `SKILL.md`. Read only enough to follow the workflow.
  2) When `SKILL.md` references relative paths (e.g., `scripts/foo.py`), resolve them relative to the skill directory listed above first, and only consider other paths if needed.
  3) If `SKILL.md` points to extra folders such as `references/`, load only the specific files needed for the request; don't bulk-load everything.
  4) If `scripts/` exist, prefer running or patching them instead of retyping large code blocks.
  5) If `assets/` or templates exist, reuse them instead of recreating from scratch.
- Coordination and sequencing:
  - If multiple skills apply, choose the minimal set that covers the request and state the order you'll use them.
  - Announce which skill(s) you're using and why (one short line). If you skip an obvious skill, say why.
- Context hygiene:
  - Keep context small: summarize long sections instead of pasting them; only load extra files when needed.
  - Avoid deep reference-chasing: prefer opening only files directly linked from `SKILL.md` unless you're blocked.
  - When variants exist (frameworks, providers, domains), pick only the relevant reference file(s) and note that choice.
- Safety and fallback: If a skill can't be applied cleanly (missing files, unclear instructions), state the issue, pick the next-best approach, and continue.
</INSTRUCTIONS>
