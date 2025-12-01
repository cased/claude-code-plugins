# Kit CLI Reference

Quick-access sheet for the most common `kit` CLI workflows. Pair this file with `SKILL.md` so you can copy the exact command without rereading help text. For additional subcommands or updates, see the official docs: https://kit.cased.com

## Install & versioning

```bash
pipx install cased-kit         # recommended (isolated env)
# or
pip install --upgrade cased-kit

kit --version                  # verify install
kit --help                     # top-level overview
```

Semantic search requires `sentence-transformers`. If `kit search-semantic` errors, run:

```bash
pip install "sentence-transformers>=2.6"
```

## Structure & metadata

| Command | Purpose | Common flags |
| --- | --- | --- |
| `kit git-info PATH` | Current SHA, branch, remote URL | `--ref <sha>` |
| `kit file-tree PATH` | Prints directory tree (optionally scoped) | `--path src`, `--output tree.json`, `--ref` |
| `kit file-content PATH file1 file2` | Cat selected files with repo-aware filtering | *(none)* |
| `kit index PATH` | Build full repository inventory (files, symbols, docs) | `--output index.json` |

## Symbols & code extraction

| Command | Purpose | Useful options |
| --- | --- | --- |
| `kit symbols PATH` | Extract functions/classes with line numbers | `--file path/to/file`, `--format table|json|names`, `--output file` |
| `kit chunk-symbols PATH file` | Split file by symbol for targeted quoting | `--output chunks.json` |
| `kit usages PATH SymbolName` | Definitions + references for a symbol | `--type function|class`, `--output usages.json` |
| `kit chunk-lines PATH file` | Line-based chunks when symbol parsing is noisy | `--max-lines 80`, `--output chunks.json` |

## Search & retrieval

| Command | Use case | Notes |
| --- | --- | --- |
| `kit search PATH "pattern"` | Regex/text search with glob filters | `--pattern "src/**/*.ts"`, `--output hits.json` |
| `kit grep PATH "literal"` | Fast literal grep with include/exclude rules | `--include "*.py"`, `--exclude "*.test.js"`, `--directory src`, `--max-results 200` |
| `kit search-semantic PATH "question"` | Vector search over code/context chunks | `--top-k 10`, `--chunk-by symbols|lines`, `--format table|json|plain`, `--embedding-model all-MiniLM-L6-v2`, `--persist-dir .kit-index` |
| `kit context PATH file line` | Surrounding code window (default ±20 lines) | `--output context.json` |
| `kit file-content PATH file1 file2` | Pull whole files verbatim | Good for manifest snapshots |

## Dependency + architecture analysis

| Command | Purpose | Helpful flags |
| --- | --- | --- |
| `kit dependencies PATH --language python` | Build import graph, detect cycles | `--format json|dot|graphml|adjacency`, `--visualize`, `--viz-format png|svg`, `--module package.sub`, `--include-indirect`, `--cycles`, `--llm-context`, `--output deps.dot` |
| `kit dependencies PATH --language terraform` | Map Terraform modules/resources | Same flags as above |
| `kit export PATH symbol-usages out.json --symbol Foo` | Saved subsets for later ingestion | `--data_type index|symbols|file-tree|symbol-usages` |

## Reviews & higher-level workflows

Even though this skill focuses on exploration, remember kit also has workflow helpers:

| Command | Summary |
| --- | --- |
| `kit review path --diff <file>` | Run AI-powered review over a local diff or GitHub PR. Useful when packaging findings. |
| `kit summarize path --pr <url>` | Quick synopsis of a PR using kit’s context retrieval. |
| `kit commit path` | Intelligent commit generator; complementary to `commit-writer` skill. |

## Cache management

Reuse previously built embeddings/indexes to keep commands snappy on large repos.

```bash
kit cache warm /repo/path            # precompute indexes
kit cache list                       # show cached repos
kit cache clear /repo/path           # drop stale state
```

Set `KIT_CACHE_DIR=/tmp/kit-cache` (or similar) to keep caches outside the repo.

## Troubleshooting

- **Permission errors:** ensure repo path is readable; run `ls -al PATH`.
- **Graphviz missing:** install via `brew install graphviz` (macOS) or `apt-get install graphviz`.
- **Large outputs:** pipe through `jq`, `rg`, or redirect to files (`--output tmp.json`) then summarize in Claude.
- **Remote refs:** add `--ref main` when inspecting a historical tree or tag without checking it out locally.

Keep this page open alongside CLI sessions; update it whenever new kit releases add commands. The docs highlight fresh subcommands under “Changelog”.***

