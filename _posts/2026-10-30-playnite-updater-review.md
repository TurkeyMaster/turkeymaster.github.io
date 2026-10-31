---
title: "How Playnite Updates Itself (and What It Trusts)"
date: 2026-10-30 12:00:00 -0700
categories: [Research, Launchers]
tags: [playnite, updaters, windows, code-review, supply-chain]
description: A source-level look at the three ways Playnite 10.63 downloads and runs code - the app updater, add-ons, and the online installer - and what each one actually verifies.
---

Auto-updaters are one of the most trusted pieces of code on a PC. They download a program from the internet and run it, often with admin rights, usually with one click from the user. If an updater trusts the wrong thing, everything it installs inherits that mistake.

[Playnite](https://github.com/JosefNemec/Playnite) is a popular open-source game library manager that pulls Steam, Epic, GOG and other launchers into one app. I read through every path it uses to download and run code. Everything here comes from the public source for **Playnite 10.63**, the current release, at commit [`3e8abb2`](https://github.com/JosefNemec/Playnite/tree/3e8abb241873e326e27c75b59dd0d2ea367698e3) (October 19, 2026), so you can check every line yourself.

> This is a design review of public code, not an exploit. I didn't attack Playnite's servers, tamper with anyone's install, or upload anything to the add-on database.
{: .prompt-info }

## Three ways code gets onto your PC

Playnite has three separate download paths, and they don't verify things the same way:

| Path | What it downloads | What it checks before running it |
|---|---|---|
| App updater | A new Playnite installer | An MD5 checksum from the update manifest |
| Add-ons | Extension packages (DLLs, scripts, themes) | Package structure only, no checksum |
| Online installer | The full Playnite installer | Nothing |

None of them check a code signature, even though Playnite already has a function that does exactly that. More on that below.

## 1. The app updater

Here's the full path from "is there an update?" to "new version installed":

1. **Check on startup.** By default Playnite checks for program updates every time it launches ([`PlayniteSettings.cs` L2141](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Settings/PlayniteSettings.cs#L2141)).
2. **Download a manifest.** It fetches `update.json` from a URL built out of a config value, the update branch, and the current major/minor version. If the main URL fails, it tries a second one ([`Updater.cs` L180-212](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/Updater.cs#L180-L212)).
3. **Ask the user.** If there's a newer version, Playnite shows a notification. Nothing downloads until you click Install ([`UpdateViewModel.cs` L51-56](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/ViewModels/UpdateViewModel.cs#L51-L56)).
4. **Download the installer** to `%TEMP%\Playnite\update.exe`, trying each package URL in turn until one works ([`Downloader.cs` L232-249](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common/Web/Downloader.cs#L232-L249)).
5. **Check the file** against the manifest's checksum.
6. **Run it** silently, and as admin if Playnite's install folder isn't writable by the current user ([`Updater.cs` L169-178](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/App/Updater.cs#L169-L178)).

The URLs in the public config are just `localhost` placeholders ([`Common.config` L2-3](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common.config#L2-L3)), but following the URL pattern in the code leads to the live manifest at `https://playnite.link/update/stable/10.63/update.json`:

```json
{
  "Version": "10.63",
  "Checksum": "D6CE9DE1BA01BD8A37F29349B1AEB058",
  "PackageUrls": [
    "https://github.com/JosefNemec/Playnite/releases/download/10.63/Playnite1063.exe",
    "https://www.playnite.link/update/stable/10.63/Playnite1063.exe"
  ]
}
```
(Trimmed. The real file also lists SDK and theme versions.)

Good news first: everything is HTTPS, and there are two mirrors.

### What the checksum does and doesn't protect

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

Because the manifest lives on playnite.link and the first download mirror is GitHub, this check does real work. Someone who tampered with only the GitHub release file would fail the MD5 check. It also catches broken downloads.

What it can't handle is a compromised manifest server. Whoever controls `update.json` controls both the expected checksum and the download URLs, so they can point Playnite at any file they like with a matching hash. Switching from MD5 to SHA-256 wouldn't change that, because the hash and the URLs come from the same place. (MD5 should still be retired, but that's a side point.)

### The signature check that never gets called

Playnite already has the function that would close this gap:

```csharp
public static bool IsTrusted(string path)
{
    return WinVerifyTrust(path) == 0;
}
```
([`SigningTools.cs` L48-51](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Common/SigningTools.cs#L48-L51))

`WinVerifyTrust` is the Windows API that checks a file's Authenticode signature, the same thing behind the "Digital Signatures" tab in a file's properties. A signature can only be made with the developer's private key, so unlike a checksum it proves who built the file, no matter which server delivered it.

I searched the whole codebase for callers. The only place `SigningTools.IsTrusted` is used is the test suite ([`SigningToolsTests.cs` L19-20](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Tests/Playnite.Tests/SigningToolsTests.cs#L19-L20)). The updater never calls it before running `update.exe`.

And right now it wouldn't have anything to check: the 10.63 installer isn't signed at all (see [What I verified](#what-i-verified) below).

## 2. Add-ons: hundreds of developers in the trust chain

Add-ons are where Playnite's trust surface gets much bigger. Extensions are DLLs or PowerShell scripts that Playnite loads directly ([`ExtensionFactory.cs` L465](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Plugins/ExtensionFactory.cs#L465)), so a malicious add-on runs with all of your user account's permissions.

Here's how an add-on update works:

1. **Check on startup.** Add-on update checks are also on every startup by default ([`PlayniteSettings.cs` L2152](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Settings/PlayniteSettings.cs#L2152)).
2. **Ask Playnite's API** for each add-on's installer manifest, which lists versions and a `PackageUrl` for each ([`ServicesClient.cs` L69-72](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Services/ServicesClient.cs#L69-L72)).
3. **Show the user** a list of available updates, all ticked by default ([`Addons.cs` L99](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Addons/Addons.cs#L99)).
4. **Download each package** from its `PackageUrl` and queue it for install on the next restart ([`AddonsViewModelBase_Online.cs` L110-123](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/ViewModels/AddonsViewModelBase_Online.cs#L110-L123)).

Notice what step 4 doesn't have. The package record has a version, a URL, a required API version, a date and a changelog, but no checksum field at all ([`AddonManifestBase.cs` L49-56](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Manifests/AddonManifestBase.cs#L49-L56)). Whatever file is at that URL gets installed.

So where do those manifests come from? Playnite's [add-on database](https://github.com/JosefNemec/PlayniteAddonDatabase) is public, so I counted (commit `5c61241`, October 20, 2026):

| Installer manifest location | Add-ons |
|---|---|
| A file on a live branch of the developer's own repo | 395 |
| Hosted by Playnite itself (mostly built-in add-ons) | 18 |
| The developer's "latest release" download | 2 |
| The developer's own cloud storage | 1 |
| Pinned to a specific commit | 1 |
| **Total** | **417** |

Every one of them is HTTPS, which is good. But for 395 add-ons, the file that decides what gets downloaded lives in someone else's repo, on a branch that can change at any time. As far as I can tell from the client side, if one developer's GitHub account gets taken over, the attacker can point that add-on's next "update" at any package they want, and Playnite will offer it to every user of that add-on with the box already ticked.

To be fair, Playnite does have defenses here:
- **There's a kill switch.** Playnite fetches an add-on blacklist from its own API and hides updates for anything on it ([`Addons.cs` L131-132](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Addons/Addons.cs#L131-L132)). Once a bad add-on is spotted, it can be pulled fast.
- **Packages get basic checks.** Extensions can't ship their own copies of Playnite's core DLLs ([`ExtensionInstaller.cs` L357](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Plugins/ExtensionInstaller.cs#L357)).
- **Zip Slip doesn't work.** Packages are extracted with .NET's `ZipFile.ExtractToDirectory` ([`ExtensionInstaller.cs` L213](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Playnite/Plugins/ExtensionInstaller.cs#L213)), which refuses entries that would land outside the target folder. A package with `../../` paths can't write files elsewhere on disk.

None of these check that the package is the one the developer actually released, though. The blacklist only helps after someone notices.

## 3. The online installer: no check at all

Playnite also ships a small web installer that downloads the real installer for you. It reads a mirror list (HTTPS, on playnite.link), downloads the installer, and runs it with `/VERYSILENT` ([`MainViewModel.cs` L188-203](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/source/Tools/PlayniteInstaller/MainViewModel.cs#L188-L203)).

There's no checksum and no signature check anywhere in that flow. Whatever the mirror list points to gets run. Windows will still show a UAC prompt if the installer asks for admin rights, but since the real installer is unsigned, that prompt already says "Unknown publisher." A swapped installer would look exactly the same.

## What I verified

Reading code only gets you so far, so I checked the actual 10.63 release on a Windows machine:

- **The MD5 matches.** I downloaded `Playnite1063.exe` from the GitHub release, and `Get-FileHash -Algorithm MD5` returned `D6CE9DE1BA01BD8A37F29349B1AEB058`, exactly what the live manifest lists. The mirror and the manifest agree.
- **The installer isn't signed.** `Get-AuthenticodeSignature` returned `NotSigned`, and there's no certificate attached.
- **The installed app is signed.** On my installed copy, `Playnite.DesktopApp.exe` has a valid signature from a SignPath Foundation certificate, issued by GlobalSign.

```text
PS> Get-AuthenticodeSignature .\Playnite1063.exe | Format-List Status, SignerCertificate

Status            : NotSigned
SignerCertificate :

PS> Get-AuthenticodeSignature "$env:LOCALAPPDATA\Playnite\Playnite.DesktopApp.exe" | Format-List Status, SignerCertificate

Status            : Valid
SignerCertificate : [Subject]
                      CN=SignPath Foundation, O=SignPath Foundation, L=Lewes, S=Delaware, C=US
                    [Issuer]
                      CN=GlobalSign GCC R45 CodeSigning CA 2020, O=GlobalSign nv-sa, C=BE
                    [Thumbprint]
                      E491F092639A94BA474A1B8BD14D2CFA4B7A9059
```

The unsigned installer surprised me, because Playnite's README credits [SignPath](https://signpath.org) for free code signing ([`README.md` L103](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/README.md?plain=1#L103)). The CI config explains it: AppVeyor sends `build\Playnite.zip` to SignPath's release-signing policy ([`appveyor.yml` L16-20](https://github.com/JosefNemec/Playnite/blob/3e8abb241873e326e27c75b59dd0d2ea367698e3/appveyor.yml#L16-L20)), and that zip holds the app's own files, which is why the installed `Playnite.DesktopApp.exe` checks out. The setup `.exe` that the updater and online installer actually run isn't built in this repo, and it isn't signed.

So the signing setup already exists. It just doesn't cover the one file that matters most for updates. That also means the unused `IsTrusted()` check couldn't help today even if it were wired in, and users have no publisher name to look for in the UAC prompt.

## How much does this matter?

Less than it sounds, but it's worth fixing.

**What limits the risk:**
- **An attacker needs to control a server or an account first:** playnite.link, a mirror, or an add-on developer's GitHub. That's a high bar for the app itself, and a lower one for the long tail of add-ons.
- **HTTPS is used everywhere I could check**, so a network attacker on your Wi-Fi can't swap files in transit.
- **The user clicks Install** for both app and add-on updates. Nothing installs entirely on its own.

**Why it still matters:** each of those servers and accounts is a single point of failure for every user downstream, and this isn't hypothetical. From June to December 2025, state-sponsored attackers who compromised Notepad++'s shared hosting [redirected selected users' update requests](https://www.theregister.com/2026/02/02/notepad_plusplus_intrusion/) to their own servers, which handed out malicious update manifests and installers, often named `update.exe`. The project's fix in version 8.8.9 was exactly the missing piece here: verifying the signature and certificate of the downloaded installer before running it.

## What a stronger setup would do

**For the app updater and the online installer**, pick one:
1. **Sign the installer too, then verify it.** Playnite already has free signing through SignPath, so the setup `.exe` could go through the same pipeline as `Playnite.zip`. Then have both the updater and the online installer call the existing `IsTrusted()` before running it, and check who signed it. Otherwise any validly signed file would pass. There's a catch: the certificate is issued to "SignPath Foundation," not Playnite, and as far as I know the foundation signs other open-source projects too. Checking the subject name alone wouldn't prove a file came from Playnite, and pinning the thumbprint breaks every time the certificate renews.
2. **Or sign the manifest, which avoids that catch.** Sign `update.json` with a private key (tools like minisign make this simple) and ship the matching public key inside Playnite. A compromised server then can't produce a manifest the app will accept, and the existing checksum check takes care of the rest. Switch that checksum to SHA-256 while you're there.

**For add-ons:**
1. **Record a SHA-256 hash for each package version** when the Playnite API first sees it, and refuse a package that doesn't match. That pins each release in place, so changing a developer's manifest later can't swap the file behind an existing version.
2. **Encourage pinned manifests.** Pointing at tagged releases instead of a live `main` branch makes a quiet swap much harder.

## Wrapping up

Playnite gets a lot right: HTTPS everywhere, user confirmation, mirror fallbacks, an add-on kill switch, and safe zip extraction. What none of its three download paths do is check *who* built the file. Both halves of the fix are already sitting in the project: a signature check that nothing calls, and a free signing pipeline that skips the installer. Connecting them would close the biggest gap.

When I look at any program's auto-updater, the question I start with is: **if the update server lied, would the app notice?** For Playnite's app updater, only if a mirror lied and the manifest didn't. For add-ons and the online installer, no.

> I reported these findings and suggestions to the Playnite developer in [issue #4417](https://github.com/JosefNemec/Playnite/issues/4417). I'll update this post if anything changes.
{: .prompt-tip }

*Playnite is MIT-licensed. Code excerpts above are quoted from the Playnite repository, copyright Josef Nemec.*
