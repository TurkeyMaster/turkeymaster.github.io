---
title: "Before You Run It: How I Check a Downloaded File"
date: 2026-11-14 12:00:00 -0800
categories: [Guides, Malware Analysis]
tags: [malware-analysis, windows, reverse-engineering, sandboxing, safety]
description: The steps I follow before running any downloaded EXE, DLL, script, or archive, from hash and signature checks to static and dynamic analysis in a VM.
---

I never run a downloaded program until I know what it does. That goes double for game tools, mods, and anything passed around on forums, where "it's a false positive, just disable your antivirus" is a line malware authors love.

Antivirus is a hint, not an answer. Packers and crypters routinely get malware past every engine, and legitimate game tools trip false positives all the time. A clean scan proves nothing and a dirty one doesn't prove much either. So this is the process I actually follow for any EXE, DLL, script, or archive before it touches my real machine.

> This is a guide for checking files you've downloaded for your own safety. Do all of it inside a VM you can throw away, never on your main PC.
{: .prompt-warning }

## Stop signs

If I find any one of these, I don't run the file. No exceptions.

- **Network activity it has no reason for.** A trainer or offline tool shouldn't connect anywhere. An updater calling its own vendor's domain is expected; anything else isn't.
- **Reading things it has no business reading:** browser profiles and saved passwords, cookies, Discord tokens, crypto wallets, SSH keys.
- **Persistence:** `Run`/`RunOnce` registry keys, scheduled tasks, new services, or the Startup folder.
- **Dropping or downloading another executable** and running it.
- **Injecting into processes** other than the one it says it targets.
- **Too packed to read.** If I can't unpack or deobfuscate it far enough to see what it does, it's untrusted.
- **The download doesn't match the description:** extra EXEs, scripts, or shortcuts in the archive.
- **A sketchy source:** a reupload of someone else's tool, a chain of link shorteners, "password in the comments", or a file sent in a DM.

## Set up once

Build an analysis VM once and reuse it for every file.

