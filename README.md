# moodle-ai-skills

Claude Code skills for Moodle core development and release workflows.

## Skills

### `moodle-library-upgrade`

Guides upgrading a vendored third-party library in a Moodle codebase end-to-end:

- Locates the library in `thirdpartylibs.xml` by shortname, fullname, or folder name
- Reads its `readme_moodle` file for documented upgrade steps
- Fetches the latest upstream release and checks the changelog/CVEs for security fixes
- Walks through replacing vendored files and reapplying Moodle-specific patches
- Updates `thirdpartylibs.xml` and commits the change on an `MDL-XXXXX-main` branch

Run it from the root of a Moodle checkout (a directory containing `thirdpartylibs.xml`), and invoke it with `/moodle-library-upgrade` or by asking to upgrade/bump a specific library.

## Installation

Copy the skill folder into your Claude Code skills directory (e.g. `~/.claude/skills/`), or clone this repo there directly.
