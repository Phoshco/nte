# Shinku Swimsuit — MMD model (v2)

`Shinku_Swimsuit.pmx` + `textures/`. Built from the mmd_tools base conversion
(`base/Shinku_Swimsuit_MMD`) by the spec-driven refinement pipeline
(`work/swimsuit/scripts/build_swimsuit_v2.py`), following the ue-to-mmd-port skill.

| | base conversion | this build |
|---|---:|---:|
| vertices / triangles | 55980 / 81304 | 55980 / 81304 (unchanged) |
| materials | 10 | 10 |
| bones | 193 | 193 (reordered parent-first; none added/removed) |
| morphs | 88 raw UE flexes | 99 (36-name standard MMD vocabulary delivered) |
| rigid bodies | 0 | 94 (19 body colliders + 75 dynamic) |
| joints | 0 | 75 |
| textures shipped | 32 in folder / 8 referenced | 7 (referenced only) |

**Geometry and skinning are provably untouched.** The vertex-position + bone-weight
signature is identical from the base conversion to the deliverable
(`dfdbd605d89d37f7`, rename-normalized; `reports/weight_invariance.json`).

## Physics

The base conversion had **no physics at all** (0 rigid bodies, 0 joints). Everything was
built by transplanting from the hand-ported `Shinku_JK.pmx` reference — same Unreal
skeleton, 113 shared bones, and every physics-relevant chain the two models share sits at
**byte-identical positions** (weiba 8/8, liuhai 11/11, jiao 8/8, daimao 4/4).

Families and strategy:

| family | bones | strategy |
|---|---|---|
| front hair (liuhai) | 11, 3 chains | per-bone transplant, depth-resampled (ref chains carry extra 錘 weight bodies) |
| ahoge (daimao) | 4 | transplant, depth-resampled (ref chain is 7 deep) |
| tail (weiba) | 8 | transplant, depth-resampled + documented ±8° stiffening (see limitations) |
| back hair (Bone_hair*) | 32, 3 chains | analogy ← reference back hair (root-heavy 30→0.1 mass taper, rising damping, locked joints) |
| hair bow + trail (Bone_bow*) | 12 | analogy ← reference back hair (see below) |
| neck ribbon (bone_ribbon*) | 6 | analogy ← reference butterfly ties |
| breast (xiong) | 2 | explicit MMD-standard (0.4 mass, 0.85/0.90 damping, ±7/5/7°, spring 40) |

