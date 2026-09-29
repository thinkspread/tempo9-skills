# Contributing to tempo9-skills

Copyright (c) 2026 Jiejing Zhang.

These skills record decisions a shipping app had to get right. A change is
easiest to take when it keeps that bar:

- A number comes with where it was measured: machine, date, model file, and
  how many runs.
- A rule comes with the failure it prevents.
- One skill per directory: a `SKILL.md` whose front matter has `name` and
  `description`.

## Sign your commits (DCO)

Every commit needs a `Signed-off-by:` line with your name and the email the
commit is authored with; `git commit -s` adds it. Signing off certifies the
[Developer Certificate of Origin 1.1](DCO). A check on every pull request
refuses commits without it.

There is no separate contributor licence agreement: contributions are taken
under this repository's licence (Apache-2.0), and you keep the copyright in
your work.

## How changes flow

Outside contributions are reviewed and merged here. The maintainers also keep
a private copy; what is merged here is carried into it with your authorship
and sign-off intact.

## Engine and SDK issues

Bugs in the engine, the Swift SDK or the `tempo9` CLI belong in
[thinkspread/tempo9](https://github.com/thinkspread/tempo9/issues), not here.
