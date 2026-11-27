# PROJECT_STATUS.md — DIY iPhone 13 Pro VR / MR headset (CardboardXR fork)

_Last updated: 2026-11-27 — Phase 0 (assessment) complete; awaiting approval to start Phase 1._

## 1. Baseline

| Item | Value |
|---|---|
| Upstream | https://github.com/brenpoly/CardboardXR (remote name: `upstream`) |
| Baseline commit | `2cb46a2` (2026-11-22), tagged `upstream-baseline-2cb46a2` |
| Our work branch | `dev` (upstream `main` left untouched) |
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

- ✅ Verified: ARKit + LiDAR (scene depth API) present; ProMotion 120 Hz; runs iOS 27 (same device list as iOS 26); above the iOS 14 minimum.
- ✅ Verified in code: screen size comes from Unity's `Screen.dpi` on iOS (no hard-coded device table), so 13 Pro geometry should be correct — **confirm** from the `[Stereo]` line in session.log.
- ⚠️ Expected risk: tested only on iPhone 15 Pro (A17 Pro). A15 is slower: hand models already cost 7–31 ms/frame on 15 Pro ⇒ expect lower FPS and earlier thermal throttling on 13 Pro. Mitigations exist (`fps`, `handHz`, `renderScale`) but are launch options — see Issue #1.
- ⚠️ Unverified: high-bitrate 8K 360 video decode on A15 — benchmark in Phase 2.

## 5. Bugs, limitations, risks

1. **Launch options are unreachable from Windows** — on iOS they come only from environment variables set by the Mac script. A home-screen launch always uses defaults (120 fps, full hand rate). Proposed fix: also read `Documents/launch_options.txt`.
2. **All build/run scripts are macOS-only** (zsh, xcodebuild, devicectl).
3. **No Editor/Standalone XR loader** configured ⇒ AR, passthrough, hands can't run in Play Mode on Windows. Option: enable Unity XR Simulation for editor tests.
4. **Bundle ID** `com.brenpoly.cardboardxr` belongs to the author's team; the CI workflow doesn't override it. Must build/sign with our own ID.
5. CI uses `game-ci/unity-builder` with Unity 6000.6.8f1 — Docker image availability for this exact version and the macOS runner's Xcode version are **unverified until the first run**.
6. Free Apple ID signing: app expires every 7 days; max 3 sideloaded apps; 10 App IDs / 7 days.
7. Upstream known issues (KNOWN_ISSUES.md): narrow camera FOV for hands (±21° vertical), hands <15 cm unreliable, flat passthrough, stray far detections, missed grabs place cubes, 120 fps not held with hands/heat.
8. Thermal: phone enclosed in a headset, camera + NN + 120 Hz ⇒ throttling within minutes is likely.

## 6. Build & install strategy (no Mac)

**Recommended (free):**
1. Develop & run EditMode tests on Windows in Unity 6000.6.8f1 (+ iOS Build Support module). Unity on Windows can export the iOS Xcode project.
2. Push to our own GitHub fork → existing `.github/workflows/ios-ipa.yml` builds on GitHub's Linux (Unity export) + macOS (xcodebuild) runners → unsigned `.ipa` artifact.
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

## 8. Decisions log

- 2026-11-27: Keep upstream architecture; new features as demos/Core builder additions only.
- 2026-11-27: Use existing GitHub Actions IPA workflow + Sideloadly (free) rather than paid cloud Mac.
- 2026-11-27: `upstream` remote, `dev` branch, baseline tag `upstream-baseline-2cb46a2`.

## 9. Next actions (owner)

- [ ] User: approve plan; choose public vs private fork (public = free CI minutes; private = macOS minutes billed against quota).
- [ ] User: install Unity Hub + 6000.6.8f1 with iOS Build Support; Git for Windows; iTunes (apple.com) + Sideloadly.
- [ ] User: create Unity account secrets for CI (UNITY_LICENSE, UNITY_EMAIL, UNITY_PASSWORD).
- [ ] Claude: add bundle-ID override to CI workflow (tiny change) once fork exists.
- [ ] Claude: Phase 1b `launch_options.txt` (after approval).
