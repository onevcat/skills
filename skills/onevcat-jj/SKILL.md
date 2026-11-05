---
name: onevcat-jj
description: "Use jj (Jujutsu) for local version control in jj-managed repositories. Activate when the user or project requests jj, or version-control work is needed in a repository where jj root identifies the current workspace."
---

# jj (Jujutsu) — Version Control for Agent Workflows

jj is a Git-compatible version control tool. Use jj locally; GitHub and collaborators see ordinary Git commits and branches.

## Version and Repository Checks

This guidance targets **jj 0.46.0**, which requires Git **2.42.0 or newer**. Check the installed version and use its help when behavior differs:

~~~bash
jj --version
git --version
jj root
jj --no-pager status
jj --no-pager log -n 10
~~~

Run these from the intended workspace. A .jj directory may be in an ancestor. In a plain Git worktree nested under a jj repository, compare jj root with git rev-parse --show-toplevel before any snapshotting or mutation: jj may resolve to the parent workspace. Do not modify that parent accidentally.

In colocated mode, .git may be a directory or a file (including jj 0.46 child workspaces). Prefer jj for local mutations; read-only Git interoperability checks are useful. Do not introduce jj into a Git-only repository unless requested.

## Core Mental Model

- **The working copy is a change**, referenced as @. File edits are snapshotted when a snapshotting jj command runs, not on every file write. Run jj status after edits before relying on recovery history. Ignored, untracked, or oversized new files may not be included; inspect warnings.
- **No staging area.** There is no git add step.
- **No stash needed.** jj new starts a child change and preserves the previous work.
- **Changes are mutable.** Editing an ancestor automatically rebases descendants, potentially creating conflicts.
- **Change IDs and commit IDs differ.** Change IDs survive rewrites; commit hashes change. Use unique change-ID prefixes for normal work. Hidden/divergent revisions can require the xyz/0, xyz/1 syntax or an exact commit ID.
- **Bookmarks are explicit references.** They do not automatically advance to newly created child changes. @- means the parent of @, not necessarily the work to publish.
- **Immutability comes from configuration and revsets**, not simply whether a commit was pushed. Do not bypass it with --ignore-immutable merely to make a command succeed.

## Non-Interactive Agent Commands

Use --no-pager for output commands and explicit messages for commands that may open an editor. Do not change the user's global pager/editor settings to accommodate automation.

~~~bash
jj --no-pager status
jj --no-pager diff
jj --no-pager diff -r <change>
jj describe -m "feat: add auth module"
jj describe -r <change> -m "new description"
jj new                         # preserve current change; start an empty child
jj new <change>                # start a child of a specific change
jj commit -m "feat: add auth"   # describe current change, then start an empty child
jj edit <change>               # continue editing an existing mutable change
~~~

For multiline descriptions use jj describe --stdin with a quoted heredoc, or a message file outside the repository. A file inside the repository could be snapshotted into the change.

Respect the user's scope and project approval rules. Loading this skill does not authorize creating commits, rewriting history, deleting workspaces, pushing, or opening PRs.

## Setup

Initialize only when requested, from the intended Git checkout:

~~~bash
jj git init --colocate
jj bookmark track master --remote origin
~~~

Replace master and origin with the actual branch and remote. The old bookmark@remote form for bookmark track/untrack is deprecated; use the separate --remote argument. Revision references such as master@origin remain valid.

Use jj config commands rather than assuming repository configuration lives under .jj: since 0.38, repository/workspace configuration is stored externally. In 0.45+, config set/edit/unset --user targets the first loaded user config; use --file <path> when a specific file matters.

## Reorganize History

Inspect the target and diff before a rewrite, then verify status and the graph afterward.

~~~bash
jj split <paths> -m "feat: first logical piece"
jj squash --from <source> --into <destination> --use-destination-message
jj rebase -s <source> -d <destination>  # source and descendants
jj rebase -r <change> -d <destination> # selected change
jj abandon <change>
~~~

Use explicit filesets plus -m for non-interactive splitting. Bare jj split opens a diff editor; in 0.46, interactive splitting uses one description-editor session for the resulting commits. Avoid bare squash when it could open an editor; supply -m or --use-destination-message.

**jj abandon discards the change's own diff** and rebases descendants onto its parents. It does not absorb the diff into its parent. Bookmarks on abandoned changes normally move to their parents; abandoning is not a reliable way to delete a bookmark or a remote branch. Inspect the result and delete the bookmark explicitly when intended.

