Install the Helbling-configured Claude Code on this machine by following
https://api-chat.helbling.ch/install/claude-code/README.html.

Environment: Windows 11, PowerShell 7+, user-level install (no admin).
Do not ask me to confirm each step — proceed end-to-end and only stop on
a real error. Report a short status after each phase.

Steps:

1. Pre-flight
   - Confirm PowerShell version is >= 5.1 (`$PSVersionTable.PSVersion`).
   - Confirm reachability of the installer host:
       Invoke-WebRequest -UseBasicParsing -Method Head `
         https://api-chat.helbling.ch/install/claude-code/install.ps1
     If this fails, stop and report (likely off-VPN / off-corporate LAN).
   - If `%USERPROFILE%\.claude\settings.json` already exists, note its
     path and size so I know a timestamped backup will be created by the
     installer.

2. Install
   Run the official one-liner exactly as published:
       iex (irm https://api-chat.helbling.ch/install/claude-code/install.ps1)
   Capture and surface the full transcript. Do NOT modify the script,
   do NOT pipe it through any filter, do NOT re-implement its steps by
   hand — invoke it as-is so it can:
     - install/upgrade Claude Code >= 2.1.111
     - back up the existing ~/.claude/settings.json with a timestamp
     - write the Helbling settings (LLM router at /llm/, Sonnet+Opus
       picker, disable 1M-context + experimental, deny WebSearch,
       strip co-authored-by trailers, API_TIMEOUT_MS=30min)
     - run its own /v1/messages probe + `claude --print` round-trip

3. PATH check
   In a fresh shell context (`powershell -NoProfile -Command "claude --version"`),
   verify `claude` resolves. If not, tell me exactly which directory the
   installer placed the binary in and the one-line instruction to add it
   to my User PATH via Settings → Environment Variables (do not edit
   PATH for me).

4. Verify
   Re-run the official verifier:
       & ([scriptblock]::Create((irm https://api-chat.helbling.ch/install/claude-code/verify.ps1))) `
         -BaseUrl 'https://api-chat.helbling.ch/llm'
   Report PASS/FAIL plus any diagnostics it prints.

5. Summary
   Print:
     - claude --version output
     - path to the new settings.json and the backup filename
     - verifier result
     - any follow-up I need to do manually (PATH, new shell, etc.)

Important caveats to respect:
- This may upgrade the Claude Code binary that is running you right now;
  warn me before step 2 that I may need to restart this session afterward.
- Never pass --no-verify or otherwise bypass installer checks.
- If any step fails, stop and show the raw error — do not retry blindly
  and do not "fix" the installer script.