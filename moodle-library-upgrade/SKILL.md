---
name: moodle-library-upgrade
description: >
  Guides upgrading a vendored third-party library in a Moodle codebase end-to-end:
  locates it in thirdpartylibs.xml by shortname or fullname, confirms the match,
  reads its readme_moodle file for upgrade steps, fetches the latest upstream
  release, classifies the bump (semver) and checks for breaking changes,
  deprecations, and security fixes, reapplies Moodle-specific patches, updates
  thirdpartylibs.xml, and commits on an MDL branch. Offers to update codebase
  call sites broken or deprecated by the new version. When security-relevant,
  walks through Moodle's security process: Jira issue type/security level,
  affected Moodle versions, and per-branch backport with `mdk push -t`
  (cherry-pick, manual patch, or minor-version fallback on stable branches). Use
  when user says "upgrade library", "update third party library", "bump
  <library>", or invokes /moodle-library-upgrade.
---

# Moodle Library Upgrade Skill

You are guiding the user through upgrading one third-party (vendored) library in a Moodle codebase. Follow each phase in order. Never skip a step. Never decide security-process or licensing questions yourself; surface them, and whatever the user decides is the final word - proceed accordingly, or stop if they say it needs review outside this conversation.

Two kinds of instructions appear below, and they mean different things:
- **"Ask:" with a quoted question - a blocking checkpoint.** Stop, ask exactly that, and wait for an explicit answer before doing anything else in that step.
- **"Note/flag/disclose" - non-blocking.** Say it as part of your normal narration (or carry it into the final summary) and keep going; it is not a stop-and-wait point.

## Required: Run From a Moodle Codebase

This skill assumes the current working directory is a Moodle codebase, or a subdirectory of one - not necessarily the location of the relevant `thirdpartylibs.xml` itself. Moodle vendors third-party libraries both at the codebase root and inside individual component directories (`mod/*`, `blocks/*`, `admin/tool/*`, `filter/*`, etc.), each carrying its own `thirdpartylibs.xml`. If no `thirdpartylibs.xml` file exists anywhere under the current directory, stop and ask the user for the correct path or to `cd` into the right checkout - do not search elsewhere on disk.

---

## PHASE 1: Locate & Confirm the Library

### Step 1.1 - Get the Library Name

If not already given, ask:
> "Which library do you want to upgrade? (shortname, fullname, or folder name is fine)"

### Step 1.2 - Find All thirdpartylibs.xml Files

Search the codebase for every `thirdpartylibs.xml` file (e.g. `find . -name thirdpartylibs.xml`, or a recursive grep), not just one at the current directory. Parse all `<library>` entries (`<name>`, `<location>`, `<license>`, `<licenseversion>`, `<version>`, `<version_comment>`) from each file found, keeping track of which file each entry came from.

### Step 1.3 - Match

Match the given name against `<name>` and the last path segment of `<location>`, case-insensitively, allowing partial/fuzzy matches, across every `thirdpartylibs.xml` file found in Step 1.2.

- **No match anywhere:** show the closest candidates (if any) and ask the user to clarify or provide the exact name.
- **Multiple matches** (the library is vendored in more than one component, or several distinct libraries match the name): list them all - file path, name, location, current version - and ask the user to pick one.
- **One match:** show it - which `thirdpartylibs.xml` it came from, name, location, current version, license - and ask explicitly:
  > "Is this the library you want to upgrade? [name] at [location], currently version [version], recorded in [path/to/thirdpartylibs.xml]."

Do not proceed past this step without an explicit yes.

Once confirmed, record the matched file's path as `THIRDPARTYLIBS_FILE` - every later step that reads or writes "thirdpartylibs.xml" (notably Step 5.1) means this specific file, which may live inside a component directory rather than at the codebase root. Operate from that component directory for locating and inspecting the library's own folder (Step 1.4 onward), but keep git operations (branch, add, commit) rooted at the actual git repository root, wherever that is relative to `THIRDPARTYLIBS_FILE`.

### Step 1.4 - Resolve the Path & Sanity-Check It

Resolve `<location>` to a full path and list its contents.

If the directory is missing, empty, or clearly doesn't contain the library's actual code (e.g. only a readme, no source files) - stop and ask:
> "[location] doesn't look like a normal vendored library checkout - [describe what's there/missing]. Is this expected (e.g. a partial or stripped checkout), or should I stop so you can point me at the right checkout?"

- If the user says it's expected: continue, but carry this forward explicitly - there's no baseline file to diff, so Step 4.2 becomes "add" the new file rather than "replace" (no diff to show), and Step 6.1's patch verification can only be guessed at from the readme's prose, not confirmed against prior code. Say this plainly in your final summary as a real limitation, not a silently-resolved detail.
- If the user says it's not expected: stop the skill entirely and wait for them to fix the checkout or give you a different path. Do not guess or search elsewhere on disk.

