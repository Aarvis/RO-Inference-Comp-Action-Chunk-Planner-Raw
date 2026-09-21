# Training-to-Inference Configuration Alignment

Use this checklist each time a new `pi05_origami_comp_action_chunk` or
`pi05_origami_comp_action_chunk_phase2` model is trained. The inference
package is generic; its bundle must be rebuilt with the exact values used by
the checkpoint being deployed.

## The one rule

Do not edit a Docker bundle by hand after it has been built. First edit the
source runtime configuration in `configs/`, validate it, then run
`scripts/prepare_comp_action_chunk_bundle.py` to regenerate
`model_bundle/`. Finally run `scripts/verify_comp_action_chunk_bundle.py`.

The only intentional modes are:

```text
planner_enabled: true
  OOI target present -> posterior branch, raw planner features
  OOI target absent  -> prior branch, raw planner features
  gamma continuity   -> disabled; no continuity-state update

planner_enabled: false
  no DINO / OOI / checkpoint planner is loaded
  OpenPI receives zero planner arrays with planner_available=false
```

`prior branch` above is the non-OOI planner model branch. It is not the gamma
continuity prior.

## Files to edit for a new trained checkpoint

Start with a copy of `configs/dataset_replay.yaml`. For an entirely
planner-free deployment, start with `configs/dataset_replay_planner_off.yaml`.

| Training property | Change in this project | Must equal |
| --- | --- | --- |
| OpenPI config name | `OpenPI_Module/configs/openpi_comp_action_chunk_runtime.yaml` → `openpi.config_name` | the `TrainConfig.name` used for the checkpoint: base or Phase 2 |
| OpenPI checkpoint | same file → `paths.checkpoint_dir` | the complete saved training-step directory containing `params/` **and** `assets/` |
| Model architecture | same file → `openpi.model` | the exact model config that trained the checkpoint |
| Action horizon | `openpi.model.action_horizon`, `server.action_horizon`, `dataset_replay.action_horizon` | training `model.action_horizon` |
| Action stride | `openpi.data.action_chunk_stride`, `server.action_chunk_stride`, `dataset_replay.action_chunk_stride` | training `data.action_chunk_stride` |
| OpenPI normalization | copied automatically from the checkpoint `assets/<asset_id>/norm_stats.json` | the training checkpoint's assets |
| OpenPI source | `vendor/openpi/src/` | exact OpenPI source revision used to train, including the requested config registration |
| Planner settings, when enabled | `configs/dataset_replay.yaml` paths and runtime section | planner checkpoint/config/manifest used by the selected VLA feature export |
| Planner feature contract | top-level `runtime` section | `planner_feature_mode: selected_branch_raw_no_continuity`, `planner_value_variant: raw` |

The bundle verifier rejects an OpenPI config name that is absent from
`vendor/openpi/src/openpi/training/config.py`. This prevents accidentally
deploying Phase 2 weights with an old vendored OpenPI source.

## Base and Phase 2 bundles

One `model_bundle/` represents one deployable OpenPI checkpoint. Keep separate
bundle directories or separate copies of this project for the base and Phase 2
weights. They may use different horizons and strides, so they must not share a
single active `bundle.yaml`.

For either model, set `openpi.config_name` to one of:

```yaml
# Base
config_name: pi05_origami_comp_action_chunk

# Phase 2
config_name: pi05_origami_comp_action_chunk_phase2
```

Then copy the full saved OpenPI checkpoint directory, not only its `params/`
subdirectory. `assets/` is required for normalization at inference.

## Action stride is deployment behavior

For horizon `H` and stride `S`, output action `j` represents the target at
the recorded trajectory index `t + j*S`. For example, `H=15, S=2` has targets
at `t, t+2, ..., t+28`.

The server publishes `action_chunk_stride` in its metadata. The robot-side
executor must intentionally choose how to consume those actions:

- execute consecutively to realize the trained speed-up/compression; or
- hold/wait between them to preserve recorded timing.

Changing only the server value is unsafe. It must match the OpenPI model and
the training manifest/shards.

## Rebuild and validate

```bash
cd /workspace/RO-Inference-Comp-Action-Chunk-Planner-Raw

python ./scripts/prepare_comp_action_chunk_bundle.py \
  --source-config ./configs/dataset_replay.yaml \
  --bundle-root ./model_bundle \
  --copy-mode copy \
  --include-openpi-source \
  --overwrite

python ./scripts/verify_comp_action_chunk_bundle.py --bundle-root ./model_bundle
```

Use `--copy-mode hardlink` only for local testing on one filesystem. Use
`copy` for a self-contained Docker bundle.

Before Docker, run a one-episode replay with the same bundle. Test both
`planner_enabled: true` and the planner-off profile if the model was trained
with planner dropout.
