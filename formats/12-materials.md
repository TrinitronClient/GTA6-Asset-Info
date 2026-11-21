# 12. Material classes and texture slots

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Shader names and shader parameter names are **joaat of the lower-case
name** (confirmed): `standard` = `bcf03fc1`, `cable` = `01605013`,
`diffusetex` = `c8e8c282`, `bumptex` = `1e77bf4c`. All 344 shaders over
254 K drawables (1.83 M shader instances) in `levels.rpf` + `packs.rpf` are
named, and 77 of 229 parameters (about 75 % of parameter uses). Earlier
attempts failed only for lack of the right names: the GTA V names tried
before are mostly not used any more. See [Names](#names).

## Shader record bytes +0x38..+0x3F

| Byte | Meaning | Confidence |
|---|---|---|
| +0x3C (byte 4) | render-bucket bit mask (below) | confirmed |
| +0x39 (byte 1) | bucket as an index (1 alpha, 2 decal, 3 cutout, 6 water), **+0x80 for instanced / batch-LOD draws** | confirmed |
| +0x3A..+0x3B | u16, maybe the constant-buffer size | unknown |
| +0x3D | always 0xFF | — |

| Bucket bit | Bucket | Instances | Shaders |
|---|---|---|---|
| 0x01 | opaque | 1.09 M | 193 |
| 0x02 | alpha (glass, bottles, VFX, windows) | 32 K | 44 |
| 0x04 | decal | 596 K | 59 |
| 0x08 | cutout (vegetation, cloth, wire) | 87 K | 90 |
| 0x40 | water | 14.6 K | 2 |
| 0x10, 0x20 | one vehicle shader | — | 1 |

**Emissive has no bucket bit**; only known shader families mark it. A
table of known families adds car_paint, window, glass, emissive, water,
terrain_multilayer.

## Texture slots

| Parameter | Name | Holds | Confidence |
|---|---|---|---|
| `c8e8c282` | `diffusetex` | base colour, sRGB | confirmed |
| `1e77bf4c` | `bumptex` | normal, BC1 (B = 255) | confirmed |
| `84c69350`, `58b6e707` | `lyr0normaltex`, `lyr1normaltex` | normal, BC5 | confirmed |
| `27dbd2be`, `7e1f7f44`, `4c759bf1`, `de84c011` | `bumptexture_layer0` … `_layer3` | terrain layer normals | confirmed (names) |
| `1a016c7e` | `tintbumptexture` | detail normal of the many-layer ground shaders | inferred |
| `c075f9b6` | `lookuptexture` | terrain RGB layer-blend mask | inferred |
| `346394ea`, `7d4f12e2` | `lyr0diffusetex`, `lyr1diffusetex` | colour layers of the map blend shaders | confirmed (names) |
| `7485dafb` | `dirtdiffusetex` | colour with opacity in alpha (grime overlay) | inferred |
| `0df47048` | `diffusetex2` | secondary / detail colour, sRGB | inferred |
| `c6f46e79` | `speculartex` | packed: **R** binary mask (metalness?), **G** roughness or gloss, **B** AO / cavity, A unused | inferred |
| `739e3d0d` | `speculartex2` | packed data. On wet-capable surfaces: R region mask, G nearly flat (puddle blobs), B detailed, A AO-like. **Most maps are R = G = 0, B ≈ 0.5**, so B is not roughness: read that way, surfaces turn mirror-like. Treat the channel meaning as unknown | inferred |
| `19df25d7` | — | format 157 (unknown), vehicle shader | unknown |

**Placeholder textures** (confirmed). Many building materials point their
packed-data slot (`739e3d0d`) at a 4 × 4 all-zero texture, and some point
the normal slot at a 4 × 4 black one. Read literally, the first gives
AO = 0 (black walls) and the second a broken normal. Ignore textures of
8 px or less in the packed and normal slots. A few materials have only such
placeholders, base colour included (e.g. one shader of
`dn_20_int_hse_14_exterior`); they draw as black boxes and are probably
non-visual proxies (inferred).

## Two-layer multilayer shaders (walls, ground pieces, blend decal)

Shaders `b217b263`, `1251cae5`, `cd8ceb5e`, `77d8ab27` (12 parameters) and
`4f0b809b` (terrain blend decal) blend two full texture sets by vertex
colour. Derived from texture sizes and vertex/UV statistics over 4.7 K
materials, not from shader code (as of 2026-11-20).

| Item | 12-parameter family | `4f0b809b` | Confidence |
|---|---|---|---|
| base layer: colour, normal, data, two-channel | p0, p6, p7, p9 | p2, p10, p3, p0 | confirmed (slots of a layer always share an aspect ratio; the layers differ) |
| layer 2: colour, normal, data, two-channel | p4, p10, p5, p2 | p7, p4, p5, p9 | confirmed (same test) |
| overlay (colour + alpha, grime) | p8 | p6 | inferred |
| layer 2 weight | COLOR0.a, the only vertex channel painted over 0..255 | same | inferred |
| layer 2 UV set | TEXCOORD_1 (texel anisotropy matches in 108/117 and 54/72 primitives) | — | inferred |
| base UV set | TEXCOORD_0 (by elimination; TEXCOORD_2 is often constant) | — | inferred, weak |
| COLOR0 r, g, b | r mostly 255, g within 0..45, b a few distinct values | — | unknown meaning |
| data maps (BC7) | R varies, G about constant per texture, B and A vary | — | unknown meaning |
| two-channel maps (BC1) | R = 0, G and B anti-correlated | — | unknown meaning |

The names agree with this layout (found independently): in the
12-parameter family the base layer is `lyr0diffusetex`, `lyr0normaltex`,
`lyr0materialbtex` (data), `lyr0materialatex` (two-channel); layer 2 is the
same with `lyr1`; the overlay is `dirtdiffusetex`; p1 is `tintpalettetex`.

The 17-parameter ground shaders (`standard_uber_4lyr_*`: `360e396b`,
`b21f1ba5`, `61e09df7`) are a different, many-layer family. Their slots are
named `diffusetexture_layer0` … `_layer3`, `bumptexture_layer0` …
`_layer3`, `lookuptexture`, `tinttexture`, `tintbumptexture` and
`grassmasktexture`; four more per-layer slots (`42110214`, `3ea37b39`,
`3022de38`, `0cd6179f`) are unnamed.

**Normal maps are DirectX convention (green = +Y down)** (confirmed by an
integrability test on 270 of 300 maps). Invert G for glTF/OpenGL.

## Names

Where the names come from (all confirmed by the hash match):

- **Shader names**: `common.rpf` holds a text list of the shader effect
  files (`cable.fx`, `glass.fx` …) and, in another entry, the shader preset
  names (`standard_vehicle_2lyr`, `ped_standard_skin`, `trunk_wind` …).
  joaat of these names gives all 344 shader hashes. Four hashes also match
  an unrelated string; the preset name wins.
- **Parameter names**: besides its compressed cache, `shaders.rpf` has
  uncompressed blocks with NUL-terminated names (techniques, sampler states,
  some textures) and 24-byte binding records. Variables are
  `{u32 hash, u32 same hash, 0x30, …}`; texture bindings are
  `{u32 texture hash, u32 parameter hash, 02 01 xx xx, 00 80 00 00, …}`,
  e.g. (`diffusetexture`, `diffusetex`), (`bumptexture`, `bumptex`). 88 of
  89 sampled drawable parameter hashes occur in these records.
- **Word search**: names that are not stored as text were found by joining
  words and testing joaat. A layer set confirms itself when the same name
  matches every layer (`lyr0materialatex` … `lyr3materialatex`). A changing
  last character does not: names that differ only in the last character
  share the joaat state before it, so `x0` … `x3` match for every prefix
  with the same state. Single matches of long multi-word names are noise
  and were dropped.
- The texture-dictionary key at TXD +0x20 is plain joaat of the texture
  name (lower case), as in GTA V.

## Open questions

- Names of the other 152 parameters (mostly constants) and of the four
  per-layer slots of the ground shaders.
- The 64-bit texture hash ([§5](05-ttpf.md)): FNV-1/1a 64, xxHash64,
  MurmurHash64A, CRC-64 (ECMA, XZ, ISO) and a 64-bit one-at-a-time over
  46 K known names (texture-dictionary texture names, model names, XML
  identifiers), as given, lower and upper case, match no texture hash or
  either 32-bit half beyond chance.
- Whether `c6f46e79` G is roughness or gloss; emissive and glass flags for
  unknown shader families.


---

← [11. Vehicle paint](11-vehicle-paint.md) · [Index](README.md) · [13. Collision and surface categories](13-collision.md) →
