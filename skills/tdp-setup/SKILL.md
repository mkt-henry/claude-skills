---
name: tdp-setup
description: Set up TDP (CPU and GPU power limits) on a new or existing Windows PC — identify the hardware, read the current limits, apply the target the user wants through the safest control path (vendor tool, Windows power plan, RyzenAdj, ThrottleStop/XTU, nvidia-smi), make it survive reboot and sleep, verify under load, and leave a one-click restore. Use when the user says /tdp-setup, "TDP 세팅", "TDP 설정", "전력 제한", "파워 리밋", "발열 줄여줘", "팬 소리 줄여줘", "배터리 오래 가게", "성능 최대로", "power limit", or otherwise asks to tune how much power the CPU/GPU may draw.
---

# tdp-setup — CPU/GPU power limits on a Windows PC

You are setting the power limits of the machine you are running on. The user is not a developer: they will not read code, cannot judge register values, and want the machine left in a clear, working state. Do the work yourself; only hand them what physically needs them (a UAC "예" click, a BIOS menu, a reboot they must approve).

Work through every step in order. Keep a running log of what you read and changed — step 8 needs it.

## 0. Goal

If the user already said what they want, map it to one of these and go. Otherwise ask exactly one question, in plain Korean, and offer these three:

| Mode | Say it as | Target |
| --- | --- | --- |
| `quiet` | 조용하고 시원하게 · 배터리 오래 | sustained limit ≈ 60–70% of rated TDP, short boost ≈ rated TDP |
| `balanced` | 균형 | sustained = rated TDP, boost = manufacturer default |
| `performance` | 성능 최대 | sustained and boost = highest the manufacturer allows on this model (never above it) |

If the user gave an exact watt number, use it — after checking it is inside the allowed range found in step 2. If it is outside, say what the allowed range is and use the nearest allowed value.

## 1. Identify the machine

Run in PowerShell and keep the output:

```powershell
Get-CimInstance Win32_ComputerSystem | Select Manufacturer, Model, SystemType
Get-CimInstance Win32_SystemEnclosure | Select ChassisTypes
Get-CimInstance Win32_Processor | Select Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed
Get-CimInstance Win32_VideoController | Select Name, DriverVersion
Get-CimInstance Win32_Battery | Select Name, EstimatedChargeRemaining
Get-CimInstance Win32_BIOS | Select Manufacturer, SMBIOSBIOSVersion
(Get-CimInstance Win32_OperatingSystem).Caption
```

Classify it: **desktop**, **laptop**, or **handheld** (ROG Ally, Legion Go, MSI Claw, Steam Deck running Windows, GPD/AYANEO/OneXPlayer…). ChassisTypes 3–7 desktop, 8–14/31/32 laptop; handhelds usually report as laptop — decide by model name.

Then look up, from the CPU maker's spec page (Intel ARK `ark.intel.com`, AMD `amd.com/en/products`), the rated TDP / base power, max turbo power, and the configurable TDP range (cTDP) for that exact CPU. For a discrete NVIDIA GPU, `nvidia-smi -q -d POWER` gives its default, min and max power limits directly.

## 2. Read the current limits

Read before you change anything, and save these readings — they are the restore point.

- **Windows power plan** (all machines):
  ```powershell
  powercfg /getactivescheme
  powercfg /query SCHEME_CURRENT SUB_PROCESSOR PROCTHROTTLEMAX
  powercfg /query SCHEME_CURRENT SUB_PROCESSOR PERFBOOSTMODE
  ```
  If a setting is hidden, unhide it with `powercfg -attributes SUB_PROCESSOR <GUID> -ATTRIB_HIDE` (PERFBOOSTMODE = `be337238-0d82-4146-a960-4f3749d470c7`, PROCTHROTTLEMAX = `bc5038f7-23e0-4960-96da-33abaf5935ec`).
- **Windows power mode slider** (Settings → System → Power & battery → Power mode). Note which overlay is active.
- **Vendor control app**, if installed — Armoury Crate / Armoury Crate SE (ASUS), Lenovo Vantage / Legion Space (Lenovo), MSI Center (MSI), Dell Power Manager / Alienware Command Center (Dell), OMEN Gaming Hub (HP). Check `Get-AppxPackage` and `Get-Package` and the Start menu. Note its current mode.
- **AMD CPU**: download RyzenAdj from its official release page `github.com/FlyGoat/RyzenAdj/releases` into its own new folder, then `ryzenadj.exe --info` (run elevated). Record STAPM, PPT FAST, PPT SLOW limits and current values.
- **Intel CPU**: the limits are PL1 (sustained) and PL2 (short boost). Read them with ThrottleStop (official page `techpowerup.com/download/techpowerup-throttlestop/`) → TPL window, or with Intel XTU if the machine supports it. If neither tool can read or set them, the firmware has locked them — note that; step 4 then uses only the vendor app, the power plan and the BIOS.
- **NVIDIA GPU**: `nvidia-smi -q -d POWER` → current, default, min and max power limit.

## 3. Pick the control path

Use the first path that can reach the target on this machine. Prefer paths that survive updates and do not need extra software running.

