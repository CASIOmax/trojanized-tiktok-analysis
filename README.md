# Trojanized TikTok — Fake "TikTok Shop" Scam (Android)

Static analysis of a repackaged TikTok APK that injects attacker code into the real app and uses it as a shell to display a fake "TikTok Shop" investment/deposit scam.

> **Scope & ethics:** Static analysis only. No code from the sample was executed, and the attacker's servers were never contacted. The malware sample and the live recruitment codes are **intentionally not included** in this repository. Attribution clues are labelled as leads, not conclusions.

---

## TL;DR

Someone took the genuine TikTok app, unpacked it, added ~130 KB of their own Java code plus a native library, patched TikTok in ~15 places to call that code, and re-signed it with their own key. The result:

1. **Fakes its own integrity check.** It hooks Android's `PackageManager` and returns TikTok's *real* certificate (bundled inside the APK, disguised as a `.png`), so TikTok's anti-tamper checks pass.
2. **Injects "Shop / Products / Orders" buttons** into the user's profile page (labelled in ~30 languages).
3. **On tap,** sends the user's TikTok ID, avatar, device model/OS, timezone and language over **plain HTTP** to a PHP server on one of four rotating `.sbs` domains, then loads the attacker's web page full-screen inside TikTok.

No password or session-token theft was observed. The goal is to funnel victims into a deposit scam that looks like an official TikTok feature.

---

## Sample

| | |
|---|---|
| Masquerades as | TikTok 39.1.3 (`com.zhiliaoapp.musically`, versionCode 2023901030) |
| File size | 469,377,587 bytes |
| **SHA-256** | `21a61defbb4757a0ef8733267e1f2103e2d0c3c5bfeeabcfed0d3139fb541948` |
| SHA-1 | `35f59db6e25002044b84422733303fd88ab6fe5a` |
| MD5 | `4d5269e43ab14d8b19d76289ec88eb2d` |

---

## How it masquerades as genuine

Android apps are signed, and many check their own signature to detect tampering. This sample defeats that by bundling TikTok's real certificate and hooking the system `PackageManager`, so any self-check returns the genuine signature.

| Certificate seen on… | Subject | SHA-256 |
|---|---|---|
| This APK (attacker key) | `CN=Android, O=Android` | `4655e54d…e2d733` |
| Genuine TikTok (bundled as decoy) | `CN=musical.ly, O=musical.ly Inc.` | `9041803e…cf2ab5ba` |

The attacker's signing key was generated `2025-06-02` and reused on a build dated `2026-08-05`, so its fingerprint links any sample they sign.

---

## Execution flow

1. **App launch** — patched `AwemeHostApplication` calls the mod's `initFunction()`: disables Android hidden-API restrictions (via the open-source FreeReflection library), installs the `PackageManager` hook, and loads the genuine certificate from `assets/`.
2. **User opens their own profile** — `I18nAbsProfileFragmentV2` is patched to inject a **Shop / Products / Orders** bar.
3. **User taps a button** — the mod gathers profile + device fields and calls a native function to build the C2 URL.
4. **Full-screen WebView** opens over TikTok with JavaScript, DOM storage, file access and an upload picker enabled, and loads the attacker's page.

### Data sent on tap

```json
{
  "customID": "<TikTok user ID>",
  "avatar":   "<avatar URL>",
  "platform": "Android-TK-<model>-<brand>-<os>",
  "timezone": "<tz>",
  "lang":     "<locale>",
  "version":  "1.0.0"
}
```

Base64-encoded, transmitted over **plain HTTP**. `customID` ties the victim's "shop account" to their real TikTok profile.

---

## Hidden C2 address

The command-and-control address was placed in the native library `libtweak-mod.so` and obfuscated with **base64 wrapped around a single-byte XOR** — with the key (`0x4B`) stored in a symbol literally named `xor_key`. Decoded strings include the `http://` scheme, the four domains, the endpoint `/tkAndroid2.php?tokenId=`, and a hard-coded AES key.

Reconstructed request:

```
http://<dmszj|tzvet|iages|opiwq>.sbs/tkAndroid2.php?tokenId=<build id>&<base64 user JSON>
```

---

## Infrastructure (passive OSINT)

Collected from public registries (RDAP) and Certificate Transparency logs — no contact with the attacker's server.

| Property | Value |
|---|---|
| Domains → one IP | `dmszj.sbs`, `tzvet.sbs`, `iages.sbs`, `opiwq.sbs` → `118.107.59.222` |
| Registrar | NameSilo, LLC |
| All registered | `2024-12-02` (same day, 3-year term) |
| DNS | Cloudflare (DNS only — origin IP exposed, not proxied) |
| Host | CTG Server Ltd · APNIC · `abuse@ctgserver.net` |
| CT certs | 30+ per domain, wildcards; one issuer TrustAsia (CN) |

Batch registration on a single day, shared DNS and a shared origin indicate one operator. UTC+8 timestamps, Chinese UI text and a China-based certificate issuer are **leads, not proof**, of a Chinese-speaking operator.

---

## What it is — and isn't

| Question | Finding |
|---|---|
| Steals password / session token? | **No** — no login hooks, no credential exfiltration observed |
| Runs in the background? | **No** — only acts when the injected buttons are tapped |
| Extra permissions beyond TikTok? | **None** |
| Leaks user + device data in clear text? | **Yes** — over plain HTTP, on tap |
| Displays attacker-controlled pages inside a trusted app? | **Yes** — the core risk: a deposit-scam funnel |

---

## Indicators of Compromise

See [`IOCs.txt`](./IOCs.txt).

---

## If you installed it

- Uninstall it; reinstall TikTok only from Google Play.
- Settings → Security → **Manage devices**: remove anything unfamiliar; enable 2-step verification.
- **Do not deposit money** or pay "fees" to withdraw — that stage is the scam.
- Report the domains to Google Safe Browsing and the host's abuse contact; report the app to TikTok.

---

## Method

Unzip → binary-manifest decode → [jadx](https://github.com/skylot/jadx) decompilation of the injected dex → ELF/XOR string recovery from the native library → registry/CT OSINT. Tools: jadx, Ghidra, OpenSSL, RDAP, crt.sh.

## Reported to

- **VirusTotal** — https://www.virustotal.com/gui/file/21a61defbb4757a0ef8733267e1f2103e2d0c3c5bfeeabcfed0d3139fb541948

---

*Analysis & write-up: **Sajjad Ahmad** (`CASIOmax`). For educational and defensive purposes.*
