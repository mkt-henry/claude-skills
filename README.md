# henry-skills

A small Claude Code plugin marketplace with everyday skills.

| Skill | What it does |
| --- | --- |
| `sync` | Fetches, then does the smallest thing that puts the current branch and its upstream on the same newest commit — fast-forward, rebase, merge, commit, or push, depending on the divergence. Never force-pushes, never discards work without asking. |
| `tunnel` | Starts the project's dev server on a free random port and exposes it through a Cloudflare quick tunnel, so a phone or another machine can open it. Reports the tunnel URL and a LAN fallback. |
| `tdp-setup` | Sets CPU/GPU power limits (TDP) on a Windows PC: identifies the hardware, reads the current limits, applies a quiet / balanced / performance target through the safest path (vendor app, power plan, RyzenAdj, ThrottleStop/XTU, nvidia-smi), keeps it across reboot and sleep, verifies under load and leaves a restore script. |

## Install

In Claude Code:

```
/plugin marketplace add mkt-henry/claude-skills
/plugin install henry-skills@henry-skills
```

If the install summary says `Run /reload-plugins to activate.`, run that.

On a fresh PC you can also just tell Claude Code: "mkt-henry/claude-skills 마켓플레이스 추가하고 henry-skills 플러그인 설치해줘" — it runs `claude plugin marketplace add mkt-henry/claude-skills` and `claude plugin install henry-skills@henry-skills` for you.

## Use

```
/henry-skills:sync
/henry-skills:tunnel
/henry-skills:tdp-setup
```

They also trigger from plain requests — "원격이랑 맞춰줘", "터널 열어줘", "TDP 세팅해줘", "sync this repo".

## Update

```
/plugin marketplace update henry-skills
```

## Requirements

- `sync` — git, and a remote configured for the repo.
- `tdp-setup` — Windows 10/11 and an account that can approve admin (UAC) prompts. Downloads RyzenAdj or ThrottleStop from their official pages only when the machine needs them.
- `tunnel` — [`cloudflared`](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) on `PATH`, and a `dev` script in `package.json`.

Quick tunnels put a dev server on the public internet, reachable by anyone with the URL. The skill warns about this before it opens one; shut it down when you're done.

## License

MIT
