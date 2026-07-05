# ARRI-Houdini

Houdini-ready OCIO config based on ARRI's studio config, using ARRI Reveal as the display transform.

## Status
Stable and ready to use. Built to avoid the usual Houdini OCIO friction so you don't have to. Works with external renderers, tested Arnold in Solaris (Houdini 21). Loaded this config in Nuke 17 and it transferred properly.

## Quick Start
Create or edit `ocio.json` file in your Houdini `packages` folder and replace the path with your local path of the config:

```json
{
  "enable": true,
  "show": true,
  "load_package_once": true,
  "env": [
    { "OCIO": "${OCIO-/path/to/ARRI-Houdini/arri-CG.ocio}" }
  ]
}
```

Restart Houdini after adding or changing the `OCIO` setting or package file, since the config is read at startup.

With this config active, `ACEScg` is the intended default working space for CG content. Select any non-default display from the viewport Color Correction toolbar and you'll be using the `ARRI ALF2 / Reveal` transform.

Houdini’s OCIO editor is unreliable in practice for config authoring/persistence (this isn’t specific to this config). If you want changes, edit the .ocio file directly and restart Houdini.

Solaris Render Gallery has its own caching/consistency quirks; don’t use it as your only ground truth when validating color.

## What This Changes
The original ARRI studio config is valid, but its defaults are camera-log oriented and its default file-rule behavior is not a good fit for a CG-first Houdini workflow.

`./arri-CG.ocio` keeps the same transform inventory and LUT payloads, but switches the defaults to a CG-friendly setup so texture loading and working-space behavior are predictable in Houdini.

| Role | Original config | CG config |
| --- | --- | --- |
| `default` | `ARRI LogC4` | `ACEScg` |
| `scene_linear` | `Linear ARRI Wide Gamut 4` | `ACEScg` |
| `reference` | not set | `ACEScg` |
| `compositing_log` | `ARRI LogC4` | `ACEScct` |
| `color_timing` | `ARRI LogC4` | `ACEScct` |
| `matte_paint` | `ARRI LogC4` | `ACEScct` |
| `texture_paint` | `Gamma 2.4 Rec.709 - Texture` | `sRGB - Texture` |

File rules in `./arri-CG.ocio`:

- `*srgb_tx*` -> `sRGB - Texture`
- `*srgb_texture*` -> `sRGB - Texture`
- `*ACEScg*` -> `ACEScg`
- `__usdz_jpg` -> `sRGB - Texture`
- `__usdz_jpeg` -> `sRGB - Texture`
- `__usdz_png` -> `sRGB - Texture`
- `__usdz_tif` -> `sRGB - Texture`
- `__usdz_tiff` -> `sRGB - Texture`
- `__usdz_exr` -> `ACEScg`
- `.jpg` -> `sRGB - Texture`
- `.jpeg` -> `sRGB - Texture`
- `.png` -> `sRGB - Texture`
- `.tif` -> `sRGB - Texture`
- `.tiff` -> `sRGB - Texture`
- `.exr` -> `ACEScg`
- fallback `Default` -> `ACEScg`
Tag rules take precedence over extension rules.
USDZ archive-internal file paths are matched by explicit regex rules before the normal extension rules.

## Troubleshooting
### File rules are a safety net (not a replacement for correct MaterialX assignment)
The file rules in this config are a **pipeline fallback** for loosely named assets. OCIO file rules are evaluated **top-down**, and the first match wins.  
See [OCIO file rules documentation](https://opencolorio.readthedocs.io/en/latest/guides/authoring/authoring.html#file-rules)

Houdini/USD can surface files inside a USDZ archive as paths that look like `.usdz[texture.ext]`. Those do not behave like plain top-level filenames, which is why this config adds explicit regex rules for USDZ payloads instead of relying only on the normal extension rules. See the [SideFX USD Zip render node](https://www.sidefx.com/docs/houdini/nodes/out/usdzip.html).

Do **not** treat extension rules (e.g. `.png`/`.tif` → sRGB) as a license to stop assigning the correct interpretation on MaterialX nodes:
- **Color textures** (albedo/baseColor/emission) are typically sRGB.
- **Data textures** (normal, roughness, metallic, AO, displacement, masks/IDs) must be **Raw/Data** (no color transform).

SideFX explicitly notes that when reading **normal maps** with MaterialX, you should set the **Signature to Vector3** to make sure a color space is not applied:  
[SideFX MaterialX documentation](https://www.sidefx.com/docs/houdini/solaris/materialx.html)

### Wrong-Looking Textures
If a texture looks washed out, oversaturated, or double-transformed, check whether it is already display-referred, whether the filename includes `srgb_tx`, `srgb_texture`, or `ACEScg`, and whether the extension matches the intended automatic rule.

- `albedo.png` -> `sRGB - Texture` (extension rule)
- `albedo_srgb_tx.png` -> `sRGB - Texture` (tag rule, takes precedence)
- `lighting.exr` -> `ACEScg` (extension rule)
- `lighting_ACEScg.exr` -> `ACEScg` (tag rule)

If you need to inspect image files directly, [OpenImageIO documentation](https://openimageio.readthedocs.io/en/stable/) is a good reference, and tools like `iinfo` and `oiiotool` are useful for checking metadata, channels, and file properties.

## Validation
`ociocheck` passes for both configs. The original config still has the extra default-rule/default-role warning, and `arri-CG.ocio` removes that specific warning.

## Official ARRI Resources
For official downloads and ARRI-provided color-management materials, start here:

- [ARRI Technical Downloads](https://www.arri.com/en/learn-help/learn-help-camera-system/technical-downloads)
- [ARRI Color Management Assets](https://arri.canto.de/v/ARRIColorManagement/landing?viewIndex=0)

## Resources for the curious ones
For deeper reading and viewing on color science and pipeline decisions:

- [Liam Collod's Picture Lab](https://liamcollod.xyz/picture-lab-lxm/lxmpicturelab.al.sorted-color.bg-black)
- [Chris Brejon's articles](https://chrisbrejon.com/articles/)
- [ARRI Tech Talk: DP's analysis of REVEAL Color Science (video)](https://youtu.be/s_RXjVeC_7s?si=0oX8oWE6Vnm1CpH3)

## Related Projects
[ARRI-Houdini](https://github.com/rolledhand/ACEScg-v2.0-Houdini) ACES 2.0 cg variant. Tweaked for Houdini by me.

This repo focuses on Houdini-friendly defaults, file rules, and predictable CG working space behavior instead. `ACEScg` as working space is a pipeline sanity choice. Open to feedback and solutions.

## Tweaking tips for the brave ones
ARRI Reveal runs on ACES 1.3 which has its own quirks, but the same rule still applies here: do not trust a pipeline until you validate the whole chain.

I recommend testing transfers between input texture encoding to scene-linear (Solaris/MaterialX), scene-linear to Hydra delegate/viewport, Arnold maketx/.tx generation to MaterialX reads (.tx + colorspace), husk/kick EXR to comp/viewers, scene-linear to display transform, and display transform to final deliverables, then lock it in your comp/grading software of choice.

The testing requirements can differ per render engine/delegate, and with the current state of documentation + inconsistent naming and some tech-debt here & there, it's honestly hit or miss unless you validate the whole chain end-to-end. Terminal with `iinfo`, `oiiotool` & `ociocheck` become your best friend.
