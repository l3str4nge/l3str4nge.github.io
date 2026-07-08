# l3str4nge — notes

A minimal Jekyll site of personal notes, hosted on GitHub Pages.
The contents list on the home page is generated automatically from the
notes in `_notes/` — there's no index to hand-edit.

## Add a new note

1. Create a markdown file in `_notes/`, e.g. `_notes/lru-eviction.md`.
2. Give it front matter:

   ```yaml
   ---
   title: LRU eviction
   category: Caching
   summary: One-line description (optional, shown at the top of the note).
   ---
   ```

3. Write the note in markdown below the front matter.

That's it. The note appears under its `category` heading on the home page
(categories and notes are sorted alphabetically). Use an existing
`category` value to add to a group, or a new one to start a group.

## Preview locally (optional)

GitHub Pages builds and publishes automatically on push, so this is only
needed if you want to preview before pushing:

```sh
bundle install
bundle exec jekyll serve
# open http://localhost:4000
```

## Structure

```
_config.yml          site config + the notes collection
index.html           home page — auto-generates the contents list
_layouts/default.html page shell (Sakura CSS, footer)
_layouts/note.html    note shell (title, summary, back links)
_notes/*.md          one markdown file per note
```