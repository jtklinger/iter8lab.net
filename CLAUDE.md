# iter8lab.net

Hugo static site (the lab's build-in-public blog), deployed to GitHub Pages by
`.github/workflows/deploy.yml` on every push to main (Hugo 0.159.1 extended).

- Local preview: `hugo server -D` → http://localhost:1313
- Content: `content/posts/`; theme: PaperMod as a **git submodule** at
  `themes/PaperMod` — clone with `--recurse-submodules`, or run
  `git submodule update --init` in a fresh checkout/worktree
- `public/` and `resources/` are build output (gitignored) — never edit them
- Edited from MULTIPLE machines: always `git fetch && git status` before working;
  never assume this checkout is current (it was found 13 behind on 2026-07-26)

## Delivery (all repos standard)

Never commit to main. Branch (`feat/…`, `fix/…`, `docs/…`, `chore/…`) → `git push -u` →
PR → squash-merge only on Jeremy's go-ahead. **Merging to main = publishing the site.**
Full standard: projects-root `CLAUDE.md`.
Fresh worktree bootstrap: `git submodule update --init` (nothing else — no npm here).
