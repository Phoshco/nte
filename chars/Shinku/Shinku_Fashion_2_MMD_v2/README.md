# Shinku Fashion 2 — MMD model (v2)

`Shinku_Fashion_2.pmx` + `textures/`. Built from the audited base conversion
(`Shinku_Fashion_2_MMD` package) by the spec-driven refinement pipeline
(`work/fashion2/scripts/build_fashion2_v2.py`): physics generation → parameter transplant
from the hand-ported `Shinku_JK.pmx` reference → rigid-frame correction → material/morph/
collider/skirt/rig specs → initial-overlap resolution.

| | base package | this build |
|---|---:|---:|
| vertices / triangles | 73952 / 102790 | 73952 / 102790 (unchanged) |
| materials | 12 (all Blender defaults) | 12 (audited values) |
| bones | 261 | 261 (2 renamed, 7 flag/parent edits) |
| morphs | 88 raw UE flexes | 99 (36/42 standard MMD names) |
| rigid bodies | 0 | 172 |
| joints | 0 | 183 |
| textures shipped | 51 files | 9 (only what the model references) |

**Geometry and skinning are provably untouched.** The vertex-position + bone-weight
signature (rename-proof: weights keyed by bone *position*) is identical before and after
the whole chain: `4fbfe6490cf40760`.

## Physics

148 dynamic bodies in 10 families, parameters transplanted from the hand-made reference
model (same Unreal skeleton — 115 bones at byte-identical positions, including every
hair/tail/accessory chain):

