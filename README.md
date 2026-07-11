# Agent Skills

Agent skills, written once, reused anywhere.

A curated collection of reusable, project-agnostic skills for AI coding agents (e.g. Claude Code). Each skill lives in its own folder and can be dropped independently into any project or global skills directory.

## Structure
<skill-name>/
SKILL.md

Each `SKILL.md` is self-contained: it declares its name, trigger conditions, and full instructions, so it can be copied out and used on its own without depending on the rest of the repo.

## Usage
Copy the skill folder you want into your agent's skills directory (e.g. `~/.claude/skills/` for Claude Code), or clone the whole repo and symlink individual skills.

## Contributing
This repo is maintained as a personal collection; skills are added manually as they're generalized enough to be useful outside their original project.

## License
MIT
