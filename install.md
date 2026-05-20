Install the Helbling-configured Claude Code on this machine by following
https://api-chat.helbling.ch/install/claude-code/README.html.

Environment: Windows 11, PowerShell 7+ available as `pwsh`, user-level
install (no admin).
Do not ask me to confirm each step — proceed end-to-end and only stop on
a real error. Report a short status after each phase.

Tool-execution note (important):
- Run every PowerShell command via the `Bash` tool by shelling out to
  `pwsh -NoProfile -Command "..."` (preferred) or
  `powershell -NoProfile -Command "..."` (Windows PowerShell 5.1 fallback).
- For anything multi-line or with quoting that would be painful inline,
  write a `.ps1` file with the `Write` tool first and then invoke it via
  `pwsh -NoProfile -ExecutionPolicy Bypass -File <path>`.

Steps:

1. Pre-flight
   - Print the PowerShell version that will run the installer:
       pwsh -NoProfile -Command "$PSVersionTable.PSVersion.ToString()"
     Confirm it is >= 5.1 (anything 7.x is fine).
   - Confirm reachability of the installer host (HEAD request, 15s timeout):
       pwsh -NoProfile -Command "try { (Invoke-WebRequest -UseBasicParsing -Method Head 'https://api-chat.helbling.ch/install/claude-code/install.ps1' -TimeoutSec 15).StatusCode } catch { 'ERROR: ' + $_.Exception.Message }"
     If this fails, stop and report (likely off-VPN / off-corporate LAN).
   - Note whether `%USERPROFILE%\.claude\settings.json` already exists so
     I know a timestamped backup will be created by the installer:
       pwsh -NoProfile -Command "$p = Join-Path $env:USERPROFILE '.claude\settings.json'; if (Test-Path $p) { $i = Get-Item $p; 'EXISTS: ' + $i.FullName + ' (' + $i.Length + ' bytes, ' + $i.LastWriteTime + ')' } else { 'NOT FOUND: ' + $p }"

2. Install
   Run the official one-liner exactly as published, captured into a
   transcript so the full output is visible:
       pwsh -NoProfile -Command "Start-Transcript -Path $env:TEMP\claude-install.log -Force | Out-Null; try { iex (irm https://api-chat.helbling.ch/install/claude-code/install.ps1) } finally { Stop-Transcript | Out-Null }; Get-Content $env:TEMP\claude-install.log"
   Surface the full transcript. Do NOT modify the script, do NOT pipe it
   through any filter, do NOT re-implement its steps by hand — invoke it
   as-is so it can:
     - install/upgrade Claude Code >= 2.1.111
     - back up the existing ~/.claude/settings.json with a timestamp
     - write the Helbling settings (LLM router at /llm/, Sonnet+Opus
       picker, disable 1M-context + experimental, deny WebSearch,
       strip co-authored-by trailers, API_TIMEOUT_MS=30min)
     - run its own /v1/messages probe + `claude --print` round-trip

   PATH-abort recovery: if the installer prints a "not in your PATH"
   warning and aborts (exit code 3) before the Helbling-specific steps,
   the upstream-installed binary is there but the wrapper can't reach
   it. In that case, fix PATH automatically and re-run the installer
   (do NOT fall back to manual instructions):
     a. Parse the directory from the installer warning (typically
        `C:\Users\<user>\.local\bin`). Confirm `claude.exe` exists there.
     b. Append it to my **User** PATH persistently AND to the current
        process PATH:
            pwsh -NoProfile -Command "$d='C:\Users\sfr\.local\bin'; $u=[Environment]::GetEnvironmentVariable('Path','User'); $parts=($u -split ';') | Where-Object { $_ -ne '' }; if (-not ($parts -contains $d)) { [Environment]::SetEnvironmentVariable('Path', (($parts + $d) -join ';'), 'User'); 'Appended to User PATH' } else { 'Already in User PATH' }; $env:Path = $env:Path + ';' + $d"
        Substitute the actual directory from step (a) if it differs.
     c. Re-run the exact same installer one-liner from above. The
        installer is idempotent; it will skip the binary install and
        proceed to write Helbling settings + run its probes.

3. PATH check
   In a fresh shell context, verify `claude` resolves:
       pwsh -NoProfile -Command "claude --version"
   If it still doesn't resolve after the recovery in step 2, stop and
   report which directory the binary lives in and what the current
   User PATH is, so I can debug it manually.

4. Verify
   Re-run the official verifier:
       pwsh -NoProfile -Command "& ([scriptblock]::Create((irm https://api-chat.helbling.ch/install/claude-code/verify.ps1))) -BaseUrl 'https://api-chat.helbling.ch/llm'"
   Report PASS/FAIL plus any diagnostics it prints.

5. Summary
   Print:
     - `claude --version` output
     - path to the new settings.json and the backup filename
     - verifier result
     - any follow-up I need to do manually (PATH, new shell, etc.)

Important caveats to respect:
- This may upgrade the Claude Code binary that is running you right now;
  warn me before step 2 that I may need to restart this session afterward.
- Never pass --no-verify or otherwise bypass installer checks.
- If any step fails, stop and show the raw error — do not retry blindly
  and do not "fix" the installer script.
- If a `pwsh -NoProfile -Command "..."` invocation hits ugly quoting
  issues, write the snippet to a temporary `.ps1` file and run it with
  `pwsh -NoProfile -ExecutionPolicy Bypass -File <path>` instead of
  fighting Bash↔PowerShell escaping.
