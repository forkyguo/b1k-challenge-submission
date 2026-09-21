# BEHAVIOR Challenge 2026 Submission

## 1. Evaluation Method

### Start Policy Server (Docker or Local)

```bash
# Docker
docker build -t b1k-policy-server -f Dockerfile .
docker run --gpus all --rm -d --name b1k-policy \
    -p 8000:8000 b1k-policy-server

# Or locally (conda openpi environment)
cd behavior-1k-solution
CUDA_VISIBLE_DEVICES=1 python scripts/serve_b1k.py \
    --task-checkpoint-mapping task_checkpoint_mapping.json \
    --port 8000 \
    policy:checkpoint \
    --policy.config pi_behavior_b1k_fast \
    --policy.dir models/checkpoint_2
```

The server provides websocket policy service on port 8000. Startup takes 1-2 minutes to load checkpoints.

### Run Evaluation

```bash
python -m omnigibson.eval.eval \
    --task-name turning_on_radio \
    --host 127.0.0.1 --port 8000 \
    --mode public_test \
    --instance-indices 0 \
    --num-rollouts 1 \
    --output-dir outputs/b1k_eval \
    --write-video
```

- Wrapper: `omnigibson.eval.wrappers.DefaultWrapper` (default)
- Robot config: `omnigibson/eval/r1pro.yaml` (unmodified, official default)
- Evaluate all 10 public test instances: `--instance-indices 0 1 2 3 4 5 6 7 8 9`

## 2. Policy Details

- Model: PI-BEHAVIOR (`pi_behavior_b1k_fast` configuration)
- Checkpoint routing: Automatic switching by task_id (`task_checkpoint_mapping.json`)
  - turning_on_radio → checkpoint_2
- Input: 3-camera RGB (224x224) + 61-dim proprioception (2026 format, adapted)
- Output: 23-dim action (base 3 + trunk 4 + left_arm 7 + left_gripper 1 + right_arm 7 + right_gripper 1)
- Eval tricks: Correction rules enabled (stage-based action correction)
- VRAM requirement: ~12GB (runs on single 24GB GPU)

## 3. Directory Structure

```
.
├── Dockerfile                 # Policy server Docker image definition
├── build_docker.sh            # Docker build script
├── README.md                  # This file
├── r1pro.yaml                 # Robot configuration (official default, unmodified)
├── eval_b1k_wrapper.py        # Policy wrapper code (challenge-track observation compliant)
├── json/                      # Evaluation result JSONs
│   └── turning_on_radio_301_0.json
└── videos/                    # Evaluation videos
    └── turning_on_radio_301_0.mp4
```

## 4. Key Code Modification

The only code modification to adapt the solution to the 2026 environment is in `extract_state_from_proprio` function (`b1k/policies/b1k_policy.py`):

- 2026 OmniGibson proprio vector is 61-dim (concatenated per r1pro.yaml proprio_obs order)
- Original solution expected 256-dim
- Modified to support both formats (auto-detected by dimension)

61-dim index mapping:
| Field | Index |
|-------|-------|
| base_qvel | [0:3] |
| arm_left_qpos | [3:10] |
| eef_left_pos/quat | [17:24] |
| gripper_left_qpos | [24:26] |
| arm_right_qpos | [28:35] |
| eef_right_pos/quat | [42:49] |
| gripper_right_qpos | [49:51] |
| trunk_qpos | [53:57] |

## 5. Docker Image Information

- Base image: nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04
- Python: 3.11 (conda openpi environment)
- Port: 8000 (websocket)
- Model files: Downloaded from HuggingFace during build (~26GB total for all 4 checkpoints)
- Build time: ~30 minutes (dependency download), image size ~9GB compressed / ~28GB uncompressed
