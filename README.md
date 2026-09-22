# always

**A self-hosted AI workbench for the browser, with a lightweight Android WebView shell.**

> This repository contains only the native Android shell. The **always** platform is a separate Next.js application; this project packages it as a minimal WebView container. The README first describes the platform, then documents the shell. [中文文档](README.zh.md)

## What is always

always consolidates multi-model chat, web search, code generation, image creation, and long-form writing in one workspace. It is a PWA, so a single session can move from a desktop browser to a phone or the Android shell.

Tagline, straight from the app: *multi-model chat, web search, code generation, and image creation — all in one place.*

### Three work modes

The application provides three purpose-built work modes, each with its own layout and toolset:

| Mode | What it's for | Highlights |
| --- | --- | --- |
| **Chat** | Everyday conversation and research | Multi-model conversation, knowledge Q&A, web search, image understanding |
| **Coding** | Working on code | Code generation, debugging, and review, with tool-token optimization so long tool outputs don't eat the context |
| **Writing** | Novels and long-form pieces | Chapter management, a prose-style profile, and hierarchical-memory injection so the assistant keeps continuity across chapters |

### Beyond the three modes

- **Image generation** with a gallery for what you've made
- **Repository review** — point it at a repo and get a pass over the code
- **Knowledge base** you can attach to conversations
- **Web extraction** — pull a page's content in as context
- **GitHub binding** for the review flow
- **Hierarchical memory** integration, so past conversations are retrieved and injected automatically rather than dumped in wholesale
- **Multi-user accounts with LAN login**, so a self-hosted instance can be shared safely
- **Light and dark themes**, and a PWA manifest for install-to-home-screen

### Deployment shape

always is designed to be self-hosted. The web app runs under plain Node (`server.js`) behind a single port, with the pages served by Next.js and the backend exposed as API routes (auth, chat, conversations, models, images, knowledge base, code review, crawling, GitHub, and the memory service, among others). A typical setup is one host running the web app, with Ollama and a local memory service alongside it. The author's own instance is served over plain HTTP on a non-standard port and is used as the shell's default target — see the security notes below before you copy that arrangement.

## The Android shell (this repo)

This is a single-Activity WebView container that wraps the always web app as a native Android app. The shell itself has zero business logic — all capabilities live and evolve on the web side. That's why **the shell code needs almost no maintenance**: once the website updates, the APK doesn't need to be rebuilt, a refresh is all it takes.

> Defaults to the author's demo server (HTTP). After cloning this repo, please update `START_URL` in `java/com/aiplatform/app/MainActivity.java` before building — see below.

### Features (implemented at the shell layer)

- **Cleartext HTTP allowed** (`usesCleartextTraffic`): the demo server is HTTP; without this, Android 9+ shows a white screen
- **JS dialog interception** (`onJsAlert / onJsConfirm / onJsPrompt`): web-side `window.confirm` calls (e.g. "unbind", "submit code") fail silently if not intercepted
- **GitHub OAuth works**: all http/https navigation stays inside the WebView, so auth redirects don't jump out to the system browser; UA is spoofed as a modern Chrome
- **File picker** (`onShowFileChooser`): supports image/attachment uploads from the web-side chat
- **Download support** (DownloadListener): files exported from the web side go through the system download manager
- **Login-state persistence**: `CookieManager.flush()` on `onPause`

### Build Steps

Dependencies: JDK 17, Android SDK (build-tools 34 + platform android-34), Python 3 (Pillow, for icon generation).

1. Set the target site: `java/com/aiplatform/app/MainActivity.java` → `START_URL`
2. Generate icons: `python make_icons.py` (outputs to `res/mipmap-*`)
3. Build: run `.\build-apk.ps1` in PowerShell
   (manual chain: `aapt2 compile/link → javac → d8 → zipalign → apksigner`, output goes to `..\always.apk` outside the repo)

Adjust the `SDK / PLATFORM / JDK` paths at the top of the script to match your local setup. The signing key is not shipped with the repo; the script will auto-generate one on first build (storepass/keypass are in the script's constants) — **keep your own keystore safe and never commit it**.

### Security Notes

- This shell loads an HTTP site and has cleartext traffic enabled — if your site is on HTTPS, remember to tighten this
  (remove `usesCleartextTraffic` or switch to `MIXED_CONTENT_NEVER_ALLOW`)
- The WebView trusts web-side content by default; make sure the server you're loading is your own

### Directory Structure

```
AndroidManifest.xml                      Manifest (usesCleartextTraffic / permissions / Activity)
java/com/aiplatform/app/MainActivity.java   All shell logic (~300 lines)
res/mipmap-*/ic_launcher.png             Launch icons (generated by make_icons.py)
make_icons.py                            Icon generation script (Pillow)
add_dex.py                               Injects classes.dex into base.apk
build-apk.ps1                            One-click build script
```

### Security Boundaries (honest disclosure)

This code is a **personal-use / demo** shell with an intentionally simple security model. Please be aware of the following boundaries:

- **Address obfuscation is not encryption**: `START_URL` is stored via XOR + Base64, with the key baked right into the binary — it only prevents "glancing at the decompiled code and seeing the server address at a glance." Anyone willing to spend a minute can recover it. Don't treat it as a confidentiality mechanism.
- **Entirely cleartext end-to-end**: the backend currently runs over HTTP (bare IP + non-standard port), and the shell allows cleartext traffic — meaning an attacker on the same subnet can see login state and content in plain sight. The SSL handling (`onReceivedSslError` rejects by default) exists **in preparation for a future HTTPS migration**; it provides no protection under the current setup.
- **Port 3389 is a historical accident**, not a deliberate RDP disguise: when deployed, the cloud host's security group only opened 22/3389, so it was reused directly. It will switch back to 443 once HTTPS is in place.
- **Not recommended for distribution as a production client**: if you plan to distribute it, put the backend on HTTPS first (once the certificate is configured, fill the certificate's public-key hash into `<pin-set>` in `network_security_config.xml` to complete certificate pinning).

### Roadmap

1. Backend HTTPS (domain or self-signed + certificate pinning)
2. Already implemented server-side: multi-user data isolation, forced auth in public mode, generated images no longer served unauthenticated (see the server-side private repo)
