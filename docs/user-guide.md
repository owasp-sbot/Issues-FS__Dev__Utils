# Issues-FS Dev Utils User Guide

**Version:** v0.1.0
**Date:** 2026-02-11

## Overview

`issues_fs_dev_utils` provides two cross-repo developer productivity tools for the Issues-FS ecosystem:

- **context-dump** — Gathers relevant source code and documentation for an LLM coding session, organized by topic
- **diff-dump** — Aggregates diffs across all 17+ submodules between two git references into a single output

Both tools are designed to run from anywhere inside the `Issues-FS__Dev` orchestration repo (including from within submodules) and support output to stdout, clipboard, or file.

## Installation

Install from PyPI:

```bash
pip install issues-fs-dev-utils
```

Or with Poetry:

```bash
poetry add issues-fs-dev-utils
```

## Context Dump

Collects files relevant to a topic across the entire ecosystem, formatted for pasting into an LLM context window. Uses a Librarian-maintained `topic_map.json` that maps 14 topics to their associated documentation, code patterns, and modules.

### Listing Available Topics

```bash
python -m issues_fs_dev_utils.context_dump topics
```

Output:

```
Topic                Description
-------------------- --------------------------------------------------
cli                  Command-line interface, Typer commands
comment              Comments on issues/nodes
graph                Graph data model, nodes, edges, MGraph-DB
lexicon              Lexicon architecture, semantic vocabulary, anchor nodes
link                 Links/edges between nodes, relationships
node                 Issue nodes, node types, node CRUD
onboarding           New developer onboarding, project overview
path                 Path resolution, file paths, directory navigation
roles                Role-based agent coordination, all 10 roles
schema               Data schemas, Type_Safe models, request/response objects
service              FastAPI REST API server
storage              Memory-FS abstraction, storage backends
testing              Testing patterns, pytest, test structure
ui                   Web UI, frontend components
```

### Gathering Context for a Topic

Gather all relevant files for the `graph` topic:

```bash
python -m issues_fs_dev_utils.context_dump gather graph
```

This collects:
1. **Always-included docs** — Core architecture and thinking-in-graphs documents
2. **Type_Safe docs** — Safe primitives and formatting guides (for coding sessions)
3. **Topic docs** — Documents specific to the topic
4. **Topic code** — Source files matched by glob patterns and grep patterns from configured modules

### Options

Exclude source code (docs only):

```bash
python -m issues_fs_dev_utils.context_dump gather storage --no-code
```

Exclude documentation (code only):

```bash
python -m issues_fs_dev_utils.context_dump gather cli --no-docs
```

Include role definition files:

```bash
python -m issues_fs_dev_utils.context_dump gather roles --include-roles
```

Exclude Type_Safe reference docs:

```bash
python -m issues_fs_dev_utils.context_dump gather graph --no-types
```

Limit the number of files collected:

```bash
python -m issues_fs_dev_utils.context_dump gather testing --max-files 20
```

### Output Modes

Print to stdout (default):

```bash
python -m issues_fs_dev_utils.context_dump gather graph
```

Copy directly to clipboard:

```bash
python -m issues_fs_dev_utils.context_dump gather graph -o clipboard
```

Write to a file:

```bash
python -m issues_fs_dev_utils.context_dump gather graph -o file
python -m issues_fs_dev_utils.context_dump gather graph -o file -f my-context.txt
```

When using `-o file` without `-f`, the output file is named `context_dump__<topic>.txt`.

### Unknown Topics

If a topic is not in the topic map, context-dump falls back to a keyword search across all modules:

```bash
python -m issues_fs_dev_utils.context_dump gather "Memory_FS"
```

This greps for the keyword in all `.py` and `.md` files under `modules/`.

## Diff Dump

Generates a cross-repo diff summary between two git references (tags, branches, or commits). Shows which submodules changed, their commit logs, and diff statistics.

### Basic Usage

Compare a tag to HEAD:

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0
```

Compare two tags:

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 v0.41.0
```

Compare two branches:

```bash
python -m issues_fs_dev_utils.diff_dump origin/main origin/dev
```

### Options

Include full diff content (not just stats):

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 --full-diff
```

Exclude file change statistics:

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 --no-stats
```

Only show module repos (skip role/dev/human repos):

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 --modules-only
```

### Output Modes

Print to stdout (default):

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0
```

