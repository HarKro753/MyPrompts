---
name: obsidian-principles
description: Personal Obsidian vault design principles, structure, templates, bases, and style rules. Use when creating, organizing, restructuring, or migrating notes in the Obsidian vault, when the user asks about vault organization, when creating new templates or bases, or when deciding where a note should go. Also use when writing frontmatter properties, choosing categories, or composing templates.
---

# Obsidian Vault Principles

Personal vault design system inspired by Steph Ango's bottom-up approach. This skill defines the organizational philosophy, folder structure, property conventions, template composition rules, Bases patterns, and personal style rules for the vault.

## Related skills

- **obsidian-cli** — CLI commands to interact with the vault. Use when executing vault operations.
- **obsidian-bases** — Full Bases `.base` file syntax. Use when writing or editing `.base` files.
- **obsidian-markdown** — Obsidian Flavored Markdown syntax. Use when writing note content.

---

## Core Philosophy

The vault follows a **bottom-up, chaos-embracing** approach to note-taking:

1. **Folders are for infrastructure, not organization** — never categorize by folder
2. **Properties are for meaning** — `categories`, `type`, `tags` do all classification
3. **Links are for connections** — wikilinks in body text and in properties create the knowledge graph
4. **Bases are for views** — `.base` files surface notes dynamically, embedded directly in notes
5. **Templates are for speed** — every replicable pattern gets a template; compose, don't duplicate
6. **Link everything, even what doesn't exist yet** — unresolved links are knowledge gaps and breadcrumbs for future connections

### The golden rule

> Never ask "where should I put this?" — it goes in the root (your world) or References (the external world). Add properties. Done.

---

## Vault Folder Structure

```
Obsidian Vault/
├── References/          ← External world (books, people, podcasts, technologies, others' projects)
├── Clippings/           ← Things other people wrote (articles, essays, web clips)
├── Attachments/         ← Images, PDFs, audio, video
├── Templates/           ← Note templates (composable)
│   └── Bases/           ← .base files (query engines)
├── Categories/          ← Category overview pages (each embeds a base)
├── Daily/               ← Daily notes (YYYY-MM-DD.md)
├── *.md                 ← YOUR world: projects, ideas, truths, journal, evergreen notes
```

### Placement rules

| If the note is about...                                       | Put it in...   | Reason                |
| ------------------------------------------------------------- | -------------- | --------------------- |
| Your project, idea, truth, course, journal, evergreen insight | Root `/`       | It's your world       |
| A book, person, podcast, technology, someone else's project   | `References/`  | It exists outside you |
| An article or essay someone else wrote                        | `Clippings/`   | Someone else wrote it |
| An image, PDF, audio, or video file                           | `Attachments/` | Binary assets         |

**Root rule:** If it's in the root, it's something you wrote or relates directly to you.

---

## Aggressive Linking — Unresolved Links as Knowledge Gaps

**Link every meaningful entity in your text, even when no note exists for it yet.** Unresolved links (links that point to non-existent notes) are not errors — they are breadcrumbs, knowledge gaps, and future connection points.

### What to link

- **People** — anyone mentioned by name
- **Places** — cities, countries, venues, landmarks
- **Quotes** — memorable phrases worth their own evergreen note
- **Concepts** — ideas, theories, patterns, even if you haven't written about them yet
- **Projects and tools** — anything with a proper name
- **Events** — conferences, meetings, moments in time

### Example

Raw text:

> Elon Musk wrote to me that he really likes the quote "We live and die only once" on Saturn

Properly linked:

> [[Elon Musk]] wrote to me that he really likes the quote [[We live and die only once]] on [[Saturn]]

Even if `We live and die only once` and `Saturn` don't have notes yet, they now show up as **unresolved links** in the vault graph. They become visible connection points. When you later create a note about Saturn, every past mention is already linked. The quote becomes an evergreen note waiting to be born.

### Why this matters

- **Knowledge gaps become visible** — unresolved links show you what you've mentioned but haven't explored yet
- **Connections emerge over time** — when you eventually create the note, all past references are already linked
- **The graph grows organically** — your vault builds a web of meaning with zero upfront planning
- **Backlinks work retroactively** — the moment you create `[[Saturn]]`, every note that mentioned it appears in its backlinks

### Rules for aggressive linking

1. **Link the first mention** of any entity in a note's body text
2. **Don't be afraid of red/unresolved links** — they are features, not bugs
3. **Quotes worth remembering get their own link** — `[[Next time is next time, now is now]]` becomes an evergreen note
4. **Properties use links too** — `author: - "[[Person Name]]"` even if the person note doesn't exist yet

---

## Properties System

### Categories use wikilinks

The `categories` property is a **list of wikilinks** pointing to Category pages. This is the primary classification mechanism.

