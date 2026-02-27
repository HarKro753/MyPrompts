---
name: obsidian-cli
description: Interact with Obsidian vaults using the Obsidian CLI to read, create, search, and manage notes, tasks, properties, and more. Also supports plugin and theme development with commands to reload plugins, run JavaScript, capture errors, take screenshots, and inspect the DOM. Use when the user asks to interact with their Obsidian vault, manage notes, search vault content, perform vault operations from the command line, or develop and debug Obsidian plugins and themes.
---

# Obsidian CLI

Use the `obsidian` CLI to interact with a running Obsidian instance. Requires Obsidian to be open.

## Requirements

- **Obsidian 1.12+** (early access, requires [Catalyst license](https://obsidian.md/pricing))
- **Installer version 1.11.7+**
- Enable in Obsidian: **Settings → General → Command line interface**
- macOS PATH (added automatically, or manually add to `~/.zprofile`):
  ```
  export PATH="$PATH:/Applications/Obsidian.app/Contents/MacOS"
  ```

Full docs: https://help.obsidian.md/cli

## Related skills

- **obsidian-markdown** — Obsidian Flavored Markdown syntax (wikilinks, callouts, properties/frontmatter, embeds, tags). Use when writing or editing note content.
- **obsidian-bases** — Obsidian Bases `.base` file format (views, filters, formulas, summaries). Use when creating or editing database-like views of notes.

## Syntax

**Parameters** take a value with `=`. Quote values with spaces:

```bash
obsidian create name="My Note" content="Hello world"
```

**Flags** are boolean switches with no value:

```bash
obsidian create name="My Note" silent overwrite
```

For multiline content use `\n` for newline and `\t` for tab.

## File targeting

Many commands accept `file` or `path` to target a file. Without either, the active file is used.

- `file=<name>` — resolves like a wikilink (name only, no path or extension needed)
- `path=<path>` — exact path from vault root, e.g. `folder/note.md`

## Vault targeting

Commands target the most recently focused vault by default. Use `vault=<name>` as the first parameter to target a specific vault:

```bash
obsidian vault="My Vault" search query="test"
```

## Global flags

- `--copy` — copy output to clipboard
- `silent` — prevent files from opening in the UI
- `total` — return count instead of list (on list commands)

---

## CRUD — Files and folders

### Create

```bash
obsidian create                                          # create "Untitled"
obsidian create name="Note" content="# Hello"            # with content
obsidian create name="Note" template=Travel              # from template
obsidian create path="Projects/Note.md" content="body"   # at specific path
obsidian create name="Note" open                         # open after creating
obsidian create name="Note" overwrite                    # overwrite if exists
obsidian create name="Note" open newtab                  # open in new tab
obsidian create name="Note" silent                       # don't open in UI
```

| Parameter | Description |
|-----------|-------------|
| `name=<name>` | file name |
| `path=<path>` | file path from vault root |
| `content=<text>` | initial content |
| `template=<name>` | template to use |
| `overwrite` | overwrite if exists |
| `open` | open after creating |
| `newtab` | open in new tab |
| `silent` | don't open in UI |

### Read

```bash
obsidian read                        # read active file
obsidian read file="Recipe"          # by name
obsidian read path="Notes/Recipe.md" # by path
```

### Update — append / prepend

```bash
obsidian append file="Note" content="New line"
obsidian append file="Note" content="inline text" inline   # no newline
obsidian prepend file="Note" content="Top line"
```

| Parameter | Description |
|-----------|-------------|
| `file=<name>` | target file |
| `path=<path>` | target path |
| `content=<text>` | **(required)** content to add |
| `inline` | no newline before content |

### Delete

```bash
obsidian delete file="Old Note"      # move to trash
obsidian delete file="Old Note" permanent   # permanent delete
```

### Move / Rename

```bash
obsidian move file="Note" to="Archive/Note.md"
obsidian rename file="Note" name="Better Name"
```

### File info

```bash
obsidian file                        # active file info
obsidian file file="Recipe"          # specific file info
obsidian files                       # list all files
obsidian files folder="Projects"     # list files in folder
obsidian files ext=md                # filter by extension
obsidian files total                 # count files
obsidian folders                     # list all folders
```

### Open

```bash
obsidian open file="Note"
obsidian open file="Note" newtab
```

---

## Search

```bash
obsidian search query="meeting notes"                # file paths matching
obsidian search query="TODO" path="Projects" limit=5 # scoped + limited
obsidian search query="bug" total                    # count matches
obsidian search query="bug" case                     # case sensitive
obsidian search:context query="TODO"                 # with line context (grep-style)
obsidian search:context query="TODO" format=json     # JSON output
```

---

## Daily notes

```bash
obsidian daily                                    # open daily note
obsidian daily:path                               # get daily note path
obsidian daily:read                               # read daily note
obsidian daily:append content="- [ ] Buy milk"    # append to daily
obsidian daily:prepend content="# Morning"        # prepend to daily
```

---

## Properties (frontmatter)

```bash
obsidian properties                              # list all vault properties
obsidian properties active                       # properties of active file
obsidian properties file="Note"                  # properties of specific file
obsidian property:read name="status" file="Note" # read one property
obsidian property:set name="status" value="done" file="Note"
obsidian property:set name="tags" value="project" type=list file="Note"
obsidian property:remove name="status" file="Note"
```

| Type values | `text`, `list`, `number`, `checkbox`, `date`, `datetime` |
|-------------|----------------------------------------------------------|

---

## Tags

```bash
obsidian tags                        # list all tags
obsidian tags counts                 # with occurrence counts
obsidian tags sort=count             # sorted by count
obsidian tags active                 # tags for active file
obsidian tag name="project"          # tag info
obsidian tag name="project" verbose  # with file list
```

---

## Tasks

```bash
obsidian tasks                       # all tasks
obsidian tasks todo                  # incomplete only
obsidian tasks done                  # completed only
obsidian tasks daily                 # from daily note
obsidian tasks daily todo            # incomplete from daily note
obsidian tasks file="Recipe"         # from specific file
obsidian tasks verbose               # grouped by file with line numbers
obsidian tasks total                 # count
obsidian task ref="Recipe.md:8" toggle        # toggle completion
obsidian task daily line=3 done               # mark daily task done
obsidian task file="Note" line=5 status=-     # set custom status
```

---

## Links

```bash
obsidian backlinks file="Note"       # files linking to Note
obsidian backlinks counts            # with link counts
obsidian links file="Note"           # outgoing links from Note
obsidian unresolved                  # broken links in vault
obsidian orphans                     # files with no incoming links
```

---

## Templates

```bash
obsidian templates                   # list templates
obsidian template:read name="Travel" # read template content
obsidian template:read name="Travel" resolve  # with variables resolved
obsidian template:insert name="Travel"        # insert into active file
```

---

## Outline

```bash
obsidian outline                     # headings for active file
obsidian outline file="Note"         # headings for specific file
obsidian outline format=json         # as JSON
```

---

## Bases

```bash
obsidian bases                       # list all .base files
obsidian base:views                  # list views in active base
obsidian base:create file="Tasks" name="New Item" content="body"
obsidian base:query file="Tasks"                    # query base (JSON)
obsidian base:query file="Tasks" view="Active" format=csv
```

See the **obsidian-bases** skill for `.base` file syntax.

---

## Plugins

```bash
obsidian plugins                         # list installed
obsidian plugins:enabled                 # list enabled
obsidian plugin id=my-plugin             # plugin info
obsidian plugin:enable id=my-plugin
obsidian plugin:disable id=my-plugin
obsidian plugin:install id=my-plugin enable
obsidian plugin:uninstall id=my-plugin
obsidian plugin:reload id=my-plugin      # reload for development
```

---

## Themes and snippets

```bash
obsidian themes                      # list installed themes
obsidian theme                       # active theme
obsidian theme:set name="Minimal"
obsidian theme:install name="Minimal" enable
obsidian snippets                    # list CSS snippets
obsidian snippet:enable name="custom"
obsidian snippet:disable name="custom"
```

---

## Workspace

```bash
obsidian workspace                   # show workspace tree
obsidian tabs                        # list open tabs
obsidian recents                     # recently opened files
obsidian workspaces                  # list saved workspaces
obsidian workspace:save name="Dev"
obsidian workspace:load name="Dev"
```

---

## Vault

```bash
obsidian vault                       # vault info (name, path, size)
obsidian vaults                      # list all known vaults
obsidian vaults verbose              # with paths
```

---

## Developer commands

```bash
obsidian eval code="app.vault.getFiles().length"   # run JS
obsidian dev:errors                                 # captured JS errors
obsidian dev:console                                # console messages
obsidian dev:console level=error limit=20           # filtered
obsidian dev:screenshot path=screenshot.png         # screenshot
obsidian dev:dom selector=".workspace-leaf" text    # query DOM
obsidian dev:dom selector=".nav-file" all           # all matches
obsidian dev:css selector=".workspace-leaf" prop=background-color
obsidian dev:mobile on                              # toggle mobile emulation
obsidian devtools                                   # toggle dev tools
obsidian dev:debug on                               # attach CDP debugger
obsidian dev:cdp method="Page.reload"               # raw CDP command
```

---

## Sync and Publish

```bash
# Sync
obsidian sync on                     # resume sync
obsidian sync off                    # pause sync
obsidian sync:status                 # sync status + usage
obsidian sync:history file="Note"    # version history

# Publish
obsidian publish:list                # published files
obsidian publish:status              # pending changes
obsidian publish:add file="Note"     # publish a file
obsidian publish:add changed         # publish all changes
```

---

## File history / diff

```bash
obsidian diff file="Note"            # list all versions
obsidian diff file="Note" from=1     # compare latest version to current
obsidian diff file="Note" from=2 to=1  # compare two versions
obsidian history file="Note"         # local history versions
obsidian history:restore file="Note" version=2
```

---

## Other useful commands

```bash
obsidian help                        # all commands
obsidian help <command>              # help for specific command
obsidian version                     # Obsidian version
obsidian reload                      # reload app window
obsidian restart                     # restart app
obsidian random                      # open random note
obsidian random:read                 # read random note
obsidian bookmarks                   # list bookmarks
obsidian wordcount                   # word + character count
obsidian web url="https://example.com"  # open in web viewer
obsidian unique name="Idea"          # create unique note
obsidian commands                    # list all command IDs
obsidian command id="app:reload"     # execute a command
```
