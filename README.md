# StrayTweaks 1.0.0

Graphics add-on for **Stray** (PC, Steam), for ReShade with add-on support. It raises the
game's lighting, shadow, ray tracing and detail settings beyond what its menu offers, live,
from an in-game overlay.

- **Lighting**: sharper shadows (4096, or 8192 for the sun) with more cascades, shadows drawn
  further and from small objects, screen-space bounce light (the game leaves it off),
  higher-quality ambient occlusion, Cinematic screen-space reflections, finer volumetric fog,
  full particle effects, lights drawn twice as far.
- **Ray tracing (DirectX 12)**: ray-traced shadows with 1 or 2 rays, ray-traced reflections
  (On, Medium, High), ray-traced ambient occlusion, ray-traced bounce light (brute force, or
  the cheaper but noisy final gather). **Ray tracing from the game** turns the ray tracing the
  game switches on by itself down to shadows only, or off.
- **Detail**: texture memory beyond the game's 1 GB, 16x anisotropic filtering, texture
  sharpness, detail meshes.
- **Post-processing**: switches for chromatic aberration, lens flares and bloom, and lighter or
  no sharpening.
- **Quality presets**: *Ultra+* (about 40% slower) and *Cinematic* (about 80% slower),
  measured on an RTX 5080 at 4K on DirectX 12. They leave post-processing alone.
  **Measure preset costs** and **Measure each option** check them on your PC.
- **F8** switches every tweak off and on, to compare.

Every option starts at the game's own value. HDR isn't part of this add-on; the RenoDX
Unreal Engine add-on (`renodx-ue-extended`) works alongside it.

## DirectX 12 and performance

Ray tracing needs DirectX 12: add `-dx12` to the game's Steam launch options. On DirectX 12 the
game turns on ray-traced shadows and, through its levels, more ray-traced effects by itself,
which makes it much slower than DirectX 11 (65 against about 156 fps here). If you want the
speed back, set **Ray tracing from the game** to *Shadows only* (about 95 fps) or *Off*
(about DirectX 11's frame rate).

## Requirements

- Stray for PC (Steam build 12863654 tested, Unreal Engine 4.27). If the game's code doesn't
  match, the add-on changes nothing and says so in the overlay and in `StrayTweaks.log`.
- [ReShade](https://reshade.me) 6.8 or newer **with full add-on support**, installed for
  `Stray-Win64-Shipping.exe` with DirectX 10/11/12.

## Install

1. Install ReShade with add-on support into `...\Stray\Hk_project\Binaries\Win64`.
2. Copy `StrayTweaks.addon64` into the same folder.
3. Start the game and press **Home** to open ReShade. The settings are on the **Add-ons** tab,
   under StrayTweaks.

Settings are saved in `ReShade.ini` under `[StrayTweaks]`. Nothing else is written to disk;
deleting the `.addon64` file removes the mod completely.

## Building

`.\build.ps1` builds `bin\StrayTweaks.addon64` (needs Visual Studio 2022 with the C++ tools).
`.\build.ps1 -Install` also copies it into the game folder (close the game first), `-Package`
makes the release zip. See `docs/FINDINGS.md` for how the game's settings were found.

## License

MIT (see `LICENSE`). Third-party headers: see `THIRD_PARTY_NOTICES.txt`.