### Step 1.5 - Get the Tracker Number

Ask:
> "What's the MDL tracker issue number for this upgrade?"

Store it as `MDL_NUMBER` for the rest of this session - you'll need it for commit messages, patch comments, and any readme notes in later phases. Do not proceed to any file edit without one, and never invent one.

---

## PHASE 2: Read the Upgrade Instructions

### Step 2.1 - Find the readme

Look inside the resolved library path, in this order: `readme_moodle.txt`, `readme_moodle.md`, `readme.md`. Use the first one found.

If none exist, stop and tell the user: "No readme_moodle file found for this library - I don't have documented upgrade steps to follow. How would you like to proceed?"

### Step 2.2 - Extract Key Info

From the readme, note:
- Upstream project URL/repo
- Documented upgrade procedure (download/replace steps, build commands)
- Any "Moodle changes" / custom patch section - record what each patch does and exactly where the readme says it lives in the file. You'll need this to verify the patch still applies correctly after upgrading (Step 6.1).

---

## PHASE 3: Fetch Latest & Assess

Research the target version and its security relevance together - don't lock in a version before you know what it fixes.

### Step 3.1 - Determine the Latest Upstream Version, Bump Type, and Security Relevance

Use the upstream URL from the readme (or WebSearch/WebFetch) to find the latest stable release. Compare the current version against the latest using semantic versioning (major.minor.patch) and classify the bump as **major**, **minor**, or **patch**.

In the same research pass, fetch release notes/changelog between the current and latest version (releases page or CHANGELOG) and check for known CVEs affecting the current version. Scan for security-relevant language (security, CVE, vulnerability, XSS, injection, RCE, sanitiz*, bypass, etc.).

If the bump is **major**: also read through the changes between current and latest (changelog, migration guide, "BREAKING CHANGES" sections, upstream issue/PR titles) and identify anything that looks like a breaking change - removed/renamed APIs, changed function signatures, changed defaults, dropped runtime support, etc. Summarize what you find, or explicitly note that nothing broke was identified, for Step 3.2.

Regardless of bump type, also scan the same sources for **deprecations** - APIs marked deprecated, soon-to-be-removed, or superseded by a new equivalent. Deprecations commonly land in minor bumps ahead of a later major removal, so don't skip this check just because the bump is minor or patch. Summarize what you find, or note that nothing deprecated was identified, for Step 3.2.

### Step 3.2 - Present Findings & Confirm the Target Version

Present version, bump-type, breaking-change, deprecation, and security findings together, in one message. If this is a **major version bump**, say so explicitly - don't let it pass as routine. Ask:
> "Current: [X]. Latest: [Y] ([major/minor/patch] bump). [If major:] Breaking changes found: [list, or 'none identified in the changelog - worth a manual look before assuming it's safe']. Deprecations found: [list, or 'none identified']. [Security findings, if any: details/CVEs, and whether the current version is affected/unpatched.] Upgrade to latest, or target a different version? [If security-relevant:] Should this be handled as a routine third-party library bump, or does it need to go through Moodle's security-issue process (restricted tracker visibility, security level, and per-branch backport)?"

Wait for confirmation before downloading or changing anything. Do not decide the security-process question yourself. Record the answer as `SECURITY_FLOW` (yes/no) - it determines whether Phase 8 runs after the upgrade. Also record whether any breaking changes or deprecations were found as `API_CHANGES` (yes/no) - it determines whether Phase 7 runs.

---

## PHASE 4: Apply the Upgrade

Walk the user through each step from the readme's upgrade procedure, one at a time. Show each command or file change before running it, and get explicit confirmation before anything that deletes or overwrites files.

### Step 4.1 - Fetch & Verify the New Release