```yaml
categories:
  - "[[Books]]" # points to Categories/Books.md
```

A note can have multiple categories:

```yaml
categories:
  - "[[Ideas]]"
  - "[[Projects]]" # an idea that became a project
```

### References in properties use wikilinks

Any property that references another entity uses `[[wikilinks]]`, even if the target note doesn't exist yet:

```yaml
author:
  - "[[Peter Thiel]]"
show:
  - "[[Founders]]"
guests:
  - "[[Elon Musk]]"
org:
  - "[[Obsidian]]"
genre:
  - "[[Sci-fi]]"
topics:
  - "[[Emergence]]"
```

### The `type` property enables composition

`type` is a list of wikilinks that represents **roles** layered on top of the base category:

```yaml
# An Author is a Person with the Authors role
categories:
  - "[[People]]"
type:
  - "[[Authors]]"
```

### Standard properties

These property names are reused across categories for cross-cutting queries:

| Property     | Type         | Used by                     | Description                                               |
| ------------ | ------------ | --------------------------- | --------------------------------------------------------- |
| `categories` | list (links) | All notes                   | Primary classification via `[[wikilinks]]`                |
| `type`       | list (links) | Composed notes              | Role/sub-type for template composition                    |
| `status`     | text         | Projects, Ideas, People     | Current state (active, paused, finished, abandoned, etc.) |
| `tags`       | list (text)  | All notes                   | Secondary classification, always pluralized               |
| `rating`     | number       | Books, References, Podcasts | 1-7 scale (see rating system)                             |
| `author`     | list (links) | Books, Clippings            | Link to person who created it                             |
| `genre`      | list (links) | Books, Movies, Shows        | Shared across all media types                             |
| `topics`     | list (links) | Books, Episodes, Ideas      | Subject matter links                                      |
| `url`        | text         | Projects, References        | External URL                                              |
| `created`    | date         | All notes                   | Creation date (YYYY-MM-DD)                                |
| `start`      | date         | Projects, Trips             | Start date                                                |
| `published`  | date         | Episodes, Posts             | Publication date                                          |
| `last`       | date         | Books                       | Last interaction date                                     |

### Property rules

- **Default to `list` type** if there's any chance a property might contain more than one value in the future
- **Short property names** are faster to type: `start` not `start-date`, `org` not `organization`
- **Reuse property names across categories** so you can query across types (e.g., `genre` works for books, movies, and shows)
- **Use `[[wikilinks]]` in properties** for anything that references another entity, even if the note doesn't exist yet

---

## Rating System (1-7 Scale)

| Rating | Meaning   | Description                                                  |
| ------ | --------- | ------------------------------------------------------------ |
| 7      | Perfect   | Must try, life-changing, go out of your way to seek this out |
| 6      | Excellent | Worth repeating                                              |
| 5      | Good      | Don't go out of your way, but enjoyable                      |
| 4      | Passable  | Works in a pinch                                             |
| 3      | Bad       | Don't do this if you can                                     |
| 2      | Atrocious | Actively avoid, repulsive                                    |
| 1      | Evil      | Life-changing in a bad way                                   |

---

## Categories

Each category has a **page** in `Categories/` that embeds its base. The category page is the target of `[[wikilinks]]` in frontmatter.

### Category page pattern

```yaml
---
tags:
  - categories
---
```

Followed by an embedded base: `![[CategoryName.base]]`

Some pages include additional context or multiple base views:

```markdown
---
tags:
  - categories
related: "[[Podcast episodes]]"
---

![[Podcasts.base]]
```

New categories can be added at any time by creating a Category page + a corresponding base file. Zero friction.

---

## Composable Templates

### The composition principle

> Templates compose through `categories` (the noun) and `type` (the role). An Author is not a different thing from a Person — it's a Person with the Authors role layered on.

### How composition works

1. **Base templates** define the "noun" — they set `categories` and define the core properties for that kind of note (e.g., a People template sets `categories: [[People]]` and provides `birthday`, `org`)
2. **Role templates** add a `type` and embed contextual bases — an Author template adds `type: [[Authors]]` and embeds `![[Books.base#Author]]` to show their books
3. **You apply multiple templates** to the same note to compose its identity

### Template design rules

- **Every replicable pattern gets a template** — if you've created a similar note structure twice, make a template. Templates are cheap; inconsistency is expensive.
- **Compose, don't duplicate** — never create a separate category for a role. Authors are People with a type, not a different category.
- **Templates should be composable** — Person and Author are two different templates that can be layered onto the same note.
- **Include embedded bases in templates** — a Podcast template should embed `![[Podcast episodes.base#Show]]` so episodes auto-appear. A Technology template should embed `![[Projects.base#Technology]]` so projects using it auto-appear.
- **Properties in templates should use `[[wikilinks]]`** for any reference fields (author, org, genre, topics, etc.)
- **Use `{{date}}` for the `created` property** — Obsidian auto-fills this on note creation.

