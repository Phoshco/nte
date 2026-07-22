# Shinku RPG — MMD model (v2)

`Shinku_RPG.pmx` + `textures/`. Built from the base conversion
(`Shinku_RPG_MMD.pmx`, geometry/weights/skeleton correct but no physics, leaked
default materials and raw UE morph names) by the spec-driven refinement pipeline
(`work/rpg/scripts/build_rpg_v2.py`): physics generation on existing weighted bones →
parameter transplant from the hand-ported `Shinku_JK.pmx` reference → tail capsule
mesh fit → capsule/joint frame correction → spec application (materials, morphs,
colliders, rig) → initial-overlap resolution.

| | base conversion | this build |
|---|---:|---:|
| vertices / triangles | 53874 / 73878 | 53874 / 73878 (unchanged) |
| materials | 9 | 9 |
| bones | 227 | 227 (renames only) |
| morphs | 88 raw UE flexes | 99 (36 of 42 standard MMD names) |
| rigid bodies / joints | 0 / 0 | 135 / 116 |
| textures shipped | 7 | 6 (only referenced) |

**Geometry and skinning are provably untouched.** The vertex-position + effective
bone-weight signature (rename-aware: the documented `上半身1→上半身2→上半身3` rename map is
applied to the base names) is identical before and after the whole chain:
`f3dc6f37154a48d1` (`reports/weight_signature.json`).

**Input selection.** The released variant is the complete level-1 production skin
(`player_076_shinku_rpg_level1_skin.uemodel`). The raw export also contains a
`Shinku rpg 0` folder — a separate *modular level-0 costume* (head/cloth/pants/shoes
parts). As in the base build, it is deliberately **not** mixed into this model; it is an
alternate outfit, not missing pieces of this one (`base .. data/model_config.json`).

## Physics

The base PMX shipped **zero** rigid bodies. This build gives every weighted accessory
chain working physics:

- **Families**: twin ponytails (`mawei`, 42 bodies incl. front/side sub-strands), long+short
  side hair (`changbinfa`/`binfa`, 18), bangs (`liuhai`, 11), ahoge (`daimao`, 4), head-bow
  butterfly cluster (`toushi`/`hudiejie`, 19), shoulder hair (`jianfa`, 8 — left side has an
  extra B-strand), armored tail + strap pendant (`weiba`+`weipd`, 12), breast (`xiong`, 2),
  plus 19 mesh-fitted passive body colliders (torso ×4, bust, neck, head, shoulders,
  arms, palms, thighs, calves).
- **Parameters are transplanted, not guessed.** The hand-ported reference is built on the
  same Unreal skeleton (200 shared bone names, 121 at byte-identical positions; every
  physics bone family present). 19 bodies took exact same-chain-length transplants
  (butterfly + accessory root), 51 were depth-resampled from the same family's reference
  curve (tail 12, side hair 18, bangs 11, ahoge 4, shoulder hair 6 — the reference chains
  are longer versions of the same assets), 42 ponytail bodies use the long-side-hair
  analogy (the reference JK simulates its ponytails on a separate rear-hair grid), 2
  jianfaB bodies use the jianfaA schedule, and the 2 breast bodies use explicit
  MMD-standard values (mass 0.4, damping 0.85/0.90, ±7°/±5°/±7°, spring 40).
- **Tail capsules are sized to this outfit's mesh** (`reports/tailfit.json`). The RPG tail
  is a different, far bulkier asset than the JK/Fashion-1 tail (median perpendicular
  radius ~1.0 at the root vs 0.47), so the generator's 0.30-capped capsules carried ~10%
  of the mesh-implied inertia and the chain swung wildly (driven-motion final 0.470).
  Re-sizing each capsule to the measured mesh radius (span preserved) brought it to
  0.128 — better than the hand-made reference's 0.224.