Download the new release files from upstream. If the project publishes a checksum (e.g. sha256 in the release notes), compare it directly. If the only artifact is a signed tag/commit (e.g. GPG, GitHub's "Verified" badge), noting that it exists is enough - full cryptographic verification (cloning and running `gpg --verify` yourself) is not required. Either way, note (non-blocking) what verification you did or didn't do rather than silently treating the download as trusted.

### Step 4.2 - Replace Vendored Files

Replace the files per the readme's instructions with the pristine upstream release and show a diff of what changed. If Step 1.4 found no baseline file to diff against, add the new file instead and say plainly that there was nothing to diff against - don't fabricate a diff.

Do not reapply Moodle-specific patches yet - that happens after this base upgrade is committed (Phase 6), so the patch diff stays isolated in its own commit.

### Step 4.3 - Run Build Steps

First run:
```bash
nvm use
```
to switch to the Node version this Moodle checkout supports, before running any other build command.

Then run any build commands the readme specifies (e.g. grunt, npm run build). If the replaced/added files include anything under an `amd/src` directory (i.e. the library ships or touches AMD modules), also run:
```bash
grunt amd
```
to compile them to `amd/build`, even if the readme doesn't mention it.

If the required tooling isn't available in this environment, note (non-blocking) that this step is unverified/incomplete and carry it into your final summary - don't silently skip it, but don't block on it either.

### Step 4.4 - Check for License Changes

If the upstream license now differs from what's recorded in `thirdpartylibs.xml`, stop and ask:
> "The upstream license is now [new], but thirdpartylibs.xml records [old]. Should I update the `<license>`/`<licenseversion>` fields to match, or does this need review outside this conversation first?"

If the user authorizes the update inline, proceed and write it in Step 5.1. If they say it needs separate review, leave the fields unchanged and note it as an open item in your final summary.

---

## PHASE 5: Finalize the Base Upgrade

### Step 5.1 - Update thirdpartylibs.xml

Update `<version>` (and `<version_comment>`/license fields if changed in Step 4.4) for the matched entry in `THIRDPARTYLIBS_FILE` (from Step 1.3) - not necessarily the one at the codebase root. Show the diff and confirm before writing.

### Step 5.2 - Branch & Commit the Base Upgrade

Check whether the current directory is a real git repository (`git status`). If it isn't, report the commands that *would* run and stop - do not fabricate a branch or commit result.

If it is, confirm the plan, then run (using `MDL_NUMBER` from Step 1.5):
```bash
git checkout -b MDL-XXXXX-main
git add <changed files>
git commit -m "MDL-XXXXX libraries: Upgrade <Library name> to <new version>"
```
Show the commit result to the user.

This commit must contain only the pristine upstream replacement and the `thirdpartylibs.xml` version bump - no Moodle-specific patches. Those are reapplied and committed separately in Phase 6.

---

## PHASE 6: Reapply Moodle-Specific Changes (Separate Commit)

If Step 2.2 found no "Moodle changes"/custom patch section in the readme, this library has no custom patches - skip this entire phase and note in your final summary that there were none to reapply.

Otherwise, do this after the base upgrade from Phase 5 is committed, so the patch diff lands in its own commit rather than being mixed into the upstream replacement.

### Step 6.1 - Reapply Patches

For each custom patch noted in Step 2.2, locate the equivalent spot in the new file and reapply it.

If the readme's description of a patch's location or mechanism doesn't match what you actually find in the new file, **stop and ask** rather than assuming your reapplication is equivalent:
> "The readme describes the [patch name] patch as [description], but I can't find/match that in the new file. Here's what I found instead: [...]. Can you confirm how this should be reapplied, or point me to the original patch/ticket?"

### Step 6.2 - Rebuild if Needed

If any patched file is under an `amd/src` directory, run `nvm use` followed by `grunt amd`, same as Step 4.3 - even if the base upgrade didn't need a rebuild.

### Step 6.3 - Update readme_moodle

If the patch descriptions (or anything else) in the readme are now stale given how you reapplied them, propose an update to it and confirm with the user before writing.

### Step 6.4 - Commit the Moodle-Specific Changes

Show a diff of the reapplied patches (and any readme/build updates from Steps 6.2–6.3) and confirm before committing. Then run (using `MDL_NUMBER` from Step 1.5):
```bash
git add <changed files>
git commit -m "MDL-XXXXX libraries: Apply Moodle specific changes"
```
Show the commit result to the user. Keep this commit separate from the Phase 5 upgrade commit - don't fold the two together even if it's tempting to save a step.

---

## PHASE 7: Update Callers for Breaking Changes / Deprecations (only if `API_CHANGES` = yes)

Skip this phase entirely if Step 3.1 found no breaking changes or deprecations for this bump. Otherwise, run it after Phase 6 (or right after Phase 5, if this library has no Moodle-specific patches to reapply), so caller updates land in their own commit, separate from the upstream replacement and any patch reapplication.

### Step 7.1 - Search for Affected Usages

Search the Moodle codebase (not just the vendored library's own directory) for callers of the APIs flagged in Step 3.1 - grep for the old function/class/method names, old call signatures, or anything else identified as removed, renamed, or deprecated. Cover PHP, JS/AMD, and template usages as relevant to the library.

### Step 7.2 - Present Findings & Confirm

List what you found - file, line, what needs to change, and why (breaking change vs. deprecation, per Step 3.1) - and ask:
> "Found [N] usages of [old API] across [files]. These need to move to [new API] because [reason]. Update them now in a separate commit, or leave this as a follow-up (e.g. a separate MDL issue)?"

Do not touch any call site without this confirmation. If the user declines, or none were found, note it in your final summary (open item, or nothing to convert) and move on to Phase 8 (or DONE).

### Step 7.3 - Apply the Updates

For each usage the user confirmed, update the call site to the new API. Show the diff and confirm before writing, same as every other file-changing step in this skill.

### Step 7.4 - Rebuild if Needed

If any changed file is under an `amd/src` directory, run `nvm use` followed by `grunt amd`, same as Step 4.3.

### Step 7.5 - Commit the Caller Updates

Show a diff of all the changes from this phase and confirm before committing. Then run (using `MDL_NUMBER` from Step 1.5):
```bash
git add <changed files>
git commit -m "MDL-XXXXX libraries: Update <Library name> API usage for breaking changes"
```
Show the commit result to the user. Keep this commit separate from the Phase 5/6 commits.

---

## PHASE 8: Security Process (only if `SECURITY_FLOW` = yes)

Skip this phase entirely for a routine upgrade. If Step 3.2 established that this upgrade addresses a security issue and the user chose to follow Moodle's security process, continue here immediately after Phase 5, Phase 6 (if this library has custom patches), and Phase 7 (if there were breaking-change/deprecation caller updates) finish for the main/current branch.

### Step 8.1 - Confirm Proceeding with the Security Flow

Note, as part of your normal narration (non-blocking): you're now following Moodle's security process for this upgrade, based on the confirmation given in Step 3.2.

### Step 8.2 - Determine Affected Moodle Versions

Work out which supported Moodle versions/branches ship a library version affected by the security issue - check the recorded `<version>` in `thirdpartylibs.xml` on each supported branch, or ask the user which stable branches to check. List the affected branches explicitly and confirm with the user before continuing:
> "Based on [what you checked], these Moodle versions look affected: [list]. Does this match your understanding, or are there other branches to check?"

### Step 8.3 - Update the Jira Issue

Using the Atlassian MCP, change the issue type to **Bug** and set its security level. Confirm the exact security level value with the user before applying it, since this varies by tracker configuration:
> "I'll set the issue type to Bug and the security level to [proposed level]. Confirm, or tell me the correct level to use."

Apply the change only after confirmation.

### Step 8.4 - Full Upgrade for Main

This is Phase 4 + Phase 5 + Phase 6 (if applicable) + Phase 7 (if applicable), applied to the `main`/current branch - complete it now if it hasn't already run in this session. Only `main` gets the full library upgrade; affected stable branches get a fix-only patch instead (Step 8.7), not a version bump.

### Step 8.5 - Push the Main Patch

Run:
```bash
mdk push -t
```
to upload the patch for `main`. Show the result to the user.

### Step 8.6 - Get the Root Dir for the Next Affected Version

For each Moodle version confirmed as affected in Step 8.2 that hasn't been patched yet, ask:
> "What's the root directory of your [version] checkout?"

### Step 8.7 - Apply the Security Fix Only

In that checkout, the goal is a fix-only patch - do not upgrade the whole library on this branch unless step 3 below is reached and confirmed; it otherwise stays on its existing library version. Try these in order:

1. **Cherry-pick.** Cherry-pick the upstream security-fix commit(s) if they apply cleanly.
2. **Manual patch.** If the cherry-pick doesn't apply cleanly, manually patch the fix yourself - hand-adapt the upstream diff to this branch's version of the file. Show the diff before writing anything, same as Steps 4.2/6.1.
3. **Fallback: minor upgrade.** If you cannot manually patch it either (the code has diverged too far, or the fix depends on other upstream changes not present on this branch), stop and ask the user:
   > "[Library] on the [version] branch has diverged too far from upstream to hand-patch the security fix directly. The latest minor version containing the fix is [X]. Should I upgrade this branch's vendored library to [X] instead of a fix-only patch, or handle this some other way?"

   Only upgrade to that minor version with explicit confirmation, and treat it as a scoped version bump on that branch (replace files, update `thirdpartylibs.xml`'s `<version>`, rebuild if needed) - not a full Phase 4-7 run, and not an upgrade past the minor version that contains the fix.

If the fix (or the fallback minor upgrade) touches anything under `amd/src`, run `nvm use` followed by `grunt amd` in that checkout before committing, same as Step 4.3.

### Step 8.8 - Commit and Push the Patch

Commit the change (using `MDL_NUMBER`) and run:
```bash
mdk push -t
```
to upload the patch for that version. Show the commit and push result.

### Step 8.9 - Repeat

Go back to Step 8.6 for the next affected version. Repeat until every version listed in Step 8.2 has a patch pushed. Once none remain, summarize all branches patched and stop.

---

## DONE

Library upgrade complete.
