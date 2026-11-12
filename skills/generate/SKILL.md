---
name: generate
description: Generate a reproducible animated SVG demo of a terminal session (asciinema + svg-term-cli via bunx or npx) for a README. Use when the user wants a demo gif/svg of CLI usage, shell aliases or git workflows, "gravar demo do terminal", or to regenerate an existing demo.
---

# Terminal demo → animated SVG

Scripted (not live) recording: a sandbox repo, a player that fakes typing, `asciinema rec`, then `svg-term-cli`. Reference implementation: `resources/demo/` in rockberpro-labs/git-lga (`record.sh`, `play.sh`, `script.txt`), output `resources/img/demo.svg`.

## Prerequisites (check first)
Run `command -v asciinema && asciinema --version` and `command -v bunx || command -v npx` before anything else. If `asciinema` is missing (or older than 2.4.0), **stop and warn the user** that it is a required dependency this skill cannot work without, give install options (`sudo apt install asciinema`, `brew install asciinema`, `pipx install asciinema`) and do not continue until it is installed. Pick the runner once: `bunx` if present, otherwise `npx -y`. If neither exists, stop and ask the user to install bun or Node.js. Use the chosen runner wherever this skill says `bunx`.

## Pipeline
1. **Sandbox outside the project** (e.g. `../<project>-demo/`), recreated on every run. If the demo pushes, add a local bare repo as `origin` (`../<project>-demo-remote.git`) — never the real remote.
2. **Isolated config**: `GIT_CONFIG_GLOBAL` → temp gitconfig that `[include]`s the project's config, sets a dummy `user.name/email`, `core.pager=cat`, `commit.gpgsign=false`. Also `GIT_CONFIG_SYSTEM=/dev/null`. Shows the working-copy aliases, not the installed ones, and keeps real identity/hooks out.
3. **`script.txt`**: one command per line, optional `# comment` shown in gray.
4. **`play.sh`**: per line, print a green `$ `, type char by char (~0.07s), `eval`, sleep ~1.2s; short pause at start, ~2s at the end so the loop doesn't cut abruptly.
5. **Record**: `asciinema rec --overwrite -q -i 1 --cols 80 --rows 18 -c "bash play.sh" demo.cast` (2.4.0 supports `--cols/--rows`; writes asciicast v2, which svg-term reads).
6. **Sanitize** the cast with `sed` (local remote path → a public-looking URL).
7. **Convert**: `bunx svg-term-cli --in demo.cast --out resources/img/demo.svg --window --no-cursor --padding 10` (with npm: `npx -y svg-term-cli …`, same flags).
8. **README**: `<p align="center"><img src="resources/img/demo.svg" alt="… demo" width="700"></p>` below the tagline.

## Gotchas
- `shopt -s expand_aliases` **before** `source`-ing the aliases file, or non-interactive bash says "command not found".
- Aliases defined by sourcing a bashrc: silence its output (`2>/dev/null`).
- Pin cols/rows; otherwise the SVG inherits the agent's pty size.
- `-i 1` caps idle gaps.
- `git status` shows an untracked dir (`src/`), not the file inside it — expected.
- Push needs a remote: use the local bare one.
- `bunx`/`npx` may write a lockfile in the cwd — check `git status` afterward.
- Heredocs/commands containing a literal `git push … main` can trip push-guard hooks even when harmless; write such files with the Write tool and keep setup from pushing to `main`.
- Keep the demo fully in the README's language (commands, commit messages) and force git's output language with `LC_ALL=C.UTF-8 LANGUAGE=en_US:en`. Plain `LC_ALL=C` makes asciinema refuse to run (needs UTF-8).
- Keep temp artifacts (cast, config) in a `mktemp -d` removed by `trap`.

## Verify
- Run the generator; every scripted command must succeed (check sandbox `git log`, bare remote `git branch`).
- `grep -c "$HOME" demo.svg` must be 0; also no `command not found`/`fatal`.
- Open the SVG in a browser: animation plays, text legible, colors present; check file size (~150 KB is fine).
- Don't commit or push the project unless the user asks.