Validated over a 150-frame settle and a 180-frame driven pass (legs/hips/head, 30 fps),
gated per family against the reference model's own numbers
(`reports/reference_physics_fine.json`; gate = ref × 1.5 + 0.02). **All families pass
both passes**; no body below the floor, no non-finite transforms, every family retains
its range of motion (peaks 80–135% of the reference's):

| family | settle ours/ref | motion final ours/ref | motion peak ours/ref |
|---|---:|---:|---:|
| back_hair | 0.042 / 0.045 | 0.115 / 0.247 | 0.425 / 0.314 |
| bow (chest_bow) | 0.012 / 0.005 | 0.131 / 0.196 | 0.386 / 0.298 |
| neck_ribbon* | 0.004 / 0.005 | 0.058 / 0.196 | 0.148 / 0.298 |
| front_hair | 0.005 / 0.006 | 0.024 / 0.094 | 0.148 / 0.184 |
| ahoge (head_ornament) | 0.001 / 0.002 | 0.047 / 0.036 | 0.212 / 0.177 |
| tail | 0.548 / 0.711 | 0.250 / 0.224 | 0.335 / 0.267 |
| breast | 0.000 / 0.003 | 0.005 / 0.009 | 0.129 / 0.144 |

\* gated against the reference butterfly family (no direct counterpart).

Decisions that took iteration (all recorded in the scripts' comments):

- **Dynamic capsule height bug fixed at source.** The generator's `length − 2·radius`
  shortened every link (the P7 pitfall); PMX capsule height is the full bone-segment
  length, which is also what the reference stores. Fixing it restored the tail's inertia.
- **The 8-deep trailing bow ribbon is not a butterfly.** Under the flat-mass butterfly
  schedule it settled 0.045 off bind (gate 0.027); the reference's own long hanging
  chains use a root-heavy 30→0.1 taper, and switching the analogy brought it to 0.012.
- **Bangs keep their face collision.** 7/11 front-hair bodies graze the head collider by
  ≤ 0.06 units at bind; the structural-embedding rule would have disabled hair-vs-body
  for the whole family (the same physical contact passed Fashion 1's 40% rule only
  because its merged family was larger). Overlap threshold raised to 0.08 so shallow
  grazes stay colliding; the truly structural embeddings (neck ribbon 0.60 deep inside
  the neck/chest, breast 0.35 inside the chest capsule) are still disabled.
- **Capsule/bone alignment**: median 0.009°, p90 0.69°; every body off by more than 1° is
  a deliberately-authored torso/head/hand collider fit (the dynamic capsules all align
  within 1°). `reports/axis_probe_final.json`.

## Materials

All 10 materials arrived on leaked Blender defaults (uniform 0.8 diffuse, white specular
@ 11.9 — none real). Rebuilt from UE material-instance JSONs + the reference model +
pixel-sampling each texture at the material's own UVs (`reports/materials_spec.json`):

- **目影 (eye shadow) inversion fixed**: UE binds a light blue-grey `eye_bantou_d.png`
  but tints it *black* under MSM_Unlit; the same UVs on the face atlas sample the dark
  swatch UE actually shows. Rebound to `face_d` with diffuse alpha 0.5 (the reference
  idiom); `eye_bantou_d.png` is no longer shipped.
- Edge flags follow UE outline overrides: ON for face/costume/hair (dedicated outline
  materials), OFF for eye internals/lashes (explicit no-outline override). Face gets the
  reference's reddish edge; everything else black, size 0.5.
- Ambient 128/255 head+hair, 192/255 costume, 1.0 eye highlight; specular (0,0,0)@5;
  no sphere maps (UE matcap slots exist but the reference wires none; Fashion 1's
  measured test rejected the same matcap family); no toon.
- Head-part triangle counts match the reference exactly (目影76 顔5378 睫毛1704 瞳240
  目HL160), so reference floats are ground truth for those.

## Morphs

88 raw UE flexes → 99 morphs; 36 standard MMD names delivered (あいうえお lipsync, まばたき,
winks, brows, ▲∧□ワん, 口角上げ/下げ…), re-measured on this mesh:

- **Wink sides measured, not trusted** (UE l/r suffixes are inconsistent):
  `EL_Happy_l_CLO` moves the character's RIGHT eye → ウィンク右; the standard pair is
  ウィンク (left, smiling close) / ウィンク右, ウィンク２ / ｳｨﾝｸ２右 (plain closes).
- まばたき２ dropped: bit-identical duplicate of まばたき (max offset diff 0.0).
- 強まばたき dropped as broken: `TD_EyesClo` moves the strong upper-lid vertices only 17%
  as far as まばたき — it cannot close the eye on this mesh.
- 下 baked at −0.5 (clearance to lashes 0.053; −1.0 collapses to 0.012 and buries the
  brows), 眉頭左/右 baked at −1.0 (clearance 0.096).
- 表情 frame: 75 morphs, ordered eyebrow → eye → mouth; helper morphs stay VMD-addressable
  but out of the frame.
- **Declared unavailable** (missing decal/geometry — never faked): ω, 星目, はぁと, 照れ,
  涙, がーん (the TD_* decal islands and independent lip-corner control do not exist in
  this export).

## Rig

- Torso off-by-one fixed: 上半身1/上半身2 renamed to 上半身2/上半身3 (positions match the
  reference and the Fashion 1 exemplar at d=0.000000) — stock VMD 上半身2 curves now
  drive the correct pivot.
- Eye rig rebuilt: 左目/右目 parented to 頭 with 回転付与 (rotation-inherit) from 両目 at
  1.0 — a 両目 key now rotates the eyes in place instead of orbiting them into the skull;
  両目 lifted above the head (weight-free), rotation-only.
- 肩P/肩C flags conformed (rotation-only / hidden cancel bone), both sides.
- IK numerics were already conventional (leg loop 40 limit 2.0 rad, knee −180…−0.5°,
  toe 4°) — untouched.
- Display frames: Root / 表情 / センター / ＩＫ / 体 / 左腕 / 右腕 / 左指 / 右指 / 足 /
  物理操作 — no bone appears in two frames.

## Deliberately not done, and known limitations

- **No skirt physics — there is no skirt.** The outfit has zero qunbai-family bones; the
  skirt spec is intentionally empty rather than invented.
- **Horns (jiao) stay rigid.** The reference model has the identical Bn_L/R_jiao01-04
  bones (byte-exact) and gives them no rigid bodies.
- **Hair-crown bases (Bone_hair, Bone_hair01) are static.** They carry ~4500 scalp/bun
  vertices; keeping them out of the sim prevents the crown detaching from the skull and
  keeps branch tips from inheriting mid-curve masses.
- **Tail joint limits are ±8°, not the reference's ±10°.** The reference tail is 13
  bodies (9 weiba + 4 weipd weight bodies) over the same 16.4-unit span; this export has
  only 8 bones. At ref-exact ±10° the coarser chain lagged driven motion at 0.38 of
  model height (gate 0.36); flat ±8° restores parity (0.250 vs ref 0.224) and the tail
  still settles *shallower* than the reference (0.548 vs 0.711). This is the only
  deliberate deviation from transplanted reference values.
- **Neck ribbon and breast do not collide with the body.** Both spawn structurally
  embedded (0.60 / 0.35 units deep); Bullet would eject them on frame 1. Their joints
  alone position them — the MMD-idiomatic solution.
- Twist-master insertion, 足D/足先EX and material draw-order re-sorting are out of scope
  (index-renumbering or weight moves); the existing 腕捩/手捩 bones already carry the
  twist weights, and no material needs re-sorting (only 髪_01 uses cutout alpha and it
  already draws after the face).

## Reports

`reports/`: generate / retune / frames / refine / overlaps / physics_validation /
axis_probe_final / weight_invariance / build_summary, the six audit specs, and the
fine-grained reference baseline (`reference_physics_fine.json`) the build gates against.
