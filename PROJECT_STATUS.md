# PROJECT_STATUS.md — DIY iPhone 13 Pro VR / MR headset (CardboardXR fork)

_Last updated: 2026-11-27 — Phase 1: v0.1.0 failed to install (pre-signed UnityRuntime.framework); fixed in CI; **v0.1.1 built green** (run 45960296063). Next: user sideloads v0.1.1 and runs the test checklist._

## 1. Baseline

| Item | Value |
|---|---|
| Upstream | https://github.com/brenpoly/CardboardXR (remote name: `upstream`) |
| Baseline commit | `2cb46a2` (2026-11-22), tagged `upstream-baseline-2cb46a2` |
| Our work branch | `dev` (upstream `main` left untouched); head `8b927b4` (v0.1.1 build) |
| Rollback point | tag `v0.1.0` + branch `rollback/v0.1.0` → `f557d6f`; v0.1.0 release/IPA untouched |
| Our fork | public `doctoravi8r/CardboardXR` (remote `origin`); default branch still `main` (switch to `dev` pending) |
| Bundle ID | `com.malekalali.cardboardxr` (CI default; repo variable `IOS_BUNDLE_ID` overrides) |
| Local path | `C:\Dev\CardboardXR` |
| License | Apache-2.0; bundled models/fonts/ported code have their own licenses (see THIRD_PARTY_NOTICES.md) |
| Unity | **6000.6.8f1** (Unity 6.6), URP 17.6, IL2CPP |
| Key packages | AR Foundation / ARKit / ARCore 6.7.1, XR Hands 1.9.0, XR Interaction Toolkit 3.6.1, Unity Inference Engine (`com.unity.ai.inference`) 2.6.1, Input System 1.20.1 |
| Code size | ~19,300 lines C# across 6 assemblies + 6 demos, with EditMode tests |
| iOS target | iOS 14.0+, iPhone only, ARKit **required**, automatic signing, ProMotion enabled |

## 2. Architecture (preserve)

- One generated **Core** scene (rig, head tracking, AR, hands, gaze, settings) + additively loaded **Lobby/Demo** scenes switched by `DemoManager`; demos reach Core via `CoreServices`.
- `Core.unity` is **regenerated on every build** by `Assets/Editor/ProjectBootstrap/StereoBuilder.cs` — never hand-edit it; change the builder.
- Lobby/demo scenes are hand-edited; each demo is self-contained in `Assets/_Project/Demos/<Name>/` and registered in `DemoCatalog.asset` (see ADDING_A_DEMO.md, `Tools/new-demo`).
- Does **not** use Google Cardboard SDK: custom stereo rig + lens distortion ported from it, so AR Foundation can be the single XR loader.
- Assemblies: Stereo, Hands, Interaction, UI (Material 3 kit), Demos, Tracking (Unity-free sensor fusion).

## 3. Feature gap analysis

| Desired feature | Status | Where / notes |
|---|---|---|
| Stereo 3D video | ✅ for spherical (360/180, SBS/TB) | `Demos/Video` — per-eye layers. ❌ flat SBS/TB 3D movies on a screen |
| VR180 & 360 video | ✅ | Format from filename tags (`_360 _180 _sbs _tb _mono`), Documents/Videos, Google Drive links |
| Head tracking | ✅ 6DoF | ARKit pose fused with gyro EKF (`HeadPoseFusion`, `Tracking/*`), 30 ms prediction, 3DoF mode |
| Camera passthrough | ✅ (mono/flat) | `PassthroughController`; one camera ⇒ no stereo depth (inherent) |
| Spatially anchored screens | 🟡 foundation only | `AnchorPlacer` + `ARAnchorManager` anchor cubes; no screen/window object yet |
| Hand tracking & pinch | ✅ | MediaPipe BlazeHand ONNX on Inference Engine; **LiDAR depth calibrates hand size on Pro iPhones**; `PinchRouter` |
| Virtual cinema | ❌ | New demo; reuse VideoLibrary/VideoFormat + anchors |
| GPT voice + vision assistant | ❌ | New; needs mic permission, network, key handling, privacy note change |
| Lens/viewer calibration | ✅ | Cardboard QR viewer profiles (`ViewerQrScanner`) |
| Extra demos | ✅ | Lobby, Cube Sandbox, Air Paint, Orb Rush, Portal, 360 Video |

## 4. iPhone 13 Pro compatibility (verified vs. assumed)

