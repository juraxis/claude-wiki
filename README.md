# claude-wiki

Cross-project memory for Claude Code with semantic search. One command: `/remember`.

## The problem

Claude Code forgets everything between sessions. Its built-in memory is per-project, caps at 200 lines, and has no search. You solve a hard problem in one project, start a new project next month, and Claude has no idea it ever happened.

claude-wiki fixes that. Say `/remember` and Claude writes what you learned into a searchable wiki. Every project feeds the same knowledge base. Search finds articles by meaning, not just keywords, using a 33MB local embedding model that runs on CPU.

| | Built-in memory | claude-wiki |
|---|---|---|
| Scope | Per-project | Cross-project |
| Limit | 200 lines | Unlimited |
| Search | None | Hybrid: keyword + semantic |
| Control | Claude decides | You decide with `/remember` |

## How it works

### A lawyer three sessions into diligence

You just found that the target's key customer contract has a change-of-control termination right threatening 38% of ARR.

```
You: /remember

Claude writes:
  ~/claude-wiki/wiki/contracts/acme-coc-termination-risk.md

  # Acme Acquisition - Change of Control Risk
  Key customer contract (38% of ARR, $4.2M) contains change-of-control
  termination right in Section 12.3. No consent obtained...
```

Two weeks later, different client, similar deal:

```
You: "Review this target's customer contracts for COC risk"

Claude searches the wiki automatically
  -> finds the Acme article
  -> already knows the pattern and what to look for
```

### A developer who just burned two hours

Your auth flow breaks after deploy. Session cookie needs `SameSite=None` with `Secure` behind a reverse proxy, and the redirect URI must match the exact casing registered with the OAuth provider.

```
You: /remember
```

Three months later, different app, same deploy setup. Claude already knows. Two hours saved.

## Inspired by Karpathy's LLM Wiki

Karpathy's [llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) builds persistent knowledge bases by ingesting external docs into a wiki. We borrowed the core idea but changed the entry point: your source material is the conversation itself, not external files.

| | Karpathy's LLM Wiki | claude-wiki |
|---|---|---|
| Source | External docs, papers, repos | Your conversation with Claude |
| Ingest | Copy to `raw/`, LLM compiles | `/remember` distills in real-time |
| Search | qmd (BM25 + vector + re-ranking) | Hybrid: FTS5 keyword + semantic vector |
| Best for | Research knowledge | Working knowledge |

His system is a research librarian. This is a working notebook. They compose if you want both.

## Setup

```bash
# Install
mkdir -p ~/claude-wiki/wiki ~/.claude/skills/remember
cp wiki.py ~/claude-wiki/wiki.py
cp SKILL.md ~/.claude/skills/remember/SKILL.md
python3 ~/claude-wiki/wiki.py init

# Optional: enable semantic search (33MB model, CPU only, no GPU)
pip install fastembed sqlite-vec
```

Open Claude Code in any project. Say `/remember`. Done.

## Search

Keyword search works out of the box via SQLite FTS5 with Porter stemming and BM25 ranking. "Finetuning" matches "fine-tuned."

With `fastembed` + `sqlite-vec` installed, semantic search activates automatically. Searching "concurrent writes deadlock" finds your article about a "race condition" even though it never uses those words. The embedding model ([bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5)) is 33MB, runs on CPU in ~50ms per article, and needs no GPU, no PyTorch, no server.

Both modes run together. Keyword first, then semantic fills in what keyword missed. If the packages are not installed, it falls back to keyword-only silently.

```bash
python3 ~/claude-wiki/wiki.py search "change of control"
python3 ~/claude-wiki/wiki.py search "indemnity structure" --json
python3 ~/claude-wiki/wiki.py stats
```

## What gets saved

Claude writes markdown articles organized by category. You never touch the structure.

```
~/claude-wiki/
  wiki.py              # one file, Python stdlib + optional fastembed/sqlite-vec
  wiki.db              # search index (disposable, rebuildable from articles)
  INDEX.md             # table of contents Claude maintains
  wiki/
    contracts/
      indemnity-cap-carveouts.md
      termination-for-convenience.md
    litigation/
      discovery-scope-preservation.md
    debugging/
      oauth-redirect-behind-proxy.md
```

Real articles with tables, exact numbers, specific commands, reasoning. Not chat logs.

## Use cases

**Legal**: diligence findings that carry across deals, clause review patterns, jurisdictional research, settlement positions with exact dollar figures.

**Technical**: architecture decisions and why, deployment fixes with the exact config, API quirks that took hours to find, evaluation results with real numbers.

**Anything multi-project**: you learn it once, `/remember`, it is there forever.

## Commands

| Command | What it does |
|---|---|
| `wiki.py init` | Create the database |
| `wiki.py index <file>` | Index one markdown file |
| `wiki.py index-all` | Re-index all files in wiki/ |
| `wiki.py search <query>` | Search (human-readable) |
| `wiki.py search <query> --json` | Search (JSON for Claude) |
| `wiki.py stats` | Show article counts |

## License

MIT
