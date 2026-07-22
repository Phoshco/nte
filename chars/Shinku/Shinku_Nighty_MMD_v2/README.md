# Shinku Nighty — MMD model (v2)

`Shinku_Nighty.pmx` + `textures/`. Built from the physics-less base conversion
(`Shinku_Nighty_MMD.pmx`, 301 bones, 0 rigid bodies) by the spec-driven pipeline
(`work/nighty/scripts/build_nighty_v2.py`): existing-bone physics generation →
parameter transplant from the hand-ported `Shinku_JK.pmx` reference (same Unreal
skeleton, 121 bones at byte-identical positions) → PMX-native frame correction →
five audited spec files → initial-overlap resolution.

| | base conversion | this build |
|---|---:|---:|
| vertices / triangles | 63395 / 93605 | 63395 / 93605 (unchanged) |
| materials | 9 (all Blender defaults) | 9 (audited values) |
| bones | 301 | 301 (34 renamed, 7 edited) |
| morphs | 88 (invented names) | 99 (36/42 standard MMD names) |
| rigid bodies | 0 | 214 (24 passive, 190 dynamic) |
| joints | 0 | 220 |
| textures shipped | 7 | 6 (eye_bantou_d dropped after the 目影 rebind) |

**Geometry and skinning are provably untouched.** The vertex-position + bone-weight
signature (weights keyed by bone *name*, with the deliberate renames mapped) is identical
between the base conversion and this build (`aac0f12b8a322518`), and the index-based
signature is identical across the whole physics chain (`0c87a52799348cec`).

## Physics

190 dynamic bodies in 10 families, parameters taken from the hand-made reference wherever
it has the same bone (exact transplant when chain lengths match, depth-resampled when they
differ, behavioural analogy otherwise):

- **Nightgown hem** (`qunbai`, 30 bodies): the bone grid is byte-identical to the Fashion 1
  outfit, so its validated skirt design carries over with panel geometry re-measured against
  the *nightgown* mesh (hem to y≈10.4, radius to 2.49 — wider and longer than the dress):
  10 box panels × 3 rows, closed ring joints per row (30), verticals anchored to the pelvis
  collider (30), five skirt-only leg-guard colliders in group 11, per-row mass 2.0/1.1/0.45
  and rising damping.
- **Tail** (`weiba`+`weipd`, 12): reference schedule — ±10° limits, damping 1.0, mass taper
  0.1→0.01, collision mask 0xF800. The umbrella-strap (`weipd`) capsule shapes transplanted
  verbatim (mesh-signature match).
- **Twin ponytails** (`mawei`, 42): no reference counterpart (the reference re-rigs its back
  hair with hand-made chains) → depth-resampled from the reference's long side-hair family.
- **Side/front hair, ahoge, shoulder hair** (`changbinfa`/`binfa`/`liuhai`/`daimao`/`jianfa`,
  41): transplanted or depth-resampled from the same-named reference chains.
- **Head bow** (`toushi`+`hudiejie`, 19): exact transplant — same bones, same chain lengths,
  capsule shapes copied (mesh-signature match).
- **Bow accessories** (44, Nighty-specific): back-waist bow (`Bone_Bow`), chest bows
  (`Bone_BowL/R`), left-thigh bow (`Bone_BowT`) — no reference counterpart → butterfly
  (head-bow) analogy: light, heavily damped, near-locked ribbon clusters.
- **Breast** (`xiong`, 2): explicit MMD-standard values (mass 0.4, damping 0.85/0.90,
  ±7/5/7° limits, spring 40, no collisions).

19 passive body colliders were rebuilt with **full bone-segment heights** (the generator's
`length − 2·radius` bug had clamped the torso capsules to 0.05) and radii measured from
*this* mesh — the sleeveless nightgown is slimmer than Fashion 1's dress (upper arm r 0.50
vs 0.65, forearm 0.45 vs 0.55, abdomen 1.30 vs 1.40). Palm capsules keep the twin-tails
out of the hands. Mask carve-outs: bow bit cleared from the abdomen, lower-chest, bust and
left-thigh colliders (the back-waist bow starts up to 0.467 inside the abdomen capsule, the
thigh bow rests on the thigh); breast bit cleared from the bust. The overlap pass dropped
two structurally-embedded pairings (shoulder-hair × body 7/8 embedded, breast × body 2/2)
and kept incidental contact (hair × body 9/94, bow × body 1/44).

