# Stray: findings

Game: Stray (BlueTwelve Studio, Annapurna), Steam app 1332010, build 12863654. Unreal Engine 4.27
(`++UE4+Release-4.27`), project `Hk_project`. No Denuvo, no anti-cheat. SteamStub (`.bind`
section) **encrypts the code on disk**: the offline call-site scan finds nothing, so the console
layout is only checked at run time (the add-on's layout check refuses to set anything if it
doesn't match). PE timestamp 0x649b65db.

Paths:
- Game: `E:\SteamLibrary\steamapps\common\Stray`
- Executable: `Hk_project\Binaries\Win64\Stray-Win64-Shipping.exe` (`Stray.exe` in the root is the
  launcher)
- User config: `%LOCALAPPDATA%\Hk_project\Saved\Config\WindowsNoEditor\` (`GameUserSettings.ini`,
  section `/Script/Hk_project.HKGameUserSettings`). No engine log (Logs empty).

ReShade 6.8 (`dxgi.dll`, full add-on support) was copied in from Pacific Drive on 2026-11-02; the
game had none. No upscaler of any kind (no DLSS, FSR, XeSS files or cvars; only TAA). DirectX 11
by default (PCGamingWiki); the exe has both D3D11 and D3D12 RHIs.

Menu state (GameUserSettings): sg view distance, post-process, foliage, shading 3 (shadow,
texture, effects and AA groups not written), resolution 100, `r.DefaultFeature.AntiAliasing=0`,
motion blur, DoF and film grain features 0, `r.DepthOfFieldQuality=0`, `r.MaxAnisotropy=16`,
`Gamma=2.425`, HDR off (`HDRDisplayOutputNits=2000`).

Game cvar prefixes (from the unencrypted data): `HK.` (6), `hk.` (5), `Zurg.` (6), `TAA.` (14),
`View.` (4); in the probe. The fur plugin is GFurPRO.

## Base

StrayTweaks is PacificDriveTweaks 1.0.0 (UE 4.27, game-thread setting from a hooked
CallAllConsoleVariableSinks, HDR via output devices 3/4 with a SetColorSpace1 fallback) with the
game's names. UE 4.27 layout expected: FindConsoleVariable +0xb0, Set +0x80, SetBy 0x08/0x09.

## Probe of 2026-11-02 (DirectX 11, menu at its highest)

- Runtime layout check passed: singleton RVA 0x4d349f0 (3 of 3), IsVariableBool +0x30, data
  cvars at +0x48, Find +0xb0 / Set +0x80 as in UE 4.27. Every value taken.
- **Ray tracing is compiled in**: `r.RayTracing=1` and `r.SkinCache.CompileShaders=1` (project),
  `r.RayTracing.Shadows=1`, other RT effects -1 (post-process volume). On DirectX 11 nothing ray
  traces; DirectX 12 (`-dx12`) would let the add-on turn on RT shadows, reflections, AO, GI.
- AA: TAA with TAA upsampling (`r.TemporalAA.Upsampling=1`, `r.DefaultFeature.AntiAliasing=2`),
  screen percentage 100, `r.Tonemapper.Sharpen=1` (code).
- Menu at its highest is sg 2 for shadows, textures, effects, AA (the game's `HK.*.NumLevels=3`):
  shadows 2048 / 4 cascades / `RadiusThreshold=0.04`; texture pool 1000 MB; AO quality 1, SSGI
  off (quality 2), SSR 3 + temporal, DF shadowing on, DFAO off, contact shadows on; fog 8 px / 128
  slices; translucency volume 48; emitter spawn rate 0.5; `r.StaticMeshLODDistanceScale=0.25`
  and `r.ViewDistanceScale=2` (code); anisotropy 8 at connect (16 in the user's settings file).
- Game cvars `HK.*` (quality level counts), `hk.*`, `Zurg.*` (crowd/pool): nothing to tune.

## HDR on DirectX 11

0.1.0 used the PQ output devices (3/4) from Pacific Drive (DirectX 12). On DirectX 11 the engine
created an FP16 scRGB swap chain instead (ReShade: colour space scRGB, back buffer format 10), so
the PQ-encoded output came out grey and washed out (user screenshots 2026-11-02). 0.1.1 picks the
scRGB devices on DirectX 11 (5 = ACES 1000 nits, 6 = ACES 2000 nits) and keeps 3/4 on DirectX 12;
the swap-chain fallback accepts scRGB.

## DirectX 12 and ray tracing (2026-11-02, `-dx12`)

- The game runs on DirectX 12 with `-dx12`; same layout. It ray traces on its own there: the
  default `r.RayTracing.Shadows=1` gave 87-535 DispatchRays per second before any option.
- 0.1.2's RT options (from TrueColorsTweaks) all switched on and off cleanly. User: shadows,
  reflections, AO and GI look good; **ray-traced glass (`r.RayTracing.Translucency`) causes issues**,
  removed in 0.2.0. User chose DirectX 12 as the target.
- HDR on DirectX 12: the engine switched to HDR10 PQ (device 4 from the saved setting), but the
  user finds it **very dark**. UE 4.27 has no HDR brightness controls (`r.HDR.*` is only
  EnableHDROutput, Display.OutputDevice, Display.ColorGamut, UI.CompositeMode, UI.Level; no
  MaxLuminance, MidLuminance or Aces.SceneColorMultiplier), and `r.Gamma`/TonemapperGamma only
  touch SDR. Proposed: RenoDX's generic Unreal Engine add-on for HDR.

0.2.0 options: shadows 4096 + 6 cascades / 8192 sun + 8 cascades, shadow distance + small-object
shadows, SSGI 3/4, AO quality 2/3, SSR 4, fog 6/4 px, full particles + translucency volume 64,
light distance x2, texture pool 3/4/6 GB, anisotropy 16, mip bias, detail mode 2, post (CA, lens
flares, bloom, sharpening), RT shadows 1/2 rays, RT reflections On/High, RT AO 1/2 rays, RT GI
final gather / brute force, HDR. 42 cvars checked present. Presets Ultra+ and Cinematic (with RT),
unmeasured.

User chose RenoDX for HDR: 0.2.1 has no HDR option (the API-dependent device code stays but is
unused).

## Costs (0.2.1, 2026-11-02, DirectX 12, RenoDX `renodx-ue-extended` for HDR)

Presets: Game 15.88 ms (63 fps, with the game's own RT shadows), Ultra+ 16.68 ms (+5%), Cinematic
37.28 ms (+135%). Each option alone (Game 15.53 ms): RT GI brute force +120.2%, SSGI Cinematic
+58.3%, RT reflections High +47.7%, RT GI final gather +44.5%, SSGI High +34.9%, fog 4 px +13.5%,
RT AO 2 rays +7.6%, RT shadows 2 rays +2.8%, fog 6 px +2.8%; everything else within +-3% (RT
reflections On -1.6%, RT shadows 1 ray -3.0%: noise).

0.2.2 (user's choices): Ultra+ adds RT reflections On; Cinematic drops SSGI and keeps RT reflections
High, RT AO 2 rays, fog 4 px (expected about +70%).

## DirectX 12 performance (2026-11-02)

User: about 65 fps at Game on DirectX 12 against a 166 fps cap on DirectX 11. On DirectX 12 the
game ray traces shadows by itself (`r.RayTracing.Shadows=1`) and the log shows about 4900
acceleration structure builds a second at Game. 0.2.3 adds "All ray tracing: Off"
(`r.RayTracing.ForceAllRayTracingEffects=0`), "Particles in ray tracing: Off"
(`r.RayTracing.Geometry.NiagaraSprites/Ribbons/Meshes=0`) and an Off choice for RT shadows, to see
how much of the gap is ray tracing and how much is DirectX 12 itself.

Measured (0.2.3, each option, DirectX 12, Game 16.17 ms / 62 fps): all RT off -60.4% (6.4 ms, about
156 fps: DirectX 12 itself is close to DirectX 11), RT shadows off -17.5%, particles out of RT
-0.1%. So about 40% is ray-traced effects the levels' post-process volumes switch on (the
reflection/AO/GI/translucency/sky light cvars read -1). Other costs as before (RT GI BF +119%, SSGI
4 +61%, RT reflections High +50%, RT GI FG +45%, SSGI 3 +38%, fog 4 px +13%, RT AO 2 rays +9%).
0.2.4 adds "Shadows only" to the master switch (reflections/AO/GI/translucency/sky light forced
to 0, shadows kept) to find which.

## To find out in game (0.2.4)

- Cost of "Shadows only", and which level effect costs the rest.

- DirectX 12 with all ray tracing off vs DirectX 11; cost of the game's RT shadows and particles.
- Both presets measured as a whole.

## 0.2.4 measurement and 0.2.5 presets (2026-11-02)

Each option alone, DirectX 12, RTX 5080 4K, Game 17.24 ms (58 fps): RT GI brute force +110%,
SSGI 4 +55%, RT reflections High +46%, RT GI final gather +42%, SSGI 3 +33%, RT AO 2 rays +9.4%,
RT shadows 2 rays +6.4%, fog 4 px +5.2%; everything else within +-2%. Ray tracing master:
Shadows only -39% (95 fps), Off -63% (156 fps); RT shadows off -19%. So the levels' own RT
effects other than shadows cost about 6.7 ms. User: final gather is far noisier than brute
force here, so brute force is listed first and final gather is marked noisy.

0.2.5: Ultra+ raised to land between Game and Cinematic (user): RT reflections Medium (new: 0.8
roughness, 1 bounce, 1 shadow ray), RT AO 2 rays, fog 4 px (expected +25-35%). Cinematic:
RT reflections High, AO 3, 8192 shadows (expected about +70%). Frame rate work is left for the
DLSS/FSR/frame generation add-on.

Presets measured on 0.2.5 (2026-11-02, same scene): Game 15.47 ms (65 fps), Ultra+ 21.67 ms
(+40%, 46 fps), Cinematic 27.66 ms (+79%, 36 fps). Ultra+ sits at the midpoint as asked.
0.2.6 only updates the preset labels and tips.
