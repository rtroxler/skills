# skills

Agent skills I use with Claude Code. Each lives in its own directory under [`skills/`](skills/)
as a `SKILL.md` plus any reference files it loads on demand.

## Skills

| Skill | Command | What it does |
|-------|---------|-------------|
| [rails-37signals-style](skills/rails-37signals-style/) | `/rails-37signals-style` | Write and review Rails code in the 37signals/DHH "vanilla Rails" style. 43 pattern cards extracted from the Campfire codebase, each with citations, a copy-pasteable example, a counter-example, when-to-break caveats, and a detection heuristic. Loads reference files on demand by dimension (models, controllers, Hotwire, jobs, persistence, testing, security, deployment, conventions). |
| [rails-code-review](skills/rails-code-review/) | `/rails-code-review` | Opinionated two-layer code review for Rails diffs: a standard correctness/security/performance pass, plus a Rails-idiom pass that flags deviations from the patterns above and suggests the preferred idiom with a citation. **Depends on `rails-37signals-style`** (it cites its reference cards) — install both. |

## Install

With the [`skills`](https://github.com/obra/skills) CLI:

```bash
npx skills@latest add rtroxler/skills
```

Or manually — copy (or symlink) a skill directory into `~/.claude/skills/`:

```bash
git clone https://github.com/rtroxler/skills.git
cp -R skills/skills/rails-37signals-style ~/.claude/skills/
cp -R skills/skills/rails-code-review     ~/.claude/skills/
```

Skills are picked up automatically; invoke one with its slash command (e.g. `/rails-37signals-style`)
or let it auto-trigger from its description.

## Notes

- `rails-37signals-style` is pinned to the 37signals *philosophy*, not a Rails version. Where the
  audited Campfire snapshot (Feb 2024) predates a framework feature it now ships (Rails 8 auth
  generator, Solid Queue/Cache/Cable, Thruster, Kamal 2), the reference files say so in an "era note"
  rather than handing you dated advice.
- `rails-code-review` references `rails-37signals-style` by its installed path
  (`~/.claude/skills/rails-37signals-style/`), so keep both installed and the directory names intact.

## License

[MIT](LICENSE)
