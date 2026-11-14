# Poki security audit / Poki 安全审计

**Date / 日期:** 2026-11-14  
**Repository / 仓库:** [Lin00700/poki](https://github.com/Lin00700/poki) (public fork of [JosssphZhou/poki](https://github.com/JosssphZhou/poki))  
**Audited tree / 审计的源码:** `main` @ `412f612fe8169645b853d2aaa7af83de5f323dd3`  
**Release under test / 被检查的发布:** [`app-0.4.0-beta.1`](https://github.com/JosssphZhou/poki/releases/tag/app-0.4.0-beta.1)  
**License / 许可证:** AGPL-3.0-only

This is a source-and-artifact review. It is not a legal opinion and not a full binary reverse-engineering of every linked library.

这是一次源码与发布文件审查，不是法律意见，也不是对每一个链接库的完整逆向。

---

## Verdict / 结论

**Caution. Safe enough to self-sign for personal use if you treat it as a high-privilege phone operator, the same class of risk as handing someone your unlocked phone plus your model key.**

**谨慎使用。可以给自己的手机自签安装，但必须把它当成一个高权限的“代操手机”工具，不能当成普通 App Store 应用。**

No backdoor, hardcoded command-and-control server, committed secret, or covert exfiltration endpoint was found in this fork or in the `app-0.4.0-beta.1` assets that were inspected.

在这个 fork 以及抽查的 `app-0.4.0-beta.1` 发布文件里，没有发现后门、写死的远控地址、入库的密钥，或隐蔽的数据外传地址。

The fork is not a modified copy. GitHub compare of `JosssphZhou/poki` `main` and `Lin00700/poki` `main` is **identical** (`ahead_by: 0`, `behind_by: 0`) at `412f612`. The fork was created on 2026-11-14 and adds no commits of its own. Trusting this fork is the same question as trusting upstream `JosssphZhou/poki` at that commit. A later push to either repository is outside this audit.

这个 fork 没有改过代码。GitHub 对比显示它和上游 `main` **完全一致**（`412f612`，领先 0、落后 0）。Fork 创建于 2026-11-14，没有自己的提交。信任这个 fork，等于信任上游在该提交上的代码。之后任何一边再推送，都不在本次审计范围内。

The reason for **Caution** is the product itself:

选择 **谨慎** 的原因是产品设计本身：

- It is built to read the screen, tap, type, and scroll in other apps, including private UI automation (XCTest / HID).
- Screenshots and screen text are sent to the model account you configure. If you turn on a relay, that same traffic is plaintext inside your Cloudflare Worker.
- “Never ask” lets password typing, message sending, and calls proceed without a prompt. Payments, deletions, and Wallet still ask.
- A free Apple ID signature lasts about 7 days, and development signing normally includes `get-task-allow` (a debugger can attach).
- Sign-in on Windows and on the iPhone uses community anisette servers. They do not receive the Apple ID password, but they do provision a machine identity.
- The published IPA and desktop installers are sideload builds from GitHub Actions, not App Store / notarized consumer apps. The IPA hash was checked. The desktop `.dmg` and `.exe` were not executed or disassembled.

中文要点：

- 它的用途就是读取屏幕、点击、输入、滚动，底层是私有的 XCTest / HID 自动化。
- 截图和屏幕文字会发给你自己配置的模型。如果启用中继，这些内容会以明文经过你的 Cloudflare Worker。
- “不再询问”会让输入密码、发消息、打电话直接执行。付款、删除、钱包仍然会问。
- 免费 Apple ID 签名大约 7 天，开发签名通常带 `get-task-allow`，调试器可以附加。
- Windows 和 iPhone 上的登录会用社区 anisette 服务器。它们收不到 Apple ID 密码，但会参与设备身份配置。
- 发布的 IPA 和桌面安装包是 GitHub Actions 打出来的侧载包，不是 App Store / 公证过的消费级应用。IPA 哈希已核对。桌面 `.dmg` 和 `.exe` 没有运行，也没有反汇编。

---

## What was checked / 检查了什么

| Item | Result |
| --- | --- |
| Fork vs upstream `main` | Identical at `412f612` |
| `app-0.4.0-beta.1` tag | Lightweight tag → commit `cda35c49ab5c32fa03a4a8b19330ec060b251302` |
| `app-0.4.0-alpha.1260` | Annotated tag → **same commit** `cda35c49`. Release notes are accurate: Beta 1 is Alpha build 1260, unchanged |
| Commits on `main` after that tag | 78. Relay change is a WebSocket ready-state check only. No new outbound URL in product code |
| `Poki-external-0.4.0-beta.1.ipa` | SHA-256 `a361860dcf9fb59de13c4465717f77947a27659aec00c144c93e360983d8fffc`, matches `SHA256SUMS` and the GitHub asset digest |
| IPA `Info.plist` | `CFBundleShortVersionString` **0.4.0**, `CFBundleVersion` **1260**, bundle `nz.zheha.poki`, iOS 18+, built with Xcode 26.5 (`iphoneos26.5`). No `embedded.mobileprovision` and no `_CodeSignature` (unsigned external IPA, meant to be re-signed) |
| IPA plugins | `PokiWidgets.appex` only. The external build does **not** embed the packet-tunnel extension |
| `poki-relay-worker.js` | SHA-256 `394450cb399d4b4bce6d9fecad0f9f8cc1aa6fc7b0ddae17247d13507889a532`, **byte-identical** to `deploy/cf-relay/worker.js` at `cda35c49` |
| Secrets scan | No private keys, `AKIA…`, `ghp_…`, `sk-…`, or `xox…` tokens in the tree |
| Binaries executed | None. The IPA was unzipped and string-scanned. Desktop installers were not run |

---

## Architecture / 架构

```text
iPhone app (native/ios-app + PokiCore)
  ├─ in-process XCTest/HID automation (screenshots, taps, typing)
  ├─ chat / voice → the model provider you choose (PiSwift, pinned revision)
  ├─ optional outbound WebSocket → your Cloudflare relay → MCP clients
  └─ self-update: GitHub release IPA → SHA-256 → re-sign with your Apple ID → install over 10.7.0.1

Computer, one time (desktop/ Poki Setup, or the Mac MCP path in src/ and agent/)
  ├─ USB pairing, developer disk image, install / renew the 7-day signature
  └─ optional local MCP daemon on 127.0.0.1 (screenshots and input for a desktop agent)

Loopback 10.7.0.1
  └─ reflects the phone's own developer services (lockdown / RemotePairing) back to itself
     External IPA: LocalDevVPN or a proxy app. Full Xcode build can use Poki's own tunnel.
```

`third_party/` is license notices. `skills/` is Markdown for agents. `deploy/cf-relay/` is the Worker you deploy yourself. There is no default relay host in the app (`https://poki-relay.you.workers.dev` and `https://your-relay.example` are placeholders).

---

## Findings / 发现

| Sev. | Path | What it does | Intentional? |
| --- | --- | --- | --- |
| Info | whole fork vs `JosssphZhou/poki@412f612` | No fork-only commits or files | Yes. Fresh mirror |
| Info | repo-wide secret scan | No committed API keys, private keys, or Apple passwords | Yes |
| High (by design) | `native/ios-app/Poki/Native/RecoveredNative.m`, `native/ios/PokiCore` | Private XCTest/HID: screenshots, taps, typing, clipboard | Yes. This is the product |
| High (by design) | `PokiCore` `Computer.swift` / `Core.swift`, `OpenAIClient.swift`, `ModelClient.swift` | JPEG screenshots and accessibility text go to your model API | Yes. Required for the chat agent |
| High (by design) | `deploy/cf-relay/src/index.js`, `PRIVACY.md` | Your Worker sees MCP bodies in memory (tool calls, screenshots). TLS ends at Cloudflare. It does not store request bodies. The only Durable Object storage is a daily request counter. Connector secret is in the URL path and is hashed, not stored | Yes. Documented. Whoever can edit the Worker can read the traffic |
| Medium | `Approvals.swift` `asked(.off)` | Never Ask turns off password-field, send, and call prompts. Payments, deletions, and `com.apple.Passbook` still wait | Yes. Tested in `CoreTests.swift` |
| Medium | `ApprovalClassifier.swift` | On reply/confirm, if sensitivity is not “ask everything”, the chat model may answer ALLOW/REFUSE for steps that are not mandatory. Password-field steps are not in the always-owner set | Yes. The model can approve a password step |
| Medium | `desktop/core/src/apple/anisette.rs` | Default anisette hosts: `new.ani.sidestore.io`, fallback `mac-ani.jkcoxson.com`, `ani2.stikstore.app`. They receive provisioning material, not the Apple ID password. GSA uses SRP to `gsa.apple.com` | Yes. Same pattern as SideStore / AltServer. macOS can use local AOSKit on older OS versions; the code defaults to a remote server because newer macOS does not hand out AOSKit headers |
| Medium | `desktop/core/src/apple/store.rs` | If you keep the Apple ID signed in, the password can be stored in the local keychain / Credential Manager so weekly renew can run | Yes. Optional. The machine can read it |
| Medium | `bin/voice-bridge.mjs` | Optional Mac voice bridge defaults to `0.0.0.0`. Auth is a secret in the URL path. Not part of the IPA | Yes. Do not expose it beyond your LAN |
| Medium | `src/daemon.mjs` + `src/data/tools.json` | Local MCP defaults to `127.0.0.1` plus a bearer token. Tools include screenshot, type, open URL, install/uninstall, container download. Most taps are not behind a macOS approval dialog | Yes. The computer-side agent path |
| Low | `ProxyApps.swift` | Surge/Loon modules are loaded from `raw.githubusercontent.com/JosssphZhou/poki/main/deploy/proxy-modules/` (unpinned `main`). The files in-tree only reflect `10.7.0.1` | Yes, but `main` can change later |
| Low | `desktop/core/src/device/ddi.rs` | Developer disk images come from `raw.githubusercontent.com/doronz88/DeveloperDiskImage` | Yes. Third-party supply chain for the DDI |
| Low | `src/update-check.mjs`, `desktop/core/src/release.rs` | Update metadata and IPAs come from `api.github.com` / `github.com` releases of `JosssphZhou/poki`. IPA install checks `SHA256SUMS` before re-sign. There is no separate Apple notarization of the upstream blob | Yes |
| Low | `project-base.yml` | `NSAllowsLocalNetworking` only (not arbitrary loads). Background modes: location, audio, processing. Location strings say location is not recorded or sent. Microphone string says voice audio goes to OpenAI | Yes |
| Low | `project.yml` vs `project-external.yml` | Full build asks for `packet-tunnel-provider`. The **published external IPA** omits that entitlement and has no tunnel appex | Yes. External users bring LocalDevVPN or a proxy |
| Info | `entitlements.rs` | When your Apple development profile contains `get-task-allow`, the re-signed app keeps it. Free accounts use development profiles | Yes. Sideload residual risk |
| Info | IPA strings | Hosts match Apple (`gsa.apple.com`, `developerservices2.apple.com`, `itunes.apple.com`), GitHub, anisette, and the pinned PiSwift provider catalog (OpenAI, Anthropic, Gemini, Groq, Cerebras, Fireworks, xAI, Moonshot, and others). Placeholders: `192.168.1.10` (voice-bridge example), `127.0.0.1`. No pastebin, Discord webhook, or Telegram bot token | Catalog is PiSwift `f8696524`, MIT, from `project-base.yml`. Those hosts are used when that provider is selected |
| Info | `agent/telegram-source.mjs` | Telegram is an optional Poki Agent channel. Default `https://api.telegram.org`. A custom `TELEGRAM_API_URL` must be that host or loopback | Yes. Not in the iPhone app |
| Info | `POKI_IPHONE_PASSCODE` | Mac agent only, env var, passed to the unlock helper on stdin. The iOS app does not store or upload the device passcode | Yes. Opt-in, and only if you set it |

Nothing in the first-party JavaScript or Swift uses `eval` / `new Function` to run a downloaded payload. `run_code` on the Mac MCP path is a `vm` sandbox with `fetch` and `require` removed, and it is off the default tool list.

第一方 JS / Swift 里没有用 `eval` 去执行下载下来的载荷。Mac 上的 `run_code` 是去掉了 `fetch` 和 `require` 的 `vm` 沙箱，而且不在默认工具列表里。

---

## What leaves the phone, and where / 什么数据会离开手机

| Data | Where it goes | When |
| --- | --- | --- |
| Screenshot JPEG + accessibility tree + the action the agent took | Your model API (OpenAI, Anthropic, DeepSeek, Gemini, OpenRouter, or a custom URL) | You use Poki's own chat or voice |
| Same screen contents | Your Cloudflare Worker, then the MCP client you connected (Claude, ChatGPT, Cursor, …) | You set a relay and a client calls it |
| Chat text, attached photos, first pages of an attached PDF | The selected model only | You send them |
| Microphone audio | OpenAI Realtime, or the voice-bridge URL you type | You start voice |
| Apple ID | `gsa.apple.com` via SRP (password is not sent as plaintext). Portal calls go to `developerservices2.apple.com` | You sign in to renew or install |
| Anisette provisioning (machine id, ADI blob, one-time headers) | `new.ani.sidestore.io` or a fallback, unless macOS local AOSKit works | Sign-in on Windows, on iPhone, or when local headers are unavailable |
| Model API key / ChatGPT OAuth material | Keychain on the phone; the key is sent only to the provider you picked | You add an account |
| Relay Ed25519 key and connector secrets | Keychain. The secret appears in the connector URL you paste into the MCP client | You create a connector |
| Pairing records | On the phone, used over `10.7.0.1` to the phone's own services | Setup |
| Device unlock passcode | Not uploaded by the iOS app | — |
| Location | Not read for tracking. The location background mode is a keep-alive. Usage strings say it is not sent | Always declared |
| Contacts / photo library bulk read | No such call. Photos leave only if you attach them, or if the agent is looking at the Photos app on screen | — |
| Update check | `api.github.com/repos/JosssphZhou/poki` (anonymous, no device id in the client code) | Startup / daily, and when you tap update |
| App icons | `https://itunes.apple.com/lookup` | Icon cache |
| Clipboard text | Available to the automation tools, so it can appear in a screenshot/tree sent to the model or relay | When a tool reads the pasteboard |

`10.7.0.1` is not a remote server. It is a virtual address that must be reflected back to the phone so Poki can talk to its own developer services. The Surge and Loon modules in this repo only add `10.7.0.1/32` reflect. A third-party VPN app can still see other traffic if you use it as your normal proxy. That is the VPN app's power, not a hidden Poki upload.

`10.7.0.1` 不是远程服务器，而是一个必须被反射回手机本机的虚拟地址，用来访问手机自己的开发者服务。仓库里的 Surge / Loon 模块只反射 `10.7.0.1/32`。如果你把同一款 VPN 同时当日常代理，它仍然能看到其他流量。那是 VPN 应用自己的权限。

---

## Relay: can it steal tokens? / 中继能不能偷令牌

The Worker is a reverse proxy you deploy (`deploy/cf-relay`, `wrangler.toml` has `observability.enabled = false`).

- There is no baked-in relay. Nothing is sent until you enter your Worker URL.
- Phone connect is an Ed25519 challenge (`poki-relay-v1`). The label is the base32 of the phone's public key.
- MCP clients use the connector URL (`/v1/<label>/<secret>/mcp`). OAuth tokens are stateless HMAC (`TOKEN_SECRET` is a Wrangler secret, not in git). PKCE S256 is required.
- Unknown HTTPS redirect hosts wait for an Allow on the phone (`CONSENT_MS`, 90s) unless you set `REDIRECT_HOSTS`.
- `ALLOW_SECRET_ONLY=1` drops the bearer check and leaves the URL secret as the only gate. Leave that unset.
- The Worker can read every MCP body while forwarding it. It does not write those bodies to KV, D1, or R2. Cloudflare's edge can still see metadata, and the path contains the connector secret.
- `ORIGIN_FALLBACK=1` forwards non-Poki paths with `fetch(request)`. Do not turn that on for a shared zone unless you mean to.
- A modified Worker, or anyone with the Cloudflare account, can log or steal whatever passes through. Deploy the file you hashed, and do not point the phone at a Worker you did not deploy.

中继是你自己部署的反向代理。应用里没有默认中继地址。Worker 在内存里能看到完整的 MCP 请求和响应（含截图），但代码不把正文落盘。Cloudflare 账号的持有者如果换掉脚本，就可以记录这些内容。只部署你核对过哈希的 `poki-relay-worker.js`，不要把手机指向别人的 Worker。

---

## Release tag vs claimed build / 版本是否对得上

Release notes say: “Beta 1 of Poki 0.4.0 is Alpha build 1260 (`app-0.4.0-alpha.1260`), unchanged.”

That matches git:

- `app-0.4.0-alpha.1260` annotated tag object `4215eaf9` points at commit `cda35c49` (“Merge pull request #303 … poki-install-path-beta”, 2026-11-13).
- `app-0.4.0-beta.1` points at the same commit.
- The IPA's `CFBundleVersion` is `1260` and `CFBundleShortVersionString` is `0.4.0`.
- The published Worker hash equals `deploy/cf-relay/worker.js` in that commit.

`main` at audit time (`412f612`, 2026-11-14, “choose Poki's language in Settings”) is **78 commits after** the IPA. Those commits are UI language, browser navigation, update copy, and the relay socket check. They are not what is inside `Poki-external-0.4.0-beta.1.ipa`. Installing the IPA installs `cda35c49`, not today's `main`.

`native/ios-app/VERSION` on current `main` is already `0.4.1` (next pre-alpha). That file does not change what the Beta 1 IPA contains.

发布说明写的“Beta 1 就是未改动的 Alpha 1260”与 git 和 IPA 的 `CFBundleVersion` 一致。今天的 `main` 比这个 IPA 新 78 个提交。安装 IPA 得到的是 `cda35c49`，不是今天的 `main`。

---

## AGPL-3.0, briefly / 许可证（简要，不是法律意见）

The repo is AGPL-3.0-only (`LICENSE`).

- Personal use, including self-signing the IPA onto your own phone, does not by itself create a distribution.
- If you **give the IPA or a modified fork to other people**, you must give them the corresponding source under the same license, keep copyright and license notices, and license your modifications under AGPL-3.0 as well.
- If you **run a modified copy as a network service** (a changed relay, or a server that users talk to), AGPL section 13 requires offering that source to those users.
- This fork currently has no modifications, so pointing at `JosssphZhou/poki` commit `412f612` (and at `cda35c49` for the Beta 1 binaries) is the corresponding source for what was audited.
- Do not strip the license. Do not relicense.

自己装到自己手机上，一般不构成再分发。把 IPA 或改过的 fork 给别人，就必须按 AGPL-3.0 提供对应源码并保留声明。改过的中继如果给别人用，第 13 条要求向网络用户提供源码。当前 fork 没有修改。以上不是律师意见。

---

## Practical advice / 实际建议

1. **Install from the upstream release, and check the hash.**  
   `Poki-external-0.4.0-beta.1.ipa`  
   `a361860dcf9fb59de13c4465717f77947a27659aec00c144c93e360983d8fffc`  
   `poki-relay-worker.js`  
   `394450cb399d4b4bce6d9fecad0f9f8cc1aa6fc7b0ddae17247d13507889a532`  
   This fork does not publish a different IPA. If a download page is not `github.com/JosssphZhou/poki/releases/tag/app-0.4.0-beta.1`, stop.

2. **Model key.** Use your own key or ChatGPT sign-in. Everything on screen can be sent there. Prefer a key you can revoke. A custom base URL sends the same data to that host.

3. **Approval.** Leave the default (confirmation, ask for each sensitive step) until you have watched it work. Avoid “Never ask” if the phone is signed into banking, mail, or messengers. Payments and deletions are keyword matches (`pay`, `delete`, `支付`, `删除`, …) plus the Wallet bundle id. A button that does not use those words can pass. The chat model can ALLOW a password field when sensitivity is not “ask everything”.

4. **Relay.** Skip it if you only use Poki's on-phone chat. If you need Claude or ChatGPT on the web to drive the phone, deploy `poki-relay-worker.js` in **your** Cloudflare account, set `TOKEN_SECRET`, leave `ALLOW_SECRET_ONLY` unset, and revoke the connector if the URL leaks. The path contains the secret.

5. **VPN / loopback.** The external IPA needs LocalDevVPN or a proxy that reflects only `10.7.0.1`. Do not install a random “Poki VPN” from outside this repo. Read the module before tapping Surge/Loon install: it should match `deploy/proxy-modules/`.

6. **Apple ID.** A free account works and expires in about 7 days. Poki Setup renews it while the computer is available and you stay signed in. That means the Apple ID password may sit in the local keychain for renewals. Developer signing includes `get-task-allow`. Do not use a primary Apple ID if you can use a separate one. The Beta 1 notes say first-time sign-in on the phone can fail until a later build, because Apple stopped accepting the older Mac identity Poki uses. Signing on a Mac is the path that release expects.

7. **Passcode.** Do not set `POKI_IPHONE_PASSCODE` unless you accept that any process on that Mac can read the environment. The iPhone app does not ask you to upload it.

8. **After install.** Watch the first tasks. Confirm the approval card appears for a send or a password field. Turn off the relay when you are not using an outside agent. Remove the model key if you stop using the app.

中文清单：

1. 只从上游 Release 下载，并核对上面的 SHA-256。这个 fork 没有另做 IPA。
2. 模型密钥用自己的，屏幕内容会发过去。用完可撤销。
3. 审批保持默认。不要对装着银行、邮箱、聊天软件的手机开“不再询问”。付款识别靠关键词，对不上词的按钮可能直接点下去。
4. 只用手机里的对话时，不必部署中继。要部署就部署到自己的 Cloudflare，并核对 worker 哈希。
5. 回环只用 LocalDevVPN 或仓库里那种只反射 `10.7.0.1` 的模块。
6. 免费 Apple ID 大约 7 天要重签。开发签名带调试权限。尽量不要用主 Apple ID。Beta 1 说明里写了：第一次在手机上登录可能失败，需要等下一版或改在 Mac 上登录。
7. 不要把开机密码写进 `POKI_IPHONE_PASSCODE`，除非你接受那台 Mac 上的程序能读到它。
8. 装好后先看审批卡片是否出现，不用外部代理时关掉中继。

---

## Limits of this audit / 本次审计的边界

- First-party source was read and searched. The IPA was opened as a zip. Its main Mach-O was string-scanned, not fully disassembled, and it was not installed or executed.
- `PokiSetup-0.2.2-661-macos.dmg` and `PokiSetup-0.2.2-661-windows-x64-setup.exe` were not downloaded and not run. The desktop behavior above is from `desktop/` source at the same tree, which is what those installers are built from. A swapped installer on a mirror would not be caught here. Check `SHA256SUMS-desktop.txt` on the GitHub release if you install them.
- PiSwift (`f8696524`) and WebRTC (`155.0.0`) are compiled into the IPA. Their provider hostnames showed up in strings and match a multi-provider SDK. Those repositories were not re-audited line by line.
- Absence of a plaintext URL is not a mathematical proof that no encrypted beacon exists inside a third-party blob. Nothing in the first-party code or the IPA string set looked like one.
- Approval keywords and “the model said ALLOW” are real gaps even in an honest build.

桌面安装包没有运行。IPA 做了字符串扫描，没有完整反汇编，也没有装到手机上。PiSwift 和 WebRTC 是编进 IPA 的第三方库。没有在第一方代码或 IPA 字符串里看到隐蔽远控地址。

---

## What to do / 怎么决定

Keep the fork: it is upstream at `412f612`, with no extra code.  
You can self-sign `app-0.4.0-beta.1` for yourself.  
Do it with your own model key, approvals left on, your own relay or no relay, and a free Apple ID you can afford to renew every 7 days.

可以保留这个 fork：它就是上游 `412f612`，没有多出来的代码。  
可以给自己自签安装 `app-0.4.0-beta.1`。  
用你自己的模型密钥，审批保持打开，中继用你自己的或者不用，Apple ID 用一个能接受每 7 天重签的免费账号。