## Remote Interaction

~~~bash
jj git fetch
jj rebase -d <default-branch>
jj bookmark create my-feature -r <completed-change>
jj git push --bookmark my-feature
~~~

Resolve the actual default branch; do not assume main or master exists. New bookmarks can be pushed explicitly with --bookmark; --allow-new was removed in 0.42. Describe every commit in the range being pushed, and inspect for conflicts and private commits.

To update a PR, point its bookmark at the change that actually contains the work:

~~~bash
jj new my-feature -m "fix: address review feedback"
# edit files, then snapshot and inspect
jj --no-pager status
jj --no-pager diff
jj bookmark set my-feature -r @
jj git push --bookmark my-feature
~~~

If jj commit or jj new has already created an empty child after completing the work, the completed change may instead be @-. Inspect the graph before choosing. Editing the existing bookmarked change rewrites it in place; do not move the bookmark back to @- afterward. Rewriting published history requires the appropriate user/project authorization.

For intentional branch deletion, use jj bookmark delete <name>, inspect remote deletions, and only then jj git push --deleted. That flag may publish multiple pending deletions.

Since 0.44, tags have remote tracking state: fetched tags are tracked by default, and tracked tags participate in pushes. jj git push --all includes tags as well as bookmarks. Prefer an explicit bookmark for a branch-only push. In 0.46, --remote can be repeated and accepts patterns; inspect targets before pushing to multiple remotes.

In colocated workspaces, jj git import/export are disabled by default since 0.44. Do not prescribe them as routine synchronization or bypass the restriction blindly; inspect jj status and installed help when diagnosing Git interoperability.

## Parallel Workspaces

~~~bash
jj workspace add ../workspace-name
jj workspace list
jj git colocation status
~~~

Each workspace has its own directory and @, but shares repository history, bookmarks, and the operation log. Give concurrent writers separate workspaces and changes; do not jj edit a change another workspace is actively editing.

In 0.46, workspace add can create a Git worktree alongside the jj workspace. In a colocated workspace this is the default when git.colocate is true. Use --colocate or --no-colocate explicitly when the mode matters. Child workspaces support jj git colocation status/enable/disable.

Before removing a workspace, inspect and snapshot its final edits inside that workspace. With authorization, jj workspace remove <name> snapshots its working-copy state and removes its directory. This deletes on-disk files; ignored/untracked files may not be recoverable from jj. jj workspace forget <name> unregisters it (and removes its associated Git worktree when present) but does not snapshot another workspace's pending edits. Do not substitute an unchecked rm -rf cleanup recipe.

## Undo, Conflicts, and Recovery

~~~bash
jj --no-pager op log -n 10
jj undo
jj redo
~~~

In 0.46, undo/redo refuse operations performed in another workspace by default. --allow-cross-workspace overrides this guard only when intentionally authorized after inspecting the shared operation history. Never assume undo is harmless in a shared repository.

jj op restore <operation-id> restores repository state and can affect other workspaces and bookmarks. Prefer recovering only the affected change when others are working:

~~~bash
jj --no-pager evolog -r <change> -p
jj restore --from <old-commit> --to <change> <paths>
~~~

Operation history is a recovery aid, not an indefinite backup; unsnapshotted or ignored files and remote side effects are not restored by undo.

Conflicts are recorded in commits; a rebase can complete with conflicts. Inspect jj status, edit conflict markers to the intended content, then snapshot and verify. Avoid interactive merge tools in unattended runs. For divergent changes, inspect the versions and consider jj converge (added in 0.45); use its non-interactive mode only after checking installed help and reviewing the intended result.

## Common Agent Patterns

- **Next task:** describe the completed change, then jj new for the next unit.
- **Interrupt and resume:** jj new <base> for urgent work, then jj edit <previous-change> to resume. Verify each target first.
- **Split after the fact:** select paths with jj split <paths> -m "message"; inspect both resulting changes.
- **Plan complex work:** create separate changes only when useful and authorized; do not manufacture empty commit sequences by default.

## Further Help

~~~bash
jj --no-pager help
jj --no-pager help <command>
jj --no-pager help -k revsets
~~~

For advanced templates/configuration, note the 0.46 changes: list get/first/last return Option values, and fix.tools.<name>.line-range-args is an array replacing line-range-arg. Do not copy older snippets without checking their types and options.

Official documentation: https://docs.jj-vcs.dev/
Version changes: https://github.com/jj-vcs/jj/blob/v0.46.0/CHANGELOG.md