Every capsule frame is written in the PMX-native intrinsic-ZXY convention: alignment probe
**median 0.0°, p90 0.022°, 96.2% within 1°** — the only >1° entries are the deliberately
X-axis/vertical shapes (hip, bust, hip-guard, head, palms).

Validation (150-frame settle + 180-frame driven animation at 30 fps, gated per family
against the reference model's own run):

| family (settle / motion final) | this build | reference |
|---|---:|---:|
| skirt (hem) | 0.0044 / 0.0111 | — |
| chest_bow (4 bow assemblies) | 0.0158 / 0.0531 | — |
| ponytail | 0.0285 / 0.0723 | — |
| side hair | 0.0172 / 0.0647 | 0.0163 / 0.1076 |
| front hair | 0.0055 / 0.0316 | 0.0062 / 0.0936 |
| head ornament (ahoge) | 0.0170 / 0.0503 | 0.0021 / 0.0360 |
| head bow (in "other") | 0.0264 / 0.0791 | 0.0546 / 0.2473 |
| shoulder hair | **0.0436** / 0.0416 | 0.0104 / 0.0458 |
| breast | 0.0025 / 0.0212 | — |
| tail | 0.6598 / **0.3861** | 0.7107 / 0.2243 |

No body ends below the floor, no non-finite transforms, and every family retains a healthy
peak range of motion (motion-pass peaks 0.13–0.45) — nothing is welded.

The two bold numbers exceed the 1.5×+0.02 reference gate and are **accepted, documented
asset properties** — the shipped Fashion 1 exemplar carries the *same two* exceedances
(shoulder_hair settle 0.0436 — bone-for-bone identical to this build; tail motion 0.3583
vs ours 0.3861):

- **Tail**: a 16.4-unit horizontal cantilever; the reference's own tail rests at 0.71 of
  model height in settle (ours 0.66). The reference wins the motion pass because it has a
  9th tail segment; that bone (`Bn_weiba009`) carries **zero skin weights** in the Nighty
  mesh and was correctly removed — re-adding it would need re-weighting, out of scope.
- **Shoulder hair**: 3 segments where the reference has 6; the reference distributes its
  sag over twice as many locked joints. Under actual *motion* this build is better than the
  reference (0.0416 vs 0.0458).

## Morphs

Same 88-flex Unreal set as Fashion 1 (verified per-morph: identical offset counts, side
signs, duplicates), so the audited Fashion 1 morph spec applies. Re-measured on this mesh:
the UE `l`/`r` suffixes are screen-side (mirrored) — `EL_Happy_l_CLO` moves the character's
**right** eye; `まばたき２` is bit-identical to `まばたき` (dropped); `強まばたき` pushes the
eyelids 0.30 into the skull instead of closing them (a decal-mask morph for a mesh this
outfit does not ship — dropped); the baked 下 at weight −0.5 clears the lashes by 0.032
(per-column clearance min 0.106, drop 0.074).

Final: 99 morphs — 86 kept/renamed vertex morphs, 10 groups, 3 baked — delivering
**36 of the 42 standard MMD names** (eyebrow 8/8, eye 13/15, mouth 15/16, other 0/3).
75 exposed in the 表情 frame, ordered eyebrow → eye → mouth. Standard wink pair mapped by
measurement: ウィンク=`EL_Happy_r_CLO`, ウィンク右=`EL_Happy_l_CLO`, ウィンク２=`biyan_L`,
ｳｨﾝｸ２右=`biyan_R`.

Six standard names are **unavailable, not faked**: `ω` (no independent lip-corner control
in the source rig), `星目`, `はぁと`, `照れ`, `涙`, `がーん` (need decal geometry this mesh
does not contain).

## Materials

All nine materials shipped with mmd_tools' leaked Blender defaults (diffuse 0.8 grey,
specular (1,1,1)@11.9, ambient 0.4 — specular blowout on the dark hair). Replaced with the
audited values: diffuse white, specular disabled, ambient 128/255 head / 192/255 costume,
black edge 0.5, reddish edge on the face, 目ハイライト unlit (ambient 1.0).

