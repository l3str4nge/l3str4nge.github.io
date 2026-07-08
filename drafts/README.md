# drafts

Scratch space for dirty notes — raw thoughts, half-baked explanations,
links to read later. **Nothing in this folder is published.** It's excluded
from the Jekyll build (see `exclude:` in `_config.yml`), so you can keep
messy files here freely.

## Turning a draft into a real post

When a draft is ready:

1. Move it into `_notes/`, e.g. `_notes/eviction-policies.md`.
2. Add front matter at the top:

   ```yaml
   ---
   title: Eviction policies
   parent: caching-strategies   # for a sub-note, OR
   category: Caching            # for a top-level note
   date: 2026-06-16             # makes it show up under "Newest"
   summary: One-line description.
   ---
   ```
3. Clean up the body. Done — it appears on the site automatically.
