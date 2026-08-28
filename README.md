# moodle-ai-skills

Claude Code skills for Moodle core development and release workflows.

## Skills

### `moodle-library-upgrade`

Guides upgrading a vendored third-party library in a Moodle codebase end-to-end:

- Locates the library in `thirdpartylibs.xml` by shortname, fullname, or folder name
- Reads its `readme_moodle` file for documented upgrade steps
- Fetches the latest upstream release, classifies the bump by semver (major/minor/patch), and flags breaking changes on major bumps
- Checks the changelog/CVEs for security fixes
- Walks through replacing vendored files and reapplying Moodle-specific patches
- Updates `thirdpartylibs.xml` and commits the change on an `MDL-XXXXX-main` branch
- For security-relevant fixes, follows Moodle's security process: sets the Jira issue to Bug with the right security level (via the Atlassian MCP), determines which Moodle versions are affected, and backports a fix-only patch per affected branch with `mdk push -t`

Run it from the root of a Moodle checkout (a directory containing `thirdpartylibs.xml`), and invoke it with `/moodle-library-upgrade` or by asking to upgrade/bump a specific library.

## Requirements

- **Atlassian MCP** connected and authorized — used for Jira status transitions, updating testing instructions, and adding labels
- **[Moodle Development Kit (mdk)](https://github.com/fmcorz/mdk)** installed
- **Git** installed

## Installation

Copy the skill folder into your Claude Code skills directory (e.g. `~/.claude/skills/`), or clone this repo there directly.
