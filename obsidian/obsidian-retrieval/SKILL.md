---
name: obsidian-retrieval
description: Graph-based retrieval algorithm for navigating the Obsidian vault as a knowledge graph. Use when an agent needs to find, retrieve, or reason about knowledge stored in the vault.
---

# Vault Retrieval Algorithm

The vault is a **knowledge graph**, not a file system. Notes are nodes, wikilinks are edges, Category pages are hub nodes. Never list directories or glob for files — enter the graph and traverse.

## Related skills

- **obsidian-principles** — Vault structure, properties, organization rules
- **obsidian-cli** — CLI commands for vault operations

---

## Algorithm: ENTER → TRAVERSE → EXTRACT

### Phase 1: ENTER

Pick the highest-precision entry point available:

1. **Direct name** — you know the note name → open it (O(1))
2. **Category page** — you know the type → open `Categories/Books.md` etc. (O(1))
3. **Related entity** — you know a connected note → open it, then traverse its links
4. **Property query** — you know a property value → grep frontmatter for `author`, `topics`, etc. (O(n))
5. **Full-text search** — last resort → search body text, then switch to traversal once you hit a node (O(n))

### Phase 2: TRAVERSE

Follow edges to reach your target. Rarely needs more than 2-3 hops.

| Edge type              | Source                                  | Direction |
| ---------------------- | --------------------------------------- | --------- |
| Body wikilinks         | `[[Name]]` in note body                 | Forward   |
| Property wikilinks     | `author: - "[[Person]]"` in frontmatter | Forward   |
| Backlinks              | Other notes linking TO this note        | Reverse   |
| Category co-membership | Notes sharing the same `categories`     | Lateral   |
| Shared properties      | Notes sharing author, topic, tag, etc.  | Lateral   |

**Strategies:**
- **Forward** — follow outgoing links to explore a note's neighborhood
- **Reverse** — find all notes that link TO a note (backlinks via grep `[[Name]]`)
- **Hub** — use a Category page to scan all members, then filter by properties
- **Property-hop** — jump via shared values: Note A → shared author → Note B
- **Expand** — widen by 1 hop at a time if initial traversal misses

### Phase 3: EXTRACT

1. **Frontmatter first** — YAML properties answer 80% of metadata questions (categories, type, status, rating, author, topics, url, created)
2. **Body second** — narrative content, wikilinks as relationships, headers as structure, blockquotes as highlights

---

## Common Retrieval Patterns

**"What do I know about X?"** → Open X.md → read frontmatter → read body → follow backlinks → follow outgoing links

**"All books/projects about Y"** → Category page → scan members → filter by topics/tags containing Y

**"What's connected to this note?"** → Open note → collect outgoing links + backlinks → read each neighbor's frontmatter

**"Knowledge gaps"** → Scan all wikilinks → find ones with no matching file → rank by frequency

**"Summarize domain Z"** → Category or tag entry → collect notes → extract frontmatter + body key points per note → traverse 1 hop for context → synthesize with citations

**"Relationship between A and B"** → Open A → check if links to B (1-hop) → if not, check A's neighbors for B (2-hop) → if not, check backlink overlap

---

## Principles

1. **Enter the graph, don't search the filesystem** — category pages and known names are your index
2. **Read frontmatter before body** — structured metadata first, narrative second
3. **Prefer traversal over search** — follow links from a known node rather than running a new search
4. **Backlinks are the most powerful edge** — a person note knows every book, project, and idea that mentions them
5. **Expand incrementally** — start narrow (1 note), widen by 1 hop if needed, never load everything
6. **Unresolved links are knowledge gaps** — `[[missing link]]` = mentioned but unexplored
7. **Write back what you learn** — new connections, inferred relationships, and filled gaps go back into the vault
