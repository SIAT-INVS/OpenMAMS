<div align="center">

# OpenMAMS: Open-Sourced Multi-Agent Memory System

### Memory in the Sky: Low-Altitude Question Answering with Multi-Agent Memory Aggregation

Chengyang Li<sup>1</sup>, Yujie Wan<sup>2</sup>, Shuai Wang<sup>3</sup>, Kejiang Ye<sup>3</sup>, Weijie Yuan<sup>2</sup>, Boyu Zhou<sup>2</sup>, Yik-Chung Wu<sup>1</sup>, Chengzhong Xu<sup>4</sup>, and Huseyin Arslan<sup>5</sup>

<sup>1</sup>The University of Hong Kong · <sup>2</sup>Southern University of Science and Technology<br>
<sup>3</sup>Shenzhen Institutes of Advanced Technology, Chinese Academy of Sciences<br>
<sup>4</sup>University of Macau · <sup>5</sup>Istanbul Medipol University

[Overview](https://siat-invs.github.io/OpenMAMS-project/#overview) · [Architecture](https://siat-invs.github.io/OpenMAMS-project/#architecture) · [CARLA Simulation](https://siat-invs.github.io/OpenMAMS-project/#carla-simulation) · [Real-World Experiments](https://siat-invs.github.io/OpenMAMS-project/#real-world-experiments) · [MemNTN](https://siat-invs.github.io/OpenMAMS-project/#memory-native-non-terrestrial-networks) · [Acknowledgements](#acknowledgements) · [Citation](#citation)

[Project Website](https://siat-invs.github.io/OpenMAMS-project/) · [arXiv:2609.35431](https://arxiv.org/abs/2609.35431) · [Paper PDF](https://arxiv.org/pdf/2609.35431)

</div>

<p align="center">
  <a href="https://siat-invs.github.io/OpenMAMS-project/assets/pmas-semantic-map.mp4">
    <img src="https://raw.githubusercontent.com/SIAT-INVS/OpenMAMS/3642621d91033dad838365b642a05a5b9e735e8c/assets/pmas-semantic-map.webp" width="100%" alt="Panoramic multi-agent system: rotating COLMAP point cloud, three UAV trajectories, and semantic observation anchors.">
  </a>
</p>

<p align="center">
  <a href="https://siat-invs.github.io/OpenMAMS-project/assets/pmas-semantic-map.mp4">▶ Watch the full video</a>
</p>

> OpenMAMS aggregates distributed UAV memories for long-horizon question answering. Our memory-centric framework measures what each candidate memory adds, then jointly selects UAVs and allocates transmit power under communication constraints.

## Code

This repository contains the runnable code and setup instructions. Figures, videos,
experimental results, and system demonstrations are on the [project website](https://siat-invs.github.io/OpenMAMS-project/).

| Module | Contents | Guide |
| --- | --- | --- |
| `uav_data_recorder/` | Synchronized CARLA RGB images, camera poses, routes, and object ground truth | [UAV data recorder](#uav-data-recorder) |
| `ntn/` | Satellite geometry, uplink/downlink scheduling, and FIFO image-delivery replay | [Satellite backhaul](#satellite-backhaul) |

The modules install independently and connect through recording files. This release
contains the recorder and satellite backhaul tools; the paper's memory valuation,
GAE, and MemCen optimization implementations are not included.

## Installation

Use **Linux and Python 3.11**. For data collection, install the CARLA server and
Town04/Town05 maps separately, with matching **CARLA 0.9.16** client/server versions.
NumPy and Pillow support capture; NumPy and pandas support the NTN module.

Run the following commands from the repository root:

```bash
git clone https://github.com/SIAT-INVS/OpenMAMS.git
cd OpenMAMS
python3.11 -m venv .venv
source .venv/bin/activate
pip install -e './uav_data_recorder[carla]' -e ./ntn
```

If you only need one module, install just its path. CARLA is not required for
satellite backhaul simulation or replaying an existing recording.
All commands below also run from the repository root, with the environment activated.

## UAV data recorder

Use `openmams-record` for perspective RGB images, `openmams-record-panorama` for
360° images and videos, and `openmams-panorama-demo` for captioned FPV demos.
[Single-frame samples](samples/README.md) include three panoramic and ten
perspective UAV views with poses and capture parameters (~9 MB).

Start CARLA in a separate terminal:

```bash
/path/to/CARLA/CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=2000
```

Collect four UAV views in Town04 for 30 seconds at 10 FPS per UAV:

```bash
openmams-record --map Town04 --uavs 4 --seconds 30 --fps 10 \
  --speed 4.5 --altitude 17.5 --pitch -45 --objects 10 --output runs/town04
```

This produces 1,200 images. The included Town05 loop supports ten cameras:

```bash
openmams-record --map Town05 --route uav_data_recorder/routes/town05_loop.json \
  --uavs 10 --seconds 15 --fps 10 --objects 10 --output runs/town05
```

This produces 1,500 images. FPS is measured **per UAV**; lower `--fps` or image
resolution if CARLA rendering becomes unstable (for example, use `--fps 1` for K=10).

### Routes and objects

Automatic routes follow roads from separated starting points. `--seed` controls
route choices and object placement; `--spawns 12 35 80 110` selects map-specific
starting points. Automatic routes do not guarantee a loop.

`--route uav_data_recorder/routes/town05_loop.json` uses the included **1,005.44 m
closed loop**, with UAVs starting at equal distances along it. Use either `--route`
or `--spawns`.

Default speed is **4.5 m/s**, altitude **17.5 m above the road**, and camera pitch
**−45°**. The recorder uses virtual cameras without flight dynamics.
`--objects 10` places five colored cars, a fire truck, motorcycle, bus, taxi,
and traffic cone along the routes, then saves their world coordinates for evaluation.
Use a dedicated CARLA instance and a new output directory.

### Recording output

```text
runs/town04/
  capture.json                 # Parameters and completion status
  frames.jsonl                 # Frame IDs, UAV IDs, timestamps, and camera poses
  images/uav_01/000000.jpg
  images/uav_02/000000.jpg
  ...
  objects.json                 # Object ground truth when --objects 10 is used
```

Each frame contains `id, uav_id, time_s, timestamp, world_frame, x, y, z, yaw, pitch, image`.
Coordinates use CARLA world meters; yaw and pitch use degrees. `time_s` starts at
zero for the recording, while `timestamp` is CARLA elapsed simulation time.
Image paths are relative to the recording directory. A finished recording has
`completed: true` in `capture.json`. Pass this output directory to [replay](#replay-a-recording).

### 360° panorama dataset capture (`panorama.py`)

Six synchronized 90° RGB cameras per UAV produce **2:1 equirectangular panoramas**.
The recorder saves every frame as PNG and a panorama MP4 per UAV, without captions
or layout overlays.

Install the optional dependencies and start a dedicated CARLA 0.9.16 server:

```bash
pip install -e './uav_data_recorder[carla,panorama]'
/path/to/CARLA/CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=2010
```

Capture **three UAVs, 40 simulation seconds, 15 FPS** on the Town05 loop:

```bash
openmams-record-panorama --port 2010 --map Town05 \
  --route uav_data_recorder/routes/town05_loop.json \
  --uavs 3 --seconds 40 --fps 15 --speed 4.5 --altitude 12 \
  --face-size 768 --pano-width 2048 --output runs/town05_panorama
```

UAVs start at equal distances along the loop, 12 m above the road. All camera
faces share manual exposure; subtle cubemap seams may remain. This mode uses
existing map objects and requires a fresh output directory.

Output contains:

```text
runs/town05_panorama/
  capture.json                     # Parameters and completion status
  route.json                       # Source route
  frames.jsonl                     # Poses, timestamps, frame IDs, and image/video paths
  uav1_panorama.mp4                 # 2048×1024, 40 s, 15 FPS
  uav2_panorama.mp4
  uav3_panorama.mp4
  panorama_frames/
    uav1/000000.png ... 000599.png
    uav2/000000.png ... 000599.png
    uav3/000000.png ... 000599.png
  previews/
```

The example writes **600 PNGs per UAV (1800 total)** plus three videos. PNGs
preserve the stitched pixels losslessly; H.264 MP4s are lossy playback copies.
The original six-face sequences are not saved. All UAVs share world frame IDs
and timestamps. Panorama output uses schema v2 and is not compatible with the
ordinary RGB [satellite replay](#replay-a-recording) input format.

### Captioned presentation video (`demo.py`)

`demo.py` derives **90° rectified FPV** views from the panorama PNGs and adds
offline Qwen3-VL 8B captions. Install and start Ollama, then run:

```bash
ollama pull qwen3-vl:8b-instruct
openmams-panorama-demo --capture runs/town05_panorama \
  --playback-speed 2 --interval 8 --fpv-pitch -20
```

For the 40-second capture above, output is
`demo/three_uav_carla_panorama_fpv_captioned_1080p.mp4`: **1920×1080, 20 s, 30 FPS**.
Three columns show UAV1–UAV3, with panorama above FPV; captions update every
4 playback seconds. Sampled FPV images, model responses, and previews are saved
in `demo/` without modifying the source capture.

Use `--output` in a new directory when changing FPV/caption settings. `--font`,
`--ollama`, and `--ffmpeg` override local defaults. For legacy schema-v1 captures,
omit `--interval` and `--fpv-pitch` to use the recorded settings.

CPU-only projection/synchronization tests (CARLA-basis tests skip if unavailable):

```bash
pip install pytest
python -m pytest uav_data_recorder/tests -q
```

## Satellite backhaul

Run the included 400-satellite example:

```bash
bash ntn/scripts/run_backhaul.sh runs/backhaul_400
```

Results are saved under `uplink/`, `downlink/`, and `effective/`. The `uid=1` rows in
`effective/mac_user_qos.csv` give backhaul rates for Proportional Fair and Max-C/I.
Backhaul capacity is the smaller endpoint mean rate and is held constant during
replay; ISL capacity is assumed sufficient.

### Generate a constellation

The included snapshot needs no topology generator. To generate a new constellation:

```bash
bash ntn/scripts/setup_dependencies.sh --leopath
openmams-ntn topology --planes 20 --sats-per-plane 20 --output runs/topology_400
bash ntn/scripts/run_backhaul.sh runs/new_backhaul runs/topology_400
```

Use `--minute` to select a snapshot time. Generated files include `constellation.tle`,
`hong_kong.json`, `istanbul.json`, and `topology.json`. To fetch the pinned LEOPath and
OpenNTN repositories without installing them, run:

```bash
bash ntn/scripts/setup_dependencies.sh --fetch
```

Fetched source stays in ignored `ntn/third_party/`; the full OpenNTN PHY is optional.

### Replay a recording

```bash
openmams-ntn replay --data runs/town04 \
  --backhaul runs/backhaul_400/effective --scheduler 'Proportional Fair' \
  --deadline-s 30 --output runs/received
```

Payload size defaults to the image file size. Use `--frame-bytes` for a fixed size,
or `--propagation-ms` to add propagation delay. The deadline is measured from the
start of the recording. Delivered images and original poses are saved in `images/`
and `frames.jsonl`; `delivery.json` summarizes delivery, and `transmissions.jsonl`
contains per-frame results. Use a new output directory for each run.

| Command | Purpose |
| --- | --- |
| `openmams-ntn topology` | Generate satellite geometry |
| `openmams-ntn endpoint` | Simulate an uplink or downlink endpoint |
| `openmams-ntn combine` | Combine endpoint capacities |
| `openmams-ntn replay` | Replay image transmission |

Append `--help` to any command for its options, including `openmams-record --help`.

### Tests

```bash
pip install -e './ntn[test]'
python -m pytest ntn/tests -q
```

## Sources and licenses

- Recorder: [MIT license](uav_data_recorder/LICENSE).
- [LEOPath](https://github.com/Fundacio-i2CAT/LEOPath) (AGPL-3.0) and
  [OpenNTN](https://github.com/ant-uni-bremen/OpenNTN) (MIT). Fetched repositories
  retain their original licenses and copyright notices.
- Building geometry: © OpenStreetMap contributors,
  [ODbL](https://www.openstreetmap.org/copyright). Some heights use default estimates.

## Acknowledgements

We thank the authors and contributors of:

- [ReMEmbR](https://github.com/NVIDIA-AI-IOT/remembr), for components used in our research QA pipeline.
- [Qwen3-VL](https://github.com/QwenLM/Qwen3-VL) and [Qwen3](https://github.com/QwenLM/Qwen3), for models used in our captioning and question-answering research.
- [CARLA](https://github.com/carla-simulator/carla) and [CARLA Dataset Tools](https://github.com/KevinLADLee/carla_dataset_tools), for simulation and data-collection tools.
- [LEOPath](https://github.com/Fundacio-i2CAT/LEOPath) and [OpenNTN](https://github.com/ant-uni-bremen/OpenNTN), for satellite-topology and optional channel-model tools.

## Citation

If you find this work useful, please cite:

### Journal version

```bibtex
@misc{li2026memoryinthesky,
  title  = {Memory in the Sky: Low-Altitude Question Answering with Multi-Agent Memory Aggregation},
  author = {Li, Chengyang and Wan, Yujie and Wang, Shuai and Ye, Kejiang and Yuan, Weijie and Zhou, Boyu and Wu, Yik-Chung and Xu, Chengzhong and Arslan, Huseyin},
  year   = {2026},
  eprint = {2609.35431},
  archivePrefix = {arXiv},
  primaryClass = {cs.RO},
  url    = {https://arxiv.org/abs/2609.35431}
}
```

### Conference version

```bibtex
@inproceedings{li2026memory,
  title={Memory centric power allocation for multi-agent embodied question answering},
  author={C. Li and S. Wang and K. Ye and W. Yuan and B. Zhou and Y.-C. Wu and C. Xu and H. Arslan},
  booktitle={Proc. GLOBECOM},
  year={2026}
}
```

### Magazine version

```bibtex
@article{li2026memntn,
  title={Memory-Native Non-Terrestrial Networks for Embodied Intelligence},
  author={Li, Chengyang and Wang, Yikun and He, Jiahui and Wan, Yujie and Wang, Shuai and Wu, Yuan and Wu, Yik-Chung and Xu, Chengzhong and Arslan, Huseyin},
  journal={IEEE Communications Standards Magazine},
  year={2026}
}
```

## Contact

- **Chengyang Li**: [KevinLADLee](https://github.com/KevinLADLee)
- **Shuai Wang**: [bearswang](https://github.com/bearswang)