- ✅ Verified: ARKit + LiDAR (scene depth API) present; ProMotion 120 Hz; runs iOS 27 (same device list as iOS 26); above the iOS 14 minimum. User's phone is on iOS 26.3.1.
- ✅ Verified in code: screen size comes from Unity's `Screen.dpi` on iOS (no hard-coded device table), so 13 Pro geometry should be correct — **confirm** from the `[Stereo]` line in session.log.
- ⚠️ Expected risk: tested only on iPhone 15 Pro (A17 Pro). A15 is slower: hand models already cost 7–31 ms/frame on 15 Pro ⇒ expect lower FPS and earlier thermal throttling on 13 Pro. Mitigations exist (`fps`, `handHz`, `renderScale`) but are launch options — see Issue #1.
- ⚠️ Unverified: high-bitrate 8K 360 video decode on A15 — benchmark in Phase 2.

## 5. Bugs, limitations, risks

1. **Launch options are unreachable from Windows** — on iOS they come only from environment variables set by the Mac script. A home-screen launch always uses defaults (120 fps, full hand rate). Proposed fix: also read `Documents/launch_options.txt`.
2. **All build/run scripts are macOS-only** (zsh, xcodebuild, devicectl).
3. **No Editor/Standalone XR loader** configured ⇒ AR, passthrough, hands can't run in Play Mode on Windows. Option: enable Unity XR Simulation for editor tests.
4. **Bundle ID** `com.brenpoly.cardboardxr` belongs to the author's team; fixed in our CI (sets our own ID before export).
5. ✅ CI verified 2026-11-27: game-ci image for 6000.6.8f1 iOS exists, Personal license activates, macos-15 Xcode archives it. Unity export ~12 min, Xcode ~10 min.
6. Free Apple ID signing: app expires every 7 days; max 3 sideloaded apps; 10 App IDs / 7 days.
7. Upstream known issues (KNOWN_ISSUES.md): narrow camera FOV for hands (±21° vertical), hands <15 cm unreliable, flat passthrough, stray far detections, missed grabs place cubes, 120 fps not held with hands/heat.
8. Thermal: phone enclosed in a headset, camera + NN + 120 Hz ⇒ throttling within minutes is likely.
9. **Fork workflow registration:** workflow is registered and active (id 432411961). `workflow_dispatch` on `dev` works (used for v0.1.1, with input `version`). Tag pushes still build + create a release.
10. Built IPA's MinimumOSVersion is **15.0** (Unity 6 floor), not 14.0 — irrelevant for the 13 Pro.
11. ✅ **Fixed in v0.1.1 — v0.1.0 install failure.** Sideloadly (v0.72) → iOS 26.3.1: `ApplicationVerificationFailed` 3892346881 / 0xe8008001 on `Frameworks/UnityRuntime.framework`. Cause: Unity 6.6 embeds a prebuilt stub `UnityRuntime.framework` (not in Unity 6.0/6.3; referenced by nothing — no load command or string in the app or UnityFramework). Unity ships it as a fat arm64 Mach-O with a linker ad-hoc signature, plus a detached signature in `_CodeSignature/` made with Unity's macOS **Developer ID Application** cert (team 9QW8UQUTAA), and a build-machine temp path as its install name. Xcode would normally overwrite this when signing, but our archive runs with signing off, so it reached the IPA untouched while the app binary and UnityFramework were unsigned (Sideloadly signed those fine). Fix: CI step "Sanitize embedded frameworks" (remove `_CodeSignature`, thin to arm64, reset install name to `@rpath/…`, strip signature, add `CFBundleSupportedPlatforms`/`CFBundleInfoDictionaryVersion`) + "Verify app bundle" (fails if any Mach-O is still signed). Applies to any future pre-signed plugin framework too. Fallback if ever needed: delete the stub entirely (unused).
12. CFBundleVersion was `0`; CI now sets it to the version string (0.1.1).
13. Claude sessions can push branches to the fork but **tag pushes are refused (HTTP 403)**; Claude builds via `workflow_dispatch` instead. Release-attached IPAs need a tag pushed from the user's machine.
14. Sideloadly is not the issue: its FAQ claims support through iOS 26+, and the v0.1.0 failure was the phone's installer rejecting the IPA contents after Sideloadly had signed and transferred it.

## 6. Build & install strategy (no Mac)