- **Name/depth transplants** (60): tail, side hair (changbinfa/binfa), bangs (liuhai),
  ahoge (daimao), head bow (toushi + hudiejie), shoulder hair (jianfaA). Chains whose
  lengths differ from the reference are resampled by normalised depth, never name-matched
  (the reference's 錘 free-swing weight entries are filtered out of every curve).
- **Analogy** (44): the twin ponytails (mawei, 42 bodies) have no counterpart in the
  school-uniform reference and borrow the long-side-hair schedule; jianfaB likewise.
- **Explicit** (2): breast, standard MMD values (mass 0.4, damping 0.85/0.90, ±7/5/7°,
  spring 40), collision with the chest collider disabled (they spawn inside it).
- **Tail shapes**: the tail is the same physical appendage as the reference's
  (perpendicular radii per bone within 15%, most within 5%), so the reference's hand-picked
  capsules are transplanted for segments 1–7. This alone took tail motion from 0.64 to
  0.38 of model height — the generator's `length − 2·radius` capsules had the wrong
  inertia. The tip (weiba008) keeps the generator body: refitting it to its own fatter
  mesh was measured and made the tail *worse* (0.38 → 0.42), because a longer, fatter tip
  adds torque arm at the end of the cantilever.
- **Skirt**: the "skirt" is a **front-open long coat** — 6 chains × 7 rows, mesh azimuth
  bins ±10° around the front are empty and the hem rises from y≈7.3 at ±65° to y≈13.6 at
  ±15°. It gets 42 dynamic box panels (mass 3.0→0.3, damping 0.90→1.00 / 0.99→1.00 per
  the reference's own 6-row grid idiom), 42 vertical joints (row 1 near-welded to the
  pelvis body, widening to −20..+34° at the hem), and **35 ring joints — the frb↔flb pair
  across the open front is deliberately absent** (an open coat's front edges are free).
  The skirt collides only with its five group-11 leg guards (hip, thighs, shins), all
  verified to clear every panel by ≥ 0.195 at bind.
- Rigid frames rebuilt in the PMX-native intrinsic ZXY convention: capsule/bone alignment
  **median 0.0°, p90 0.023°** (94.1% within 1°; every body over 1° is a deliberately
  horizontal passive proxy — pelvis, bust, hip guard, head, shin guards — no chain capsule
  is misaligned).
- Families that spawn inside a body collider (all 6 shoulder-hair bodies, both breast
  bodies) have that pairing disabled; the 7-of-90 hair bodies grazing the head by ≤0.06
  keep their collision (losing hair-vs-head lets the bangs sink through the face).

Measured over a 150-frame settle plus a 180-frame driven animation (legs swinging ±35°,
hips/torso twisting, head nodding, 30 fps), resting displacement as a fraction of model
height (lower is better; reference = the hand-made model's own families):

| family | settle | motion | reference (motion) |
|---|---:|---:|---:|
| skirt (coat) | 0.015 | 0.031 | — |
| ponytails | 0.026 | 0.075 | — |
| side hair | 0.009 | 0.068 | 0.108 |
| bangs | 0.005 | 0.033 | 0.094 |
| head bow | 0.007 | 0.050 | — |
| ahoge | 0.018 | 0.046 | 0.036 |
| shoulder hair | 0.042 | 0.038 | 0.046 |
| breast | 0.000 | 0.018 | — |
| tail | 0.628 | 0.380 | 0.224 |

Every family retains a healthy range of motion (motion peaks 0.13–0.41); nothing is
stiffened into a weld. No body ends below the floor; no non-finite transforms.

## Morphs

99 morphs: 70 kept + 16 renamed + 10 group + 3 baked, delivering **36 of the 42 standard
MMD names** (eyebrow 8/8, eye 13/15, mouth 15/16). 75 exposed in the 表情 frame, ordered
eyebrow → eye → mouth. `name_e` keeps the untouched UE flex name on every morph.

- The UE `l/r` suffixes are screen-side on the smile flexes: measured on this model,
  `EL_Happy_l_CLO` moves the character's **right** eye. The wink block is therefore
  ウィンク=`EL_Happy_r_CLO`, ウィンク右=`EL_Happy_l_CLO`, ウィンク２=`biyan_L`,
  ｳｨﾝｸ２右=`biyan_R`.
- Dropped: `まばたき２` (bit-identical duplicate of まばたき, max offset diff 0.0) and
  `強まばたき` (`TD_EyesClo` translates the eyelid island 0.30 backwards into the skull —
  it is a mask for a 表情 decal mesh this outfit does not ship).
- 下 (brow lower) is baked at −0.5: measured clearance to the lashes is 0.053 at −0.5 and
  0.012 (touching) at −1.0.
- **Not deliverable, not faked** (need geometry this mesh does not contain): `ω`, `星目`,
  `はぁと`, `照れ`, `涙`, `がーん`.

## Materials

All 12 materials carried the mmd_tools Principled-BSDF defaults (0.8 grey diffuse,
specular (1,1,1)@11.9, ambient 0.4). Replaced with audited values: diffuse white,
specular off, ambient 128/255 head parts / 192/255 costume, black edges 0.5 with the
reference's reddish edge on the face; edge bit per UE outline ground truth (衣装_04 is
the one costume part whose outline UE explicitly overrides to none).

- **目影 rebound**: UE binds `eye_bantou_d` (a uniform *light* blue-grey) but multiplies
  it by BaseColor **black** (MSM_Unlit). The mesh's own UVs land on the face atlas' flat
  dark swatch (RGB 43/29/27), which is exactly what the hand-ported reference binds
  (`face_d` at diffuse alpha 0.5). Reproducing the look, not the binding, and dropping
  `eye_bantou_d` from the package.
- **衣装_04** is a genuine alpha cutout (30.8% of used texels below UE's 0.333 clip);
  **髪_01** likewise (30.0% below 1.0) and is drawn last. Everything else is opaque at its
  own UVs.
- **胸元FX**: UE renders the chest star as a translucent *additive* unlit FX. Shipped as
  the base package's deterministic bake (`RGB×(1.68,0.30,2.0)`, alpha=max(RGB),
  sha256-verified) with ambient (1,1,1), no edge/shadow flags — an alpha-blend
  approximation of an additive blend.
- No sphere maps (all matcaps are ID-zone-gated remaps MMD cannot express) and no toon
  (the UE ramps are near-unity tint multipliers), matching the reference.

## Rig

Weight-safe fixes only: 上半身1/上半身2 → 上半身2/上半身3 (torso chain was off by one —
the renamed bones sit at the reference's 上半身2/3 positions); 左目/右目 re-parented to 頭
with 回転付与 from 両目 (they were hierarchical children, so a 両目 key orbited the
eyeballs into the skull); 両目 lifted to the reference's above-head position; 肩P/肩C
flag parity with the reference on both sides. Display frames rebuilt (11 frames, no bone
in two frames; 物理操作 holds the 23 physics chain roots).

Deliberately **not** done:
- Twist-master insertion (renumbers every bone index; 腕捩/手捩 already carry the twist
  weights, so VMDs keying them work today).
- **Horn physics.** `Bn_L/R_jiao01..04` are weighted (340–526 verts each) but the
  hand-made reference keeps the identical horns static — they are rigid armor pieces, so
  this build follows the reference and leaves them bone-driven.

## Known limitations

- **The tail hangs and swings wide.** It is a 16.4-unit horizontal armored cantilever on a
  ~20-unit model; the reference's identical tail also hangs (0.71 of model height in
  settle vs ours 0.63). Under driven motion ours rests at 0.380 vs the reference's 0.224:
  the reference wins through an extra 9th segment plus a separate 4-body side strand that
  this outfit's skeleton does not carry (both were unweighted in this mesh and removed at
  source). Formally this fails the 1.5× baseline gate (0.357), exactly as Fashion 1's
  shipped tail did (0.358).
- **Shoulder hair rests 0.042 in settle** vs the reference's 0.010; under actual motion it
  is *better* than the reference (0.038 vs 0.046). This outfit's shoulder hair is a
  different 2-segment asset (reference: 6 segments), derived by depth-resampling.
- The open coat's front edges are not ring-joined by design; a hard sideways hip snap can
  momentarily scissor the two lapels apart. 116 mesh vertices sit in the ±20° front span
  above y=13.59 (the collar); if lapel separation is visible in a specific motion, add a
  row-1-only frb↔flb ring joint.
- Hair collision proxies are thinner than the hair mesh (generator sizing kept on
  purpose): copying the reference's fatter bang proxies was measured to spawn them 0.19
  inside the head collider and the frame-1 depenetration kick regressed the bangs' settle
  9×. Raise `size[0]` per family only if a specific motion shows clipping.
- Group morphs with confidence ≤ 0.55 (`▲`, `なごみ`, `はぅ`, `口角上げ/下げ`, `口横広げ`,
  `眉頭左/右`) are compositions, not authored poses.
- `う` is bound to `jawOpen_wu` (the artist-named phoneme with the jaw drop); the
  reference binds `mouthFunnel`. Rebind if lipsync presets built against the reference.

## Reports

`reports/`: `generate.json` (physics generation contract), `retune.json` (per-body
strategy + before/after), `frames.json` (per-capsule rotation correction), `refine.json`
(spec application), `overlaps.json` (collision pairings dropped and why),
`physics_validation.json` (settle + driven motion, baseline-gated),
`axis_probe_final.json` (capsule/bone alignment), `build_summary.json` (counts + weight
signature). The audit and adversarial-verification evidence lives in
`work/fashion2/reports/` and `work/fashion2/specs/verify_*.json`.
