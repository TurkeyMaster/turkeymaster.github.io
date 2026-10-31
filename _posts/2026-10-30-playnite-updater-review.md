---
title: "How Playnite Updates Itself (and What It Trusts)"
date: 2026-10-30 12:00:00 -0700
categories: [Research, Launchers]
tags: [playnite, updaters, windows, code-review, supply-chain]
description: A source-level look at Playnite's auto-updater - where it gets updates, how it checks them, and the one check it has but never uses.
---

Auto-updaters are one of the most trusted pieces of code on a PC. They download a program from the internet and run it, often with admin rights, usually with one click from the user. If an updater trusts the wrong thing, everything it installs inherits that mistake.

[Playnite](https://github.com/JosefNemec/Playnite) is a popular open-source game library manager that pulls Steam, Epic, GOG and other launchers into one app. Since it's open source, I read through how its updater works. Everything here comes from the public source at commit [`3e8abb2`](https://github.com/JosefNemec/Playnite/tree/3e8abb241873e326e27c75b59dd0d2ea367698e3) (October 19, 2026), so you can check every line yourself.

> This is a design review of public code, not an exploit. I didn't attack Playnite's servers or tamper with anyone's install.
{: .prompt-info }

## The update flow

Here's the full path from "is there an update?" to "new version installed":

1. **Check on startup.** By default Playnite checks for program updates every time it launches ([`PlayniteSettings.cs` L2141](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Settings/PlayniteSettings.cs#L2141)).
2. **Download a manifest.** It fetches `update.json` from a URL built out of a config value, the update branch, and the current major/minor version. If the main URL fails, it tries a second one ([`Updater.cs` L180-212](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/Updater.cs#L180-L212)).
3. **Read the manifest.** It contains the latest version, a list of package download URLs, and a `Checksum` ([`UpdateManifest.cs`](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/UpdateManifest.cs#L9-L28)).
4. **Ask the user.** If there's a newer version, Playnite shows a notification. Nothing downloads until you click Install ([`UpdateViewModel.cs` L51-56](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/ViewModels/UpdateViewModel.cs#L51-L56)).
5. **Download the installer** to `%TEMP%\Playnite\update.exe`, trying each package URL in turn until one works ([`Downloader.cs` L232-249](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common/Web/Downloader.cs#L232-L249)).
6. **Check the file** against the manifest's checksum.
7. **Run it** silently, and as admin if Playnite's install folder isn't writable by the current user ([`Updater.cs` L169-178](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/Updater.cs#L169-L178)).

Steps 6 and 7 are where it gets interesting.

## The checksum only proves the download didn't break

Here's the whole verification step:

```csharp
private bool VerifyUpdateFile(string checksum, string path)
{
    var newMD5 = FileSystem.GetMD5(path);
    if (newMD5 != checksum)
    {
        logger.Error($"Checksum of downloaded file doesn't match: {newMD5} vs {checksum}");
        return false;
    }

    return true;
}
```
([`Updater.cs` L125-135](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/Updater.cs#L125-L135))

The expected MD5 comes from `update.json`, and `update.json` comes from the same update servers as the installer. So this check answers one question: did the file arrive intact? It catches a corrupted or cut-off download.

It doesn't answer the more important question: did this file come from the Playnite developer? Anyone who can change what the update server sends can change the installer and the checksum together, and the check passes. That could be someone who compromises the server or one of the mirrors, or someone who gets in the middle of a connection that isn't properly encrypted.

MD5 is also long broken for this kind of use, but that's a side point. Even SHA-256 would have the same problem here, because the hash and the file come from the same place.

## The signature check that never gets called

The interesting part: Playnite already has a function that would fix this.

```csharp
public static bool IsTrusted(string path)
{
    return WinVerifyTrust(path) == 0;
}
```
([`SigningTools.cs` L48-51](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common/SigningTools.cs#L48-L51))

`WinVerifyTrust` is the Windows API that checks a file's Authenticode signature, the same thing behind the "Digital Signatures" tab in a file's properties. A signature can only be made with the developer's private key, so unlike a checksum it proves who built the file.

I searched the whole codebase for callers. The only place `SigningTools.IsTrusted` is used is the test suite ([`SigningToolsTests.cs` L19-20](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Tests/Playnite.Tests/SigningToolsTests.cs#L19-L20)). The updater never calls it before running `update.exe`.

## How much does this matter?

Less than it sounds, but it's still worth fixing.

**What limits the risk:**
- **An attacker needs control of the update channel first.** That means a compromised server or mirror, or a network position on a connection that isn't protected. That's a high bar.
- **The user has to click Install.** Nothing downloads or runs on its own.
- **The UAC prompt is a last line of defense.** When the installer needs admin rights, Windows shows a prompt with the publisher name. A swapped, unsigned installer would show "Unknown publisher," which an alert user might notice.

**What I couldn't confirm from the source:**
- **The real update URLs.** The public config only has `localhost` placeholders ([`Common.config` L2-3](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common.config#L2-L3)), so release builds set them somewhere outside the repo. If they're all HTTPS, a network attacker can't swap files. If any mirror is plain HTTP, they can.
- **Whether release installers are signed.** If they aren't, there's nothing for `WinVerifyTrust` to check yet, and step one of the fix is signing them.

The bigger point is that the update server becomes a single point of failure for every Playnite install, and this isn't hypothetical. From June to December 2025, state-sponsored attackers who compromised Notepad++'s shared hosting [redirected selected users' update requests](https://www.theregister.com/2026/02/02/notepad_plusplus_intrusion/) to their own servers, which handed out malicious update manifests and installers, often named `update.exe`. The project's fix in version 8.8.9 was exactly the missing piece here: verifying the signature and certificate of the downloaded installer before running it.

## What a stronger updater would do

Pick one of these:

1. **Verify the Authenticode signature before running the installer.** Call the existing `IsTrusted()` on `update.exe`, then also check that the signing certificate belongs to the Playnite developer. Otherwise any validly signed file would pass.
2. **Sign the manifest.** Sign `update.json` with a private key and ship the matching public key inside Playnite. Then a compromised server can't produce a manifest the app will accept. Tools like minisign make this fairly simple.

And separately: switch the checksum to SHA-256, and make sure every update and mirror URL is HTTPS.

## Wrapping up

Playnite's updater gets the user-facing parts right. It asks before installing, falls back across mirrors, and checks downloads for corruption. What it doesn't do is check where the file came from, even though the code for that check is already sitting in the project.

When I look at any program's auto-updater, the question I start with is: **if the update server lied, would the app notice?** For a lot of updaters, including this one, the answer is no.

*Playnite is MIT-licensed. Code excerpts above are quoted from the Playnite repository, copyright Josef Nemec.*