- **Frames**: every capsule's rotation rebuilt in the PMX-native intrinsic ZXY convention
  (the generator's `to_euler("XYZ")` left them a median ~29° off). Final capsule/bone
  alignment: **median 0.007°, 97.1% within 1°, p90 0.022°** (`reports/alignment_probe.json`;
  the two >1° bodies are the deliberately horizontal pelvis/bust capsules).
- **Initial overlaps**: shoulder hair (8/8 bodies) and breast (2/2) spawn inside body
  colliders by construction, so those two group pairings are disabled (Bullet would eject
  them on frame 1). The 9-of-94 hair bodies incidentally grazing the head keep their
  collision — losing hair-vs-head would let the bangs sink through the face.

Validated with sequentially-stepped 150-frame settle + 180-frame driven motion at 30 fps,
gated per family against the same validation of the hand-made reference
(`reports/physics_validation.json`; final displacement from bind ÷ model height):

| family | settle (ref) | motion (ref) | gate |
|---|---:|---:|---|
| tail (weiba+weipd) | 0.647 (0.711) | 0.128 (0.224) | pass |
| ponytails (mawei) | 0.029 (—) | 0.363 (—) | no ref family; vs the JK rear-hair grid 0.247 it is within 1.5× |
| side hair | 0.012 (0.016) | 0.111 (0.108) | pass |
| bangs | 0.005 (0.006) | 0.033 (0.094) | pass |
| ahoge | 0.017 (0.002) | 0.074 (0.036) | pass |
| head bow (in "other") | 0.025 (0.055) | 0.086 (0.247) | pass |
| shoulder hair | **0.044 (0.010)** | 0.040 (0.046) | **settle fails — known limitation, see below** |
| breast | 0.0004 (—) | 0.005 (—) | no ref family (explicit values) |

No body below the floor, no non-finite transforms, every family retains motion range
(no welds).

## Morphs

88 raw UE flexes → 99 morphs: 86 kept/renamed, 10 group morphs, 3 baked (negative weights
are pre-baked into plain vertex morphs so no downstream tool sees them). **36 of the 42
standard MMD names** (eyebrow 8/8, eye 13/15, mouth 15/16); 75 exposed in the 表情 frame in
eyebrow → eye → mouth order. Group morphs sit after their members in file order.

- The wink pair follows the *measured* offset side, not the UE suffix (the `l/r` suffixes
  are inconsistent — `EL_Happy_l_CLO` moves the character's **right** eye):
  `ウィンク` = EL_Happy_r_CLO, `ウィンク右` = EL_Happy_l_CLO, `ウィンク２` = biyan_L,
  `ｳｨﾝｸ２右` = biyan_R.
- Dropped: `まばたき２` (bit-identical duplicate of `まばたき`, verified element-wise) and
  `強まばたき` (broken in this export: it translates the eyelid island +0.30 into the skull —
  it is a mask for a 表情 decal mesh this skin does not ship).
- **Declared unavailable, not faked** (need geometry/decals absent from this export):
  `ω`, `星目`, `はぁと`, `照れ`, `涙`, `がーん`.

## Materials

All nine materials carried the mmd_tools Principled-BSDF defaults (diffuse 0.8 grey,
specular (1,1,1) @ 11.9, ambient 0.4) — none were real. Replaced with values from the UE
material-instance JSONs, pixel-sampling each texture at the material's own UVs, and the
reference's conventions: diffuse white, specular (0,0,0) @ 5.0, ambient 128/255 (head) /
192/255 (costume), black edge 0.5, reddish edge on the face only.

- `目影` (eye shadow): UE binds `eye_bantou_d` — a uniform light blue-grey — then
  multiplies it by **black** (`BaseColor 000000`, unlit translucent). Reproducing the
  binding would invert a dark shadow into a light wash. Bound to the face atlas' dark
  swatch (verified by sampling this material's own UVs) with diffuse alpha 0.5.
- `衣装_01/02` (level-1 costume): albedo is `Textures.BaseColor` (= `*_d`); `PM_Diffuse`
  points at the `*_id` ID mask and was not bound. Both fully opaque at their own UVs.
- `目ハイライト`: ambient (1,1,1) — the one deviation from the head convention, backed by
  UE RampID 24 (bright/emissive ramp row).
- `髪_01` is the only genuine alpha-cutout (min alpha 0.0, 12.3% below 0.5).
- No sphere maps, no toon (matches the reference).
- `eye_bantou_d.png` became unreferenced and is not shipped.

## Rig

Weight-safe fixes only (verified: skinning signature identical):

- `上半身1/上半身2` → `上半身2/上半身3` — torso chain was off by one (base `上半身1` sits at the
  reference's `上半身2` position y=14.54); a stock VMD's `上半身2` curve now drives the right
  pivot.
- Eye rig: `左目`/`右目` re-parented to `頭` with 回転付与 1.0 from `両目` (flags 0x011A); as
  plain children of `両目` a VMD eye key *orbited* the eyeballs into the skull. `両目`
  becomes a rotation-only control (0x001A) lifted to the reference position above the head
  (weight-free, so safe).
- `肩P` 0x001A / `肩C` 0x0112 flag parity with the reference, both sides.
- Display frames rebuilt: `Root`/`表情` special, センター, ＩＫ, 体, 左腕/右腕, 左指/右指, 足,
  物理操作 (the 17 physics chain roots). No bone appears twice.

Deliberately **not** done:

- No skirt physics — **the RPG level-1 costume has no skirt**. The raw skeleton's `qunbai`
  bones are unweighted in this skin and were removed by the base build; the outfit is
  shorts + shirt + boots (see previews).
- `Bn_L/R_jiao` horn chains (weighted, 4 bones/side) carry **no physics on purpose**: the
  hand-ported reference has the same bones at byte-identical positions and gives them no
  rigid bodies — horns are rigid accessories.
- Twist-master insertion (`腕捩1..3`) and `足D`/`足先EX`: renumber bones / need weight moves;
  the existing `腕捩`/`手捩` bones already carry the twist weights.
- Level-0 modular costume excluded (alternate outfit, see Input selection).

## Reports

`reports/`: `generate.json`, `retune.json` (per-body strategy + before/after),
`tailfit.json`, `frames.json`, `refine.json`, `overlaps.json`,
`physics_validation.json`, `alignment_probe.json`, `weight_signature.json`,
`build_summary.json`.

## Known limitations

- **Shoulder hair settles at 0.044 vs the reference's 0.010** (the one family failing the
  1.5×+0.02 settle gate). Same numbers as the accepted Fashion 1 build (0.0436): this
  outfit's shoulder hair is a 3-segment version of the reference's 6-segment asset, all its
  joints are fully locked (the sag is quasi-static Bullet constraint droop, not swing), and
  it does not worsen under motion — 0.040 vs the reference's 0.046, i.e. *better*.
  Transplanting the reference's name-matched masses was tried and measured no better
  (0.0440).
- The tail hangs under gravity (settle 0.647): it is a ~15-unit horizontal armored
  cantilever; the hand-made reference's own tail hangs *further* (0.711). Asset property,
  not a defect. Its driven-motion peak (0.81) is larger than the reference's (0.27) — the
  mesh-true capsules store more energy; it stays within its ±10° joint limits and returns
  to rest (final 0.128, better than the reference).
- Driven-motion *final* ratios for long free chains are phase-sensitive (the pass ends
  mid-swing); settle numbers are the stable comparison.
- Group morphs with confidence ≤ 0.55 in the spec (`▲`, `なごみ`, `はぅ`, `口角上げ/下げ`,
  `口横広げ`, `眉頭左/右`) are compositions, not authored poses.
- `う` is bound to the artist phoneme flex `jawOpen_wu` (the reference uses `mouthFunnel`);
  rebind if lipsync presets were built against the reference.
- Hair collision proxies stay thinner than the hair mesh, matching the reference's
  deliberately thin proxies; a violent motion can clip hair through the body briefly.

Converted only from the source assets supplied in this workspace. No donor model geometry,
weights, bones, morph offsets, or physics were used. The hand-ported reference model was
consulted for *parameter values* only.
