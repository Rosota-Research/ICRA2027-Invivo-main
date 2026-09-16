# Hand-held laparoscopic appendectomy demonstrations (ex vivo and in vivo)

Companion data release for the ICRA 2027 submission
*From Instrument-Mounted Demonstrations to In-Vivo Execution: Learning Bimanual Laparoscopic Appendectomy Without Robot-Collected Demonstrations* (under double-anonymous review; author information withheld).

Both training corpora described in the paper are released here in their recorded form, not as the derived training arrays. Every stream carries absolute host timestamps (Unix epoch, UTC) so that the per-channel latency matching of the paper can be reproduced or replaced.

**Status.** The files are hosted on Hugging Face in four public repositories because of their size; this page keeps the same address. Both training corpora used in the paper, the ex-vivo logger and endoscope recordings and the in-vivo episodes, are complete. The RealSense D405 bags are a supplement used only for the simulator reconstruction; they are not needed to reproduce any result in the paper, and their upload (about 487 GB) is still in progress, so those two repositories may appear empty.

| Corpus | Content | Size | Link |
|---|---|---|---|
| Ex-vivo, logger + endoscope | MCU, Blackmagic, Camera, NI6009, labels | ≈ 61 GB | [link](https://huggingface.co/datasets/surgicalrobotics/appendectomy-exvivo) |
| Ex-vivo, RealSense D405 | ROS 2 bags used for the simulator reconstruction | ≈ 487 GB | [part 1](https://huggingface.co/datasets/surgicalrobotics/appendectomy-exvivo-d405-part1) · [part 2](https://huggingface.co/datasets/surgicalrobotics/appendectomy-exvivo-d405-part2) |
| In-vivo, preprocessed episodes | 868 episode folders | ≈ 46 GB | [link](https://huggingface.co/datasets/surgicalrobotics/appendectomy-invivo) |

---

## 1. Ex-vivo corpus (recorded 2026-07-12)

Rabbit cadaver appendix placed in a laparoscopic training phantom; two surgeons (morning and afternoon sessions), two specimens. 585 episodes were recorded; 533 are used for training in the paper after quality filtering. Each episode is one attempt at the appendectomy with hand-held instruments carrying the ex-vivo logger (USB-C, VL53L0X ToF, BNO085 IMU, handle Hall sensor). All files of one episode share the timestamp suffix `<yyyymmdd_hhmmss>`.

```
exvivo/
  MCU/4_left_gripper/episode_<ts>.h5      logger on the left grasper
  MCU/5_right_cutter/episode_<ts>.h5      logger on the right cutter
  Blackmagic/episode_hyperdeck_<ts>.mp4   endoscope video, 4K 60 fps (policy input)
  Blackmagic/episode_hyperdeck_<ts>.h5    recorder start/stop times for the clip
  Camera/episode_cam8_<ts>.mp4 / .h5      overhead webcam and per-frame timestamps
  Realsense/episode_rs_<serial>_<ts>.db3  Intel RealSense D405 RGB-D, ROS 2 bag
  Realsense/episode_rs_<serial>_<ts>.json absolute start/end time of the bag
  NI6009/episode_<ts>.h5                  strain-gauge force channels (not used by the policy)
  labels/260712_step_labels_final.jsonl   hand-corrected phase segments per episode
```

**Logger file (`MCU/*/episode_<ts>.h5`, 60 Hz).**

| Dataset | Shape | Meaning |
|---|---|---|
| `timestamp` | (N,) | host absolute time, Unix epoch UTC |
| `device_ms` | (N,) | device clock, ms |
| `dist` | (N,) | insertion depth from the ToF ray, mm (firmware mapping) |
| `grip` | (N,) | jaw aperture from the Hall sensor, 0 (closed) to 100 (open) |
| `raw_quat` | (N, 4) | IMU rotation vector, xyzw |
| `rcm_quat_rel` | (N, 4) | attitude relative to the reference set at the trocar |
| `rcm_position` | (N, 3) | tip position on the shaft line through the trocar, m |

Attributes record the board name, serial port, record rate and start/end times.

**Force file (`NI6009/episode_<ts>.h5`, 100 Hz).** `forces` (N, 6) after calibration, tare and low-pass filtering, `raw` (N, 6) and `timestamp`. Channel order and units are stored in the attributes (`force_keys`, `force_units`). The policy does not use these channels.

**Phase labels (`labels/260712_step_labels_final.jsonl`).** One JSON object per episode with `episode`, `frames`, `fps` and a `steps` list of `{step, f0, f1, t0, t1}` segments. Frame indices refer to the Blackmagic clip; times are absolute. Vocabulary: `preparing`, `grasp_approach`, `lift`, `cut_approach`, `cut`. These are the labels used for training; earlier automatic versions are not included.

**Known issues.** The last three episodes of the day (`164329`, `164338`, `164347`) have truncated `.h5` files in every stream and are excluded from training. The ex-vivo logger's IMU is mounted facing backwards on the shaft, so its body frame is rotated 180° about the shaft axis with respect to the in-vivo logger.

---

## 2. In-vivo corpus (recorded 2026-08-30)

Four live rabbits in one operating-room session, trocars through the abdominal wall, no phantom. The right instrument is a dissector with an electrosurgical unit; the cut is rehearsed with the pedal in several episodes per animal and performed with energy once. 1,003 episodes were recorded; the 868 released here are those with usable sensor streams (the remaining 135 lack a complete stream); 849 are used for training after the filters described in the paper. Instruments carry the in-vivo logger (battery, BLE, VL53L1X ToF, IMU, handle Hall sensor).

```
invivo/
  ARM_MAPPING.md
  batch_<nnnn>/<yyyymmdd_hhmmss>/
    camera.mp4        endoscope, 1920x1080, about 30 fps
    camera.h5         frame_index, timestamp (absolute time of every frame)
    <board>.h5        one file per logger (two per episode)
    energy.h5         pedal state, 100 Hz, plus exact ON/OFF events
    labels.json       phase marks and segments
    session.json      devices, sides, clock-sync statistics
```

**Logger file (`<board>.h5`).** Groups `imu/{accel, gyro, mag, rv}`, `tof/{range_mm, range_status, t, mcu_us}`, `hall/{adc, t}`, `batt/`, `clock/`, `timesync/`, `stats/`, and per-sample datasets `timestamp`, `device_ms`, `dist`, `grip`, `raw_quat`, `rcm_quat_rel`, `rcm_position`, `pc_recv`. `timestamp` and every `<group>/t` are device times mapped to host time with the session's clock probes (transport delay removed); use them for alignment. `pc_recv` and `*_pc_ts` are the raw host receive times, kept for comparison. Hardware, protocol, packet statistics and the clock fit are stored in the attributes.

**Energy file (`energy.h5`).** `state` (1 = pedal pressed) and `timestamp` at 100 Hz; `event/{timestamp, state, latency_ms}` gives the exact transitions. The pedal mirrors the electrosurgical pedal; it does not measure the generator.

**Labels (`labels.json`).** `marks` (frame, phase) and `steps` `{step, f0, f1, t0, t1}` on the endoscope clip; `reviewed`, `excluded` and `note` record the manual review. Vocabulary: `grasper_approach`, `appendix_lift`, `cut_approach`, `appendix_cut`.

**Known issues.** (i) Boards were exchanged during battery replacement, so board names do not identify the arm; `session.json` assigns sides from the BLE adapter, and that assignment is wrong for a subset of episodes. The corrected assignment used in the paper was derived from the jaw statistics of each episode (the grasper closes during the grasp approach, the dissector during the cut approach); the script will be published with the data. (ii) On one board the Hall channel saturates at full closure in three sessions and reads as open; the value is recoverable from `hall/adc`. (iii) A small number of episodes have no phase labels or end early; they are excluded by the paper's filters.

---

## 3. Coordinate conventions

An instrument through a trocar has four degrees of freedom. The policy observation uses only gravity- and trocar-referenced quantities: shaft roll and depression from the IMU attitude, insertion depth from the ToF ray (shaft length minus board-to-trocar distance), and jaw aperture from the Hall sensor. Magnetometer heading is recorded but not used by the policy.

## 4. Ethics

All animal procedures were approved by an institutional animal care and use committee (approval number withheld for review).

## 5. License and citation

License: to be stated at release (research use). Citation: the ICRA 2027 paper above; a BibTeX entry will be added after the review period.