---

## Bases (Query Engines)

Bases live in `Templates/Bases/`. They are **embedded directly into notes** using `![[BaseName.base]]` or `![[BaseName.base#ViewName]]`.

### Key base patterns

#### Contextual views with `this`

Bases can have views that filter by `this` — the note embedding the base. This is how a Person page shows only their books:

```yaml
views:
  - type: table
    name: Author
    filters:
      and:
        - list(author).contains(this)
    order: [file.name, year, genre]
```

When embedded in a person's note via `![[Books.base#Author]]`, it shows only books where `author` contains that person.

#### Category filtering

All category bases filter by the category wikilink:

```yaml
filters:
  and:
    - categories.contains(link("Books"))
    - '!file.name.contains("Template")'
```

#### Where bases are embedded

Bases are not just for category dashboards. They belong **inside individual notes** for contextual, relationship-driven views:

| Note type            | Embedded base                      | What it shows                  |
| -------------------- | ---------------------------------- | ------------------------------ |
| Category page        | `![[Books.base]]`                  | All books                      |
| Person note (Author) | `![[Books.base#Author]]`           | Books by this person           |
| Podcast note         | `![[Podcast episodes.base#Show]]`  | Episodes for this podcast      |
| Person note (Guest)  | `![[Podcast episodes.base#Guest]]` | Episodes featuring this person |
| Technology note      | `![[Projects.base#Technology]]`    | Projects using this tech       |
| Any note             | `![[Related.base]]`                | Notes with overlapping links   |
| Any note             | `![[Backlinks.base]]`              | Notes linking to this note     |

### Base design rules

- **Every category needs a base** — stored in `Templates/Bases/`
- **Include a `this`-filtered view** for contextual embedding (e.g., Author view, Show view, Guest view)
- **Always exclude templates** — add `'!file.name.contains("Template")'` to filters
- **The Related and Backlinks bases** are universal — they can be embedded in any note for discovery

---

## Personal Style Rules

1. **Avoid folders for organization** — only infrastructure folders
2. **Avoid non-standard Markdown** — stick to Obsidian Flavored Markdown
3. **Always pluralize** categories and tags — `books` not `book`, `technologies` not `technology`
4. **Use `[[wikilinks]]` in properties** — categories, author, show, guests, org, genre, topics
5. **Use `YYYY-MM-DD` dates everywhere** — in properties, file names, body text
6. **Link aggressively in body text** — link first mentions of people, places, quotes, concepts, projects — even when no note exists yet. Unresolved links are knowledge gaps and future connections.
7. **Every replicable pattern gets a template** — if you've written it twice, make a template
8. **Compose templates with `type`** — Author = People + Authors role, never a separate category
9. **Embed bases in notes** — not just in category dashboards, but in individual notes for contextual views
10. **One vault for everything** — avoid splitting content into multiple vaults
11. **Navigate via Quick Switcher, backlinks, and links** — not the file explorer
12. **Unresolved links are features, not bugs** — they reveal what you've mentioned but haven't explored yet

---

## Migration Checklist (from folder-based vault)

When restructuring from a folder-based vault to this system:

1. Create infrastructure folders: `References/`, `Clippings/`, `Attachments/`, `Templates/`, `Templates/Bases/`, `Categories/`, `Daily/`
2. Create all composable templates in `Templates/`
3. Create all `.base` files in `Templates/Bases/`
4. Create Category pages in `Categories/` (each embeds its base)
5. Move reference notes (books, people, podcasts, technologies, others' projects) to `References/`
6. Move personal notes (projects, ideas, truths, courses) to root `/`
7. Update all frontmatter: add `categories: - "[[X]]"` using wikilinks, add `type` for composed roles, normalize properties
8. Update wikilinks: remove old folder prefixes (Obsidian resolves by name)
9. Add embedded bases to notes: Person notes get role-specific base views, Podcast notes get episode base views, etc.
10. Delete old empty folders

---

## Quick Reference: Creating a New Note

1. **Decide: root or References?** — Is it your world or the external world?
2. **Pick a template** — or compose from multiple templates
3. **Fill in properties** — use `[[wikilinks]]` for anything that references another entity, even if the note doesn't exist yet
4. **Write content** — link aggressively: people, places, quotes, concepts. Don't worry if the target note doesn't exist yet.
5. **Embed bases if needed** — contextual views like `![[Books.base#Author]]` or universal ones like `![[Related.base]]`

That's it. No folder decisions. No category anxiety. Just write and link everything.
