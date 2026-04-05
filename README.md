# claude-wiki

Cross-project memory for Claude Code with semantic search. One command: `/remember`.

## The problem

Claude Code forgets everything between sessions. You spend an hour figuring something out, close the terminal, and next time Claude starts from zero.

Its built-in memory does not solve this. Here is what it actually does:

- It saves small notes to `~/.claude/projects/<your-project>/memory/`
- Those notes only load when you are in that specific project
- It truncates at 200 lines, so longer findings get cut off
- There is no search. Claude reads the files top to bottom and hopes the relevant one is near the top
- Claude decides what to save automatically, so you cannot control what sticks

That works for simple per-project preferences like "use tabs not spaces." It does not work when you need knowledge to travel between projects, or when the thing you learned is too detailed for 200 lines, or when you need to find something you saved three months ago.

claude-wiki fixes that.

| | Built-in memory | claude-wiki |
|---|---|---|
| Scope | Per-project only | Cross-project, one wiki for everything |
| Size limit | Truncates at 200 lines | Unlimited articles, no cap |
| Search | None, reads files in order | Hybrid: keyword (FTS5) + semantic (33MB model) |
| What gets saved | Claude decides automatically | You decide with `/remember` |
| Retrieval | Loaded only in that project | Searchable from any project, any session |
| Format | Short memory notes | Full articles with tables, numbers, reasoning |

## How it works

### A lawyer building a case across sessions

You are reviewing a target's contracts and find that a change-of-control clause threatens 38% of ARR. That kind of finding matters beyond this one deal.

```
You: /remember

Claude writes:
  ~/claude-wiki/wiki/contracts/coc-termination-risk.md

  # Change of Control - Termination Risk Pattern
  Key customer contract (38% of ARR) contains change-of-control
  termination right. No consent obtained. Must be closing
  condition or renegotiated pre-sign...
```

Two weeks later, different client, similar deal structure:

```
You: "Review this target's customer contracts for COC risk"

Claude searches the wiki automatically
  -> finds the article from last deal
  -> already knows the pattern, the risk, and what to flag
```

The lesson carried over. You never re-explained it.

### A vibe-coder who keeps hitting the same walls

Every few projects, you run into something that takes real time to solve. A deploy config that silently breaks in production. A database migration that works locally but fails on the remote. An API that behaves differently on v2 vs v3 with no mention in the changelog.

```
You: "That took way too long to figure out"
You: /remember

Claude writes:
  ~/claude-wiki/wiki/debugging/postgres-migration-remote-timeout.md
```

Next project, same stack. Claude searches the wiki before you even ask, finds the article, and skips the debugging entirely. The more you use it, the fewer hours you waste re-solving problems you already solved.

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

Both modes run together. Keyword first, then semantic fills in what keyword missed. Falls back to keyword-only silently if the packages are not installed.

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
      coc-termination-risk.md
    litigation/
      discovery-scope-preservation.md
    debugging/
      postgres-migration-remote-timeout.md
```

Real articles with tables, exact numbers, specific commands, reasoning. Not chat logs.

## Use cases

**Legal professionals**: diligence findings that carry across deals, clause patterns that worked (or did not), jurisdictional research you do not want to redo, settlement positions with exact figures and deadlines.

**Developers**: the deploy fix that took three hours and you never want to debug again, the API behavior that is not in the docs, architecture decisions and the reasoning behind them.

**Anyone working across projects**: you learn it once, `/remember`, Claude has it forever.

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