1. **A Windows VM** (Hyper-V, VMware, or VirtualBox). Turn off shared folders, shared clipboard, and drag-and-drop while a sample is inside.
2. **No real internet.** Use host-only networking or disconnect it. To see what a file tries to reach, fake the internet inside the lab with [FakeNet-NG](https://github.com/mandiant/flare-fakenet-ng) or INetSim.
3. **Install every tool first, then take a clean snapshot.** Revert to it after every file.
4. **Tools:** Detect It Easy, PE-bear or CFF Explorer, IDA or Ghidra, x64dbg, dnSpyEx or ILSpy, Sysinternals (Process Monitor, Process Explorer, Autoruns, Sigcheck, Strings), Wireshark, Regshot, 7-Zip.
5. **Never sign into real accounts inside the VM.** Nothing in it should be worth stealing.

## Step 1: Before you open it

1. **Check the source.** Get it from the author's own release page or repo, not a reupload or mirror. If source code is available, building it yourself beats trusting a binary.
2. **Hash it.** If the author published a hash, they must match.
   ```powershell
   Get-FileHash .\file.exe -Algorithm SHA256
   ```
3. **Look up the hash** on VirusTotal by searching the hash, not uploading the file. Uploads become available to VirusTotal's paying subscribers, which matters for anything private. Generic labels like "HackTool" or "Riskware" are common on game tools. A named malware family (a stealer, RAT, or miner) is not.
4. **Check the signature.** A valid signature from the expected publisher is good. A signature from an unrelated company, or a revoked certificate, is a red flag. Unsigned is normal for hobby tools, so I treat it as neutral.
   ```powershell
   Get-AuthenticodeSignature .\file.exe | Format-List Status, SignerCertificate
   ```
5. **List archives before extracting.** Be wary of nested archives and password-protected ones, which are often used to slip past scanners.
   ```text
   7z l archive.zip
   ```

## Step 2: Identify and unpack

From here on, everything happens inside the VM.

1. **Identify what it really is,** not what the extension says. Detect It Easy tells you whether it's a native PE (x86 or x64), .NET, a script compiled to an EXE (AutoIt, PyInstaller, Electron), or an installer (Inno Setup, NSIS).
2. **Check for packing.** Signs: high entropy, section names like `UPX0`, `.vmp`, or `.themida`, very few imports, or a protected .NET assembly (ConfuserEx, .NET Reactor).
3. **Unpack what you can.** UPX: `upx -d`. .NET: de4dot. PyInstaller: pyinstxtractor. Installers: extract with 7-Zip or innoextract instead of running them. Heavier protectors (VMProtect, Themida) usually mean dumping from memory in a debugger.
4. **If it won't come apart, stop.** Ask the author for an unpacked build or the source. A file you can't read is a file you can't trust.

## Step 3: Static analysis

Read the file without running it: IDA or Ghidra for native code, dnSpyEx or ILSpy for .NET. I look for these, and for each one I ask whether it fits what the tool claims to do.

- **Network:** `WinINet`/`WinHTTP` (`InternetOpen`, `HttpSendRequest`, `URLDownloadToFile`), `ws2_32` (`socket`, `connect`, `send`), or .NET `WebClient`, `HttpClient`, `Socket`.
- **Running other things:** `ShellExecute`, `WinExec`, `CreateProcess` launching cmd or PowerShell, .NET `Process.Start`.
- **Injection:** `VirtualAllocEx`, `WriteProcessMemory`, `CreateRemoteThread`, `NtUnmapViewOfSection`, `SetWindowsHookEx`. Normal for a trainer or injector, so check *which* process it targets.
- **Persistence:** registry `Run` keys, `schtasks`, service creation, Startup folder paths.
- **Credential theft:** strings pointing at browser profile folders, `Login Data`, `Cookies`, Discord `Local Storage`, wallet folders.
- **Suspicious strings:** URLs, IP addresses, `discord.com/api/webhooks`, `api.telegram.org`, paste sites, and base64 blobs (decode them).
- **Anti-analysis:** `IsDebuggerPresent`, timing checks, VM or sandbox detection. Not proof of malware on its own, but a reason to look harder.

For .NET, start at the entry point, find any string-decryption routine, and follow what the decrypted strings turn into.

## Step 4: Dynamic analysis in the VM

Run it only inside the snapshotted VM, with monitoring started first.

1. **Take a Regshot "before" shot** and start Process Monitor, Process Explorer, and Wireshark (or FakeNet-NG).
2. **Run it and use it** the way a normal user would, then leave it running for several minutes. Some malware waits before it acts.
3. **Watch for:**
   - DNS lookups or connection attempts you didn't expect
   - Files written to `%TEMP%`, `%APPDATA%`, or `ProgramData`, especially new executables or scripts
   - Child processes: cmd, PowerShell, `rundll32`, `mshta`, `regsvr32`, or anything it just dropped
   - Registry writes to `Run` keys or services
   - Handles opened on processes other than its stated target
4. **Take the "after" Regshot and diff it.** Run Autoruns to catch anything set to start on boot.
5. **Compare what it did with what it claims to do.** Anything extra is a stop sign.
6. **Revert the snapshot** when you're done.

## Hidden tricks to check for

These are the ways a bad file hides inside something that looks fine.

- **Double extensions:** `cheat.txt.exe` or `readme.pdf.scr`. Turn on "File name extensions" in Explorer so you always see the real one.
- **Shortcut and script files:** `.lnk`, `.bat`, `.cmd`, `.vbs`, `.js`, `.ps1`, `.hta` in an archive that should only hold a DLL or EXE. Open them in a text editor; a `.lnk` can hide a long PowerShell command in its target.
- **A working tool with a passenger:** the program does what it says, but also drops a second file in the background. Static analysis plus the Regshot diff catches this.
- **Delayed payloads:** a long `Sleep`, a date check, or a "run on the Nth launch" counter before anything bad happens. Look for these in static analysis instead of trusting a quick run.
- **Embedded resources:** executables or encrypted blobs stored in the PE's resources or a .NET manifest resource. Inspect them in Resource Hacker, PE-bear, or dnSpyEx.
- **Bundled DLLs:** a legitimate EXE shipped next to a DLL with a common name (`version.dll`, `winmm.dll`) so the EXE loads the attacker's copy. Check every DLL in the folder, not just the main file.
- **Alternate data streams:** `dir /r` shows hidden streams attached to a file.
- **Installers:** extract them (7-Zip, innoextract) and read the install script before running them. Check for extra programs or scheduled tasks.

## Verdict

- **Run it** only when it unpacked cleanly, its behavior matches its description, and it hit no stop signs. Save its SHA-256 so you can tell if a later copy differs.
- **Don't run it** if it hit any stop sign. Delete it and revert the VM.
- **Ask first** when it's too packed to read or does something you can't explain. Ask the author for source or an unpacked build, and wait.

When in doubt, don't. A missing tool costs you nothing; a stealer costs you every saved password.

## Quick reference

| Step | What to do | Stop if |
|---|---|---|
| Before opening | Source, SHA-256, VirusTotal hash search, signature | Sketchy source, named malware family, bad signature |
| Identify | Detect It Easy, real file type | Extension doesn't match the real type |
| Unpack | UPX, de4dot, pyinstxtractor, 7-Zip/innoextract | Can't unpack it far enough to read |
| Static | Imports, strings, decompile | Unexplained network, credential paths, persistence, droppers |
| Dynamic | Regshot, Procmon, Wireshark/FakeNet, Autoruns | Does anything it didn't claim to |
| Hidden tricks | Extensions, extra files, resources, side-loaded DLLs, ADS | Anything in the package you can't account for |

> If you find something malicious, report it where you got it (the forum's staff, the repo's issue tracker, or the hosting site) so it comes down before someone else runs it.
{: .prompt-tip }
