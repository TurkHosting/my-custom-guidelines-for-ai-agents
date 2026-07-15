# My Custom Guidelines for AI Agents

Custom AI guidelines for [Laravel Boost](https://laravel.com/docs/13.x/boost) projects used across [TurkHosting](https://github.com/TurkHosting).

Laravel Boost supports [custom AI guidelines](https://laravel.com/docs/13.x/boost#adding-custom-ai-guidelines) by loading `.md` or `.blade.php` files from your project's `.ai/guidelines/` directory. This repo stores our shared guidelines so they can be pulled into any project.

## What's Inside

- **`.ai/guidelines/my-custom-guidelines.md`** — Rules for communication, code style, git workflow, and build commands.
- **`.claude/commands/`** — Reusable Claude Code slash commands that enforce the guidelines:
  - `english-comments` — translate non-English internal code comments to English.
  - `clean-body-comments` — remove multi-line explanatory comment blocks from inside function/method bodies.

## Quick Setup

### Option 1: Shell Function

Add this to your `~/.zshrc`:

```bash
my-custom-guidelines() {
  if ! command -v gh &> /dev/null; then
    echo "Error: GitHub CLI (gh) is not installed."
    echo "Install it with: brew install gh"
    return 1
  fi

  if ! gh auth status &> /dev/null; then
    echo "Error: GitHub CLI is not authenticated."
    echo "Run: gh auth login"
    return 1
  fi

  mkdir -p .ai/guidelines
  gh api repos/TurkHosting/my-custom-guidelines-for-ai-agents/contents/.ai/guidelines/my-custom-guidelines.md --jq '.content' | base64 -d > .ai/guidelines/my-custom-guidelines.md

  echo "Custom guidelines added to .ai/guidelines/"
}
```

Then run `source ~/.zshrc` and use `my-custom-guidelines` in any project directory.

### Option 2: Claude Code Skill

See the [gist](https://gist.github.com/boranbar/10abba63997bc8a416b7e1620ea0875e) for instructions on setting up a `/my-custom-guidelines` slash command in Claude Code.

## Requirements

- [GitHub CLI (`gh`)](https://cli.github.com/) — `brew install gh`
- Authenticated with `gh auth login`
