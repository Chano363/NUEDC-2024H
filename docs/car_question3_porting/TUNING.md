# H3 tuning guide

The state machine is in `src/car/empty/App/h3_task.c`; tune values in
`src/car/empty/App/h3_config.h` first.

## Control ownership

- `STRAIGHT_AC` / `STRAIGHT_BD`: `heading_ctrl` holds the gyro heading.
- `ARC_CB` / `ARC_DA`: `tracking.c` reads the eight grayscale sensors and
  controls the semicircle.
- `TURN_BD` / `ARC_ALIGN`: `heading_ctrl` requests an angle and `turn_ctrl`
  runs the angle PD loop.

## Parameters to adjust

| Symptom | Parameter | Direction |
|---|---|---|
| Straight speed too high/low | `H3_CFG_STRAIGHT_SPEED` | decrease/increase |
| Semicircle speed too high/low | `g_base_speed` in `tracking.c` | decrease/increase |
| Straight oscillates | `H3_CFG_STRAIGHT_MAX_CORRECTION` or heading PD in `turn_ctrl.c` | reduce first |
| Arc exits too early | `H3_CFG_MODE_B_ARC_HEADING_DEG`, `H3_CFG_ARC_CENTER_STABLE_TICKS` | increase |
| Arc exits too late while line is lost | `H3_CFG_MODE_B_ARC_LOST_EXIT_TICKS` | decrease carefully |
| Car body has not cleared the arc after line loss | `H3_CFG_ARC_EXIT_STRAIGHT_MS` | increase |
| Car overshoots after exiting the arc | `H3_CFG_ARC_EXIT_STRAIGHT_MS` | decrease |
| Mode C lap count | `H3_CFG_MODE_C_LAPS` | normally 4 |
| In-place alignment does not move | `H3_CFG_ALIGN_MIN_PWM` | increase |
| Alignment overshoots | `TURN_KP`/`TURN_KD` in `turn_ctrl.c` | reduce KP or increase KD |

For mode C, `H3_CFG_MODE_C_INITIAL_HEADING_OFFSET_DEG` defines the A→C
direction after gyro zeroing. Set it to `0` when the vehicle is zeroed while
pointing along AC. If zeroing along AB, set the signed diagonal offset (about
`±52.73` degrees, according to the measured turn direction). The firmware
cannot infer which physical line was used for zeroing from gyro data alone.

Tune one parameter at a time. Verify OLED state code: `C`/`D` are straight,
`B`/`A` are arc tracking, and `R` is exit alignment.