**Recommended (free):**
1. Develop & run EditMode tests on Windows in Unity 6000.6.8f1 (+ iOS Build Support module). Unity on Windows can export the iOS Xcode project.
2. Push to our own GitHub fork → `.github/workflows/ios-ipa.yml` builds on GitHub's Linux (Unity export) + macOS (xcodebuild, sanitize, verify) runners → unsigned `.ipa` artifact.
3. Sign & install from Windows with **Sideloadly** (needs Apple's iTunes for Windows from apple.com, not the Microsoft Store build) using a free Apple ID; enable Developer Mode and trust the profile on the iPhone. Optional Wi-Fi auto-refresh handles the 7-day expiry.
4. Logs: `session.log` is in the app's Documents folder, exposed to file sharing (iTunes/Apple Devices on Windows).

**Paid fallbacks (need approval):** Apple Developer Program ($99/yr: 1-year signing, TestFlight); cloud Mac rental. Not needed to start.

## 7. Roadmap

| Phase | Goal | Exit criteria |
|---|---|---|
| **0** ✅ | Assessment, git, this file | Approved plan |
| **1** | Original app on iPhone 13 Pro, unmodified except bundle ID | Fork + CI green; IPA sideloaded; lobby, 6DoF, passthrough, hands, 360 demo work; session.log baseline captured |
| **1b** | Windows-friendly tooling (small) | `launch_options.txt` support; PowerShell helper; optional XR Simulation in editor |
| **2** | 13 Pro performance & comfort | Measured FPS/thermal per demo; tuned defaults (e.g., 60–90 fps, handHz); headset lens/QR profile calibrated |
| **3** | Spatial screens | Anchored, resizable, pinch-movable flat screen (2D + SBS/TB 3D) in passthrough |
| **4** | Virtual cinema | New demo: dark theater, curved screen, video library, playback controls |
| **5** | Media sources | Local files + Drive (exists) → optional DLNA/HTTP stream from the RTX 4080 PC |
| **6** | AI assistant | Voice (push-to-pinch) → OpenAI Realtime/Responses API; camera-frame vision queries; key via user entry/Keychain or small proxy; privacy notice |

## 8. Phase 1 status

### Done
- Windows inventory (2026-11-27): present: Docker, Node.js, .NET, Android SDK, Steam/SteamVR.
- User installed (2026-11-27): iTunes, iCloud, Sideloadly, Unity Hub, Unity 6000.6.8f1 + iOS Build Support; Unity Personal license activated. Git/gh not confirmed — `Phase1-Install.ps1` re-run installs them and logs every check to `C:\Dev\logs\phase1-install.log`.
- CI: `ios-ipa.yml` sets our own bundle id before the Unity export (env vars don't reach game-ci's container, so ProjectSettings is edited in the throwaway checkout, like the version step).
- **Workflow security review (2026-11-27):**
  - Triggers are only version-tag push and manual dispatch — no `pull_request`/`pull_request_target`, so strangers' PRs to the public fork never see secrets.
  - All 5 third-party actions pinned to full SHAs; each SHA verified against the official tag (checkout v4.4.0, unity-builder v4.8.2, upload-artifact v4.6.2, download-artifact v4.3.0, action-gh-release v2.6.2).
  - No `${{ }}` interpolation of user-controlled input in `run:` steps; tag name constrained by pattern; the manual `version` input goes through `env` and is regex-validated; bundle id from a repo variable only the owner can set.
  - Secrets (Unity license/email/password) go only to game-ci's step; GitHub masks them in logs.
  - Hardening added (commit `1018b42`): workflow-level `permissions: contents: read` (write only on the release job), `persist-credentials: false`, `fetch-depth: 0`, job timeouts (90/60 min), per-ref concurrency, 3-day retention for the intermediate Xcode-project artifact.
  - Project script now also sets the fork's default workflow token to read-only, pushes only our baseline tag, and pauses its log while the Unity email/password are typed.
  - Accepted: `gh` login needs the broad `repo` + `workflow` OAuth scopes (GitHub has no narrower option for pushing workflows); token is kept in Windows Credential Manager. Artifacts on a public repo are downloadable by any signed-in GitHub user — they contain only the open-source app.
- **Free signing verified sufficient:** the app requests no entitlements (camera string, network, file sharing only).
- Delivered to `C:\Dev`: `CardboardXR.bundle` (rebuilt with `dev` @ `1018b42`, `main`, tags), `Phase1-Install.ps1`, `Phase1-Project.ps1` (both parse-checked with PowerShell 7.4).

### First build v0.1.0 (2026-11-27) — rollback point, does not install
- Run https://github.com/doctoravi8r/CardboardXR/actions/runs/45950502676 — tag `v0.1.0` → `f557d6f`; both jobs green.
- Release asset: https://github.com/doctoravi8r/CardboardXR/releases/download/v0.1.0/CardboardXR-v0.1.0.ipa (46.7 MB, sha256 `f53044ef5e8fd2089b19a61d3a5b47fbec2e3908abfec90257effadb3fa3e1e7`).
- Sideloadly install failed (issue #11). Kept unchanged as the rollback point (tag + branch `rollback/v0.1.0`).

### Build v0.1.1 (2026-11-27) ✅ built, awaiting sideload test
- Commits on `dev`: `06aa8bb` (sanitize + verify + build number), `5adbc98` (versioned manual dispatch), `8b927b4` (install name before signature strip — first attempt failed because `install_name_tool` rejects a just-stripped binary).
- Run https://github.com/doctoravi8r/CardboardXR/actions/runs/45960296063 (workflow_dispatch on `dev`, version 0.1.1) — both jobs green.
- Verify step (Apple codesign/lipo/otool on the runner): CardboardXR, UnityFramework, UnityRuntime all thin arm64 and unsigned; no `_CodeSignature` anywhere; UnityRuntime install name now `@rpath/UnityRuntime.framework/UnityRuntime`.
- Artifact `CardboardXR-ipa` → `CardboardXR-v0.1.1.ipa`, sha256 `57b17cc37bcc813cbc97968468305d1e63adbf0ffc88c4ac7e9ce318aeab1810`. No release (no tag yet).
- To get a release-attached copy: on Windows in `C:\Dev\CardboardXR`: `git fetch origin` → `git tag v0.1.1 origin/dev` → `git push origin v0.1.1` (rebuilds; ~25 min; different hash).

### Sideloading on Windows (free Apple ID)
1. Get `CardboardXR-v0.1.1.ipa`: Actions run 45960296063 → Artifacts → `CardboardXR-ipa` (zip; sign in to GitHub) → extract the `.ipa`.
2. iPhone on USB, unlocked → "Trust This Computer". iTunes should show the phone.
3. Sideloadly → drop the `.ipa` → pick the iPhone → enter Apple ID (typed by user into Sideloadly) → Start.
4. iPhone: Settings > General > VPN & Device Management → trust your Apple ID. Settings > Privacy & Security > Developer Mode → On → restart → confirm.
5. Open CardboardXR → allow camera. Re-sign within 7 days (Sideloadly can auto-refresh over Wi-Fi).
6. Session log: iTunes → phone → File Sharing → CardboardXR → save `session.log`.
7. If install still fails: copy the full Sideloadly log (the framework path + error code matter).

### Test checklist (priority order)
| # | Area | How | Pass criteria | Result |
|---|---|---|---|---|
| 1 | Stereoscopic display | Launch, insert in headset; gear → Set viewer (scan headset QR if it has one) | Two undistorted-looking eye views fuse into one; `[Stereo]` log screen ≈ 140 × 65 mm | |
| 2 | Head tracking | Lobby: turn, tilt, step; settings → toggle 6DoF; hold tap = recenter | No visible lag/drift; positional movement in 6DoF; `[Pose]` lines healthy | |
| 3 | VR180 / 360 video | App library → 360 Video → both sample clips; then own clip in Documents/Videos via iTunes file sharing | Correct projection; 3D clip has depth; smooth playback | |
| 4 | Passthrough | Settings → Passthrough | Live camera, correct orientation, acceptable latency | |
| 5 | Hands & anchors | Cube Sandbox: place cube by gaze+pinch, grab/throw; palm-up gear | Pinch reliable; cubes stay anchored when walking around | |

Record FPS (`[Frames]`) and thermal (`[Thermal]`) for each.

## 9. Decisions log

- 2026-11-27: Keep upstream architecture; new features as demos/Core builder additions only.
- 2026-11-27: Use existing GitHub Actions IPA workflow + Sideloadly (free) rather than paid cloud Mac.
- 2026-11-27: `upstream` remote, `dev` branch, baseline tag `upstream-baseline-2cb46a2`.
- 2026-11-27: Public fork (free CI minutes). Bundle id `com.malekalali.cardboardxr`.
- 2026-11-27: Repo delivered as a git bundle (git can't run safely in the shared folder from Claude's side); user's Git clones it.
- 2026-11-27: Builds triggered by `vX.Y.Z` tags on `dev`, or by manual dispatch with a `version` input.
- 2026-11-27: Workflow hardened (read-only token default, no persisted credentials, timeouts); don't push upstream `v*` tags to the fork.
- 2026-11-27: Embedded frameworks are handed to Sideloadly unsigned (same state as the app binary), not ad-hoc signed — matches what already worked for UnityFramework. UnityRuntime stub kept (sanitized) rather than deleted, to stay close to Unity's output.
- 2026-11-27: v0.1.0 preserved as rollback (tag + `rollback/v0.1.0`).

## 10. Remaining Phase 1 / next
- [x] Software installed (user-reported; confirm via phase1-install.log)
- [x] Fork + secrets + first CI build green (v0.1.0)
- [x] Install failure diagnosed and fixed in CI; v0.1.1 built green
- [ ] Sideload v0.1.1 and confirm it launches
- [ ] Optional: push tag `v0.1.1` from Windows for a release-attached IPA
- [ ] Switch fork default branch to `dev` (optional)
- [ ] Checklist items 1–5 pass; baseline session.log saved
- [ ] Then (approved): Phase 1b `Documents/launch_options.txt` for fps/handHz/renderScale
