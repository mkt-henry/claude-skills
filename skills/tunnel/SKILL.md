---
name: tunnel
description: Start this project's dev server on a free random port and expose it through a Cloudflare quick tunnel, so a phone or another device can open it. Use when the user says /tunnel, "퀵 터널", "터널 열어줘", "폰에서 확인", or otherwise asks to reach the local dev server from outside this machine. Not related to /remote-control, which is a built-in Claude Code command.
---

# tunnel — dev server + Cloudflare quick tunnel

Gets the current project's dev server running and publicly reachable in one shot. Works in any repo — nothing here is specific to one project. Run every step; report the URLs at the end.

**Tell the user, before or with the first command:** a quick tunnel puts the dev server on the public internet, reachable by anyone who has the URL. Dev servers usually run with real credentials from a local `.env`, and often with weaker auth than production. Ask them to say the word when they're done so the tunnel can be shut down.

## Steps

1. **Find the dev command.** Look for a `package.json` with a `dev` script — first in the current directory, then one level down in an app directory (`web/`, `app/`, `frontend/`, `client/`, `site/`). If exactly one candidate exists, use it. If several do, ask which one. If none does, ask the user how to start their dev server instead of guessing.

   Pick the package manager from the lockfile next to that `package.json`: `pnpm-lock.yaml` → `pnpm`, `yarn.lock` → `yarn`, `bun.lockb` → `bun`, otherwise `npm`.

2. **Pick a free random port.** Let the OS hand out an unused one rather than assuming 3000 or 3001 is available:

   ```powershell
   $l = [System.Net.Sockets.TcpListener]::new([System.Net.IPAddress]::Loopback, 0)
   $l.Start(); $l.LocalEndpoint.Port; $l.Stop()
   ```

   On macOS or Linux, `python3 -c 'import socket;s=socket.socket();s.bind(("",0));print(s.getsockname()[1]);s.close()'` does the same.

   Never kill whatever holds a port — it may be something of the user's with nothing to do with this project.

3. **Start the dev server** in the background on that port, bound to all interfaces so the LAN address works too. Pass the port both ways, since the flag spelling varies by framework while `PORT` is widely respected:

   ```bash
   cd <app-dir> && PORT=<port> npm run dev -- -H 0.0.0.0 -p <port>
   ```

   If the tool rejects those flags (Vite wants `--host --port <port>`, and some setups take only the env var), drop the flags and rely on `PORT`. Adjust `npm run` to the package manager from step 1.

4. **Read the task output file and take the port from what the server actually printed** — that line is the truth, not the port requested in step 2. A server that fell forward to a different port is fine; carry the printed one into step 5.

   The one failure worth acting on is a dev server refusing because an earlier instance for *this same directory* outlived its parent process. Next.js says so explicitly:

   ```
   ⨯ Another next dev server is already running.
   - PID:  33264
   - Dir:  C:\dev\project\my-app\web
   ```

   Confirm the reported `Dir` is the app directory from step 1, then kill that PID (`taskkill /PID <pid> /F`, or `kill <pid>`) and start again from step 2. If `Dir` points somewhere else, stop and ask the user — it is not ours to kill.

5. **Open the tunnel** in the background, against the port from step 4:

   ```bash
   cloudflared tunnel --url http://localhost:<port> --no-autoupdate
   ```

   If `cloudflared` is not on `PATH`, check `C:\Program Files (x86)\cloudflared\cloudflared.exe`. If it is genuinely missing, say so and stop — do not install it without asking.

6. **Read the tunnel's output file** for the generated hostname. It appears a few seconds in, as a boxed line ending in `.trycloudflare.com`. If the box is not there yet, read the file again; do not invent or predict a URL.

7. **Get the LAN address** as a fallback for a phone on the same Wi-Fi:

   ```powershell
   Get-NetIPAddress -AddressFamily IPv4 |
     Where-Object { $_.IPAddress -notlike '127.*' -and $_.PrefixOrigin -ne 'WellKnown' } |
     Select-Object IPAddress, InterfaceAlias
   ```

   Skip virtual adapters — a `192.168.56.x` address is typically VirtualBox and a phone cannot reach it. Prefer the physical Wi-Fi or Ethernet interface. On macOS or Linux use `ipconfig getifaddr en0` or `hostname -I`.

8. **Report** the tunnel URL first, the LAN URL second, and remind the user the tunnel stays up until they ask to stop it.

## Notes

- Dev servers block cross-origin requests by default, which lets a tunnelled page load while HMR silently dies. Next.js needs `allowedDevOrigins: ["*.trycloudflare.com", "192.168.*.*", "172.*.*.*", "10.*.*.*"]` in `next.config.ts`; Vite needs the tunnel host in `server.allowedHosts`. If the page loads but never hot-reloads, check that first — and add the entry to the project's config rather than working around it.
- Quick tunnels are account-less and get a fresh random hostname every run, so a URL from a previous session is dead. Never reuse one.
- To shut down: stop the `cloudflared` background task, then the dev server one. Check the port is actually released afterwards — the dev server can outlive its parent process, which is exactly how the zombie in step 4 gets created.
