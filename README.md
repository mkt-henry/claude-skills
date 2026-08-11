# henry-skills

A small Claude Code plugin marketplace with two skills that work in any repo.

| Skill | What it does |
| --- | --- |
| `sync` | Fetches, then does the smallest thing that puts the current branch and its upstream on the same newest commit — fast-forward, rebase, merge, commit, or push, depending on the divergence. Never force-pushes, never discards work without asking. |
| `tunnel` | Starts the project's dev server on a free random port and exposes it through a Cloudflare quick tunnel, so a phone or another machine can open it. Reports the tunnel URL and a LAN fallback. |

## Install

In Claude Code:

```
/plugin marketplace add mkt-henry/claude-skills
/plugin install henry-skills@henry-skills
```

If the install summary says `Run /reload-plugins to activate.`, run that.

## Use

```
/henry-skills:sync
/henry-skills:tunnel
```

Both also trigger from plain requests — "원격이랑 맞춰줘", "터널 열어줘", "sync this repo".

## Update

```
/plugin marketplace update henry-skills
```

## Requirements

- `sync` — git, and a remote configured for the repo.
- `tunnel` — [`cloudflared`](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) on `PATH`, and a `dev` script in `package.json`.

Quick tunnels put a dev server on the public internet, reachable by anyone with the URL. The skill warns about this before it opens one; shut it down when you're done.

## License

MIT