1. **BIOS / firmware option** — some desktops and laptops expose power limits or "cTDP" in the BIOS. You cannot change the BIOS yourself. Use this path only if steps 4.2–4.5 cannot reach the target; then tell the user the exact menu path for their board (look it up for the exact model) and the value to enter, and stop there for that part.
2. **Vendor app mode** — on laptops and handhelds the manufacturer modes (e.g. Silent / Performance / Turbo, or a manual watt slider on handhelds) are the supported way and keep fan curves consistent. If a mode matches the target, use it.
3. **Windows power plan** — boost mode and max processor state. Cheap, safe, persistent; the main tool for `quiet` on Intel laptops whose PL1/PL2 are locked.
4. **RyzenAdj** (AMD) — sets STAPM / fast / slow limits directly.
5. **ThrottleStop or Intel XTU** (Intel) — sets PL1 / PL2 when not locked.
6. **nvidia-smi** (NVIDIA discrete GPU) — `nvidia-smi -pl <watts>`, within the min–max it reported.

Desktops with a discrete GPU usually want path 6 for the GPU plus path 3 or 1 for the CPU. Handhelds usually want path 2, or path 4 if the vendor slider does not go where the user wants.

## 4. Apply

Rules for every change:

- Never set a limit above the manufacturer's maximum for this model, and never below the cTDP minimum.
- No undervolting, no clock or voltage offsets, no fan-curve changes unless the user asked for them.
- Admin rights: start an elevated PowerShell with `Start-Process powershell -Verb RunAs …`. Before it appears, tell the user one line: "화면에 '이 앱이 디바이스를 변경하도록 허용하시겠어요?'가 뜨면 '예'를 눌러줘."
- Download tools only from the official pages listed above, each into its own new folder under `%LOCALAPPDATA%\tdp-setup\`.

Then, per path:

- **Vendor app**: switch it to the chosen mode (use its CLI or settings file if it has one; otherwise drive the UI or tell the user the exact tab and button).
- **Power plan**:
  ```powershell
  # quiet: turn turbo boost off (max state 99% also keeps the CPU out of turbo)
  powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PERFBOOSTMODE 0
  powercfg /setdcvalueindex SCHEME_CURRENT SUB_PROCESSOR PERFBOOSTMODE 0
  powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCTHROTTLEMAX 99
  powercfg /setdcvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCTHROTTLEMAX 99
  powercfg /setactive SCHEME_CURRENT
  ```
  For `balanced`/`performance` restore boost (`PERFBOOSTMODE 2` = aggressive) and `PROCTHROTTLEMAX 100`.
- **RyzenAdj** (values in milliwatts):
  ```powershell
  ryzenadj.exe --stapm-limit=<sustained_mW> --slow-limit=<sustained_mW> --fast-limit=<boost_mW>
  ```
- **ThrottleStop**: set PL1/PL2 in the TPL window, tick "Start minimized" and "Minimize on close", save; or with XTU set "Turbo Boost Power Max" (PL1) and "Turbo Boost Short Power Max" (PL2).
- **nvidia-smi**: `nvidia-smi -pl <watts>` (elevated).

## 5. Make it stick

Limits set by RyzenAdj and nvidia-smi reset on reboot and often on sleep or AC/battery changes. Create one scheduled task that re-applies them:

- A script `%LOCALAPPDATA%\tdp-setup\apply.ps1` holding exactly the commands from step 4.
- A scheduled task `TDP Setup - apply` running that script with highest privileges, triggered **at log on**, **on resume from sleep** (event log System, source `Microsoft-Windows-Power-Troubleshooter`, event ID 1), and **every 5 minutes** (some firmware re-imposes its own limits).

For ThrottleStop, put it in a scheduled task at log on with highest privileges instead of the Startup folder (the Startup folder triggers a UAC prompt every boot). Power plan and vendor app settings persist on their own.

## 6. Verify

1. Re-read every value from step 2 and confirm each one now shows the target.
2. Put the CPU under load for 60 seconds, for example:
   ```powershell
   $jobs = 1..([Environment]::ProcessorCount) | % { Start-Job { $e = (Get-Date).AddSeconds(60); while ((Get-Date) -lt $e) { [math]::Sqrt(12345) | Out-Null } } }
   ```
   While it runs, read package power and temperature (`ryzenadj --info`, ThrottleStop's package power, `nvidia-smi --query-gpu=power.draw,temperature.gpu --format=csv`, or HWiNFO if already installed). Stop the jobs afterwards (`$jobs | Stop-Job; $jobs | Remove-Job`).
3. The sustained power should settle at or below the target. If it overshoots, the firmware is overriding the setting — re-apply once, then say so and fall back to the vendor app or the power plan.
4. If the machine can sleep, sleep and wake it once, then re-read the limits to confirm step 5 works. Skip if the user is using the machine for something that must not be interrupted.

## 7. Restore point

Write `%LOCALAPPDATA%\tdp-setup\restore.ps1` that puts back every value read in step 2 and deletes the `TDP Setup - apply` task. Run it once in your head against the log — every change from step 4 must have a matching line.

## 8. Report

Tell the user, in plain Korean and briefly:

- What machine this is (model, CPU, GPU) in one line.
- A small before → after table: sustained power, boost power, GPU power (only rows that changed), in watts.
- What was measured under load (watts, temperature).
- That it survives reboot and sleep, and that to go back they can just say "TDP 원래대로 돌려줘" (you will run `restore.ps1`).
- Anything only they can do (a BIOS change), with the exact menu path.

Do not list commands, file paths, or register names unless they ask.
