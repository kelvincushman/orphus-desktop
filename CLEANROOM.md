# Clean-room rules

Orphus Desktop is an independent, MIT-licensed recreation of the feature set of
[PI-Desktop](https://github.com/vastsa/PI-Desktop). PI-Desktop is licensed
LGPL-3.0-or-later (adopted 2026-08-18); before that date it had no licence at all
("TBD", so all rights reserved). **None of its code may enter this repo.** These
rules apply to every commit. If a change breaks one, it does not merge.

## Never

1. **No PI-Desktop code.** Not source, types, styles, tests, scripts, configs,
   build files, patches, icons or other assets, from any version or commit.
2. **No PI-Desktop prose.** Don't copy sentences from its README, docs, i18n
   strings or comments. Write our own words.
3. **No porting by translation.** Rewriting one of their files line by line in
   another style, language or naming scheme is still copying.
4. **No implementing with their source open.** Whoever writes code here works
   from [`docs/PRD.md`](docs/PRD.md) and [`docs/DESIGN.md`](docs/DESIGN.md), not
   from the PI-Desktop repo.

## Allowed

- **The functional spec.** `docs/PRD.md` describes *what* PI-Desktop does, in our
  own words. Contributors build from that.
- **Public, user-visible facts.** Feature names, file extensions a user sees, and
  supported platforms and providers. Interoperability formats only where a
  documented public spec exists.
- **Permissively licensed dependencies** (MIT, Apache-2.0, BSD, ISC), recorded in
  [`PROVENANCE.md`](PROVENANCE.md). The pi packages (`@earendil-works/pi-*`) are
  MIT and are allowed.
- **Orphus code** from `kelvincushman/orphus` (MIT).

## Roles

- **Spec writers** may read PI-Desktop and write `docs/`. They describe
  behaviour, never implementation.
- **Implementers** read only `docs/` and our own code. A contributor who has read
  PI-Desktop source should not implement the same area from memory within the
  same session.