Copy to clipboard:

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 -o clipboard
```

Write to file:

```bash
python -m issues_fs_dev_utils.diff_dump v0.40.0 -o file
python -m issues_fs_dev_utils.diff_dump v0.40.0 v0.41.0 -o file -f release-diff.txt
```

When using `-o file` without `-f`, the output file is named `diff_dump__<from>..<to>.txt`.

### Example Output

```
# Cross-Repo Diff: v0.40.0 .. HEAD
Repository: Issues-FS__Dev
================================================================================

## Main Repository Commits
----------------------------------------
abc1234 Update submodule pointers for v0.41.0

## Main Repository Diff Stats
----------------------------------------
 modules/Issues-FS | 2 +-
 modules/Issues-FS__CLI | 2 +-
 2 files changed, 2 insertions(+), 2 deletions(-)

## Submodule: modules/Issues-FS
   v0.40.0 -> v0.41.0
----------------------------------------
### Commits
def5678 Add Memory-FS abstraction layer
789abcd Refactor storage backends

### Diff Stats
 issues_fs/storage/Memory_FS.py | 120 +++++++++++++++
 1 file changed, 120 insertions(+)

================================================================================
Total submodules changed: 2 / 17
```

## How It Works

### Root Discovery

Both tools use `find_dev_repo_root()` from `Dev_Repo.py` to locate the `Issues-FS__Dev` orchestration repo. It walks up the directory tree looking for a `.gitmodules` file with 5+ submodule entries, which distinguishes the Dev repo from individual submodules that may also have `.gitmodules`.

This means you can run the tools from:
- The `Issues-FS__Dev` root
- Inside any submodule (e.g., `modules/Issues-FS/`)
- Any nested directory within the tree

### Topic Map

The `topic_map.json` file is maintained by the Librarian role and defines:

| Key | Purpose |
|-----|---------|
| `always_include` | Documents included in every context dump |
| `type_safety` | Type_Safe reference docs for coding sessions |
| `topics.<name>.docs` | Documentation files for the topic |
| `topics.<name>.modules` | Which submodules to search for code |
| `topics.<name>.code_globs` | File glob patterns to match |
| `topics.<name>.code_patterns` | Grep patterns to search for |
| `topics.<name>.role_files` | Associated ROLE.md files |

### Clipboard Support

Both tools detect available clipboard commands in order: `xclip`, `xsel`, `pbcopy` (macOS). If none are available, clipboard mode falls back to stdout with a warning.

## Workflows

### Starting an LLM Coding Session

```bash
# 1. See what topics are available
python -m issues_fs_dev_utils.context_dump topics

# 2. Gather context for the area you're working on
python -m issues_fs_dev_utils.context_dump gather storage -o clipboard

# 3. Paste into your LLM session
```

### Reviewing Changes Before a Release

```bash
# 1. See what changed since the last release
python -m issues_fs_dev_utils.diff_dump v0.40.0 --modules-only

# 2. Get full diffs for detailed review
python -m issues_fs_dev_utils.diff_dump v0.40.0 --full-diff -o file

# 3. Share the summary
python -m issues_fs_dev_utils.diff_dump v0.40.0 -o clipboard
```

### Preparing a Handoff Between Agent Sessions

```bash
# Gather context for the next session's topic
python -m issues_fs_dev_utils.context_dump gather service -o file -f handoff-context.txt

# Include the recent diff for awareness
python -m issues_fs_dev_utils.diff_dump HEAD~5 -o file -f handoff-diff.txt
```

## Troubleshooting

### "Could not find Issues-FS__Dev root"

You are running the tool from outside the `Issues-FS__Dev` directory tree. Navigate to any directory inside the Dev repo or a submodule:

```bash
cd /path/to/Issues-FS__Dev
python -m issues_fs_dev_utils.context_dump topics
```

### "No topics configured in topic_map.json"

The `topic_map.json` file is missing or empty. It should be at:

```
issues_fs_dev_utils/context_dump/topic_map.json
```

### Empty context dump output

The topic's configured files may not exist at the expected paths. Check that submodules are initialized:

```bash
git submodule status
```

### Clipboard not available

Install a clipboard utility:

```bash
# Linux
sudo apt install xclip
# or
sudo apt install xsel

# macOS — pbcopy is built-in
```