The head/hair materials are triangle-for-triangle identical to the audited Fashion 1 build
and adopt its verified values — including the **目影 rebind**: UE binds `eye_bantou_d.png`
but tints it *black* (BaseColor (0,0,0,1), MSM_Unlit); reproducing the binding without the
tint would turn a dark eye shadow into a light wash. It is rebound to the dark swatch its
own UVs occupy in the face atlas (sampled: RGB 0.169/0.114/0.106) at diffuse alpha 0.5.
The two nighty costume materials keep edge ON (UE supplies outline materials) and are
verified opaque at their own UVs. `髪_01` (bangs) keeps its genuine alpha cutout
(min 0.0, 21.2% of used texels below 0.9) and is conveniently the **last** material in
draw order in this outfit. No sphere maps, no toon (matches the reference; the UE matcaps
are ID-masked mid-grey hemispheres that MMD cannot remap).

## Rig

Weight-safe fixes only:

- **`上半身1`/`上半身2` → `上半身2`/`上半身3`** — the torso chain was off by one
  (positions prove it: y=14.54/15.39 match the reference's 上半身2/3).
- **Finger chain off by one** (Nighty-specific, caught by audit): the base names the
  metacarpals `指１` and the tips `指４`; a stock VMD's `人指１` curve would bend the
  metacarpal. All 32 non-thumb finger bones renamed down by one (`指１..４` → `指０..３`) —
  verified position-for-position against both the reference and Fashion 1.
- **Eye rig**: `左目`/`右目` re-parented to `頭` with 回転付与 from `両目` (weight 1.0,
  flags 0x11A); `両目` made rotation-only (0x1A) and lifted above the head — as children of
  `両目` a gaze key *orbited* each eyeball into the skull.
- `肩P` 0x1A / `肩C` 0x112 flag parity with the reference, both sides.
- Display frames rebuilt: Root / 表情 / センター / ＩＫ / 体 / 左腕 / 右腕 / 左指 / 右指 /
  足 / 物理操作 (31 physics chain roots), no bone in two frames.

Deliberately **not** done: twist-master insertion (renumbers every bone index; the existing
`腕捩`/`手捩` bones already carry the twist weights), 足D/足先EX (need weight moves),
`Bn_L/R_jiao` horn physics (the hand-made reference deliberately leaves the same bones
rigid).

## Reports

`reports/`: `generate.json`, `retune.json` (per-body strategy + before/after),
`frames.json` (147 capsules re-framed, median correction 24.5°), `refine.json`,
`overlaps.json`, `physics_validation.json` (full settle + driven results),
`axis_probe.json`, `build_summary.json`.

## Known limitations

- The tail hangs under gravity and swings further than the 9-segment reference tail
  (see Physics above) — asset property, matches the shipped Fashion 1 exemplar.
- Shoulder-hair static settle (0.0436) exceeds the reference gate; identical number to the
  Fashion 1 exemplar, and better than the reference under motion.
- 目影 is drawn at material index 1, before the face/eyes it overlays — in edge-on views it
  blends against the background. Same known limitation as the shipped Fashion 1 exemplar;
  fixing it needs a face-buffer permutation, deliberately out of scope.
- `髪_01` keeps the edge bit despite its alpha cutout; if the target MMD build outlines
  transparent texels, clear flag bit 0x10 on that material.
- The nightgown hem mesh extends slightly below the last weighted skirt bone row; the
  row-3 panels are extended to the measured hem (per-chain 2nd percentile), but the very
  lowest drape vertices follow `Bn_*_qunbai03` rigidly.
- Bow assemblies do not collide with the torso/left-thigh colliders they rest against
  (deliberate carve-outs); they are held by their near-locked joints. In extreme poses they
  can intersect the body visually.
- Group morphs with confidence ≤0.55 in the morph spec (`▲`, `なごみ`, `はぅ`,
  `口角上げ/下げ`, `口横広げ`, `眉頭左/右`) are compositions, not authored poses; each spec
  entry records a zero-risk fallback.
- `う` is bound to `jawOpen_wu` (the artist-named phoneme); the reference binds
  `mouthFunnel`, which is kept separately as `口漏斗`.
