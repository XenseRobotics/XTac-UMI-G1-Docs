# 4. Calibration & self-check

This chapter covers the two **one-off** jobs: calibrating the gripper, and the tracker
self-check. Whether the whole chain is present is confirmed with the
[preview](05-data-collection.md#preview) just before recording.

## 4.1 Gripper calibration (zero + travel span) {#41}

### 4.1.1 When you need it

Two situations call for it — the program catches one, the other is only visible to you:

| Situation | How it shows | Who catches it |
|---|---|---|
| **Never calibrated** | The collection program **refuses to connect**, with the calibration command in the error | The program; you cannot miss it |
| **Calibrated, but the stored span no longer matches** | It connects, but a jaw opened to its mechanical limit reads clearly short of `1.0` (around `0.8`, say) | **Only you, in the preview** — see [4.1.3](#413) |

The second usually follows **refitting the encoder or moving a limit stop**: the program knows a
value was stored, not whether it is still right, so it does not raise. Outside those two there is
nothing to redo — the values live in MCU flash and survive power cycles and host changes, so
**once per unit is enough**. `gripper.pos` in the dataset is a **normalised opening**, `0.0` fully
closed and `1.0` fully open, and those two endpoints are the two numbers calibration writes to
flash:

| Endpoint | Source | Written by |
| --- | --- | --- |
| `0.0` closed | encoder zero | calibration step 1 (fully closed) |
| `1.0` open | that gripper's travel span | calibration step 2 (open to the mechanical limit, firmware >= V2.1) |

**What happens without the travel span.** A leader fails at connect, and the error names the
command that fixes it:

```text
This leader gripper has no encoder-max calibration, so its jaw travel is unknown
and gripper.pos cannot be computed (...).

Calibrate it once, then re-run:

    python third_party/taccap-gripper/python/examples/calibrate.py <left|right>
```

!!! danger "Calibrating one side of a bimanual rig is worse than calibrating neither"
    With neither calibrated the two scales at least agree. Calibrating one leaves
    `left_gripper.pos` and `right_gripper.pos` on **different scales** — the same physical grip
    reads differently on each side, and nothing in the data shows it. **Do both or neither.**

### 4.1.2 How to calibrate

Pick the gripper by side and run it once per unit — **both** of them:

```bash
python third_party/taccap-gripper/python/examples/calibrate.py left
python third_party/taccap-gripper/python/examples/calibrate.py right
```

Side is read from the firmware-burned SN, the same rule collection applies to
`left_gripper.pos` — so `calibrate.py left` is guaranteed to be the gripper `left` means
everywhere else. The script prints the firmware SN it resolved along with
**every gripper the scan saw**, so a wrong pick is visible before anything reaches flash. To pin a
unit explicitly, pass its firmware SN instead (`calibrate.py TCGU01A28Z0024m` — that SN is an
example; use your own).

!!! danger "`needs command set >= V2.1` → flash first, then come back and calibrate"
    The script checks the firmware version **before** touching anything. If it is too old it
    **exits without changing a thing**, printing what this unit reports and the command to run:

    ```text
    ✗ encoder-max calibration needs command set >= V2.1 (leader >= 1.2.0); this gripper reports 1.1.0.
      Nothing was changed. Flash it first: ...
    ```

    The procedure is at **[Firmware OTA upgrade](versions.md#ota)** — note that the image is
    chosen **by role, not by which hand it is on**, and that the **SDK must be updated before the
    firmware**. Re-run the command in this section once the flash is done.

One command, two steps, following the prompts:

1. **Hold the gripper fully closed** → Enter. Latched as the encoder zero, then re-read to verify
   the residual (tolerance ±0.01 rad).
2. **Open fully (against the mechanical limit)** → Enter. That angle is sampled and written
   **straight** to MCU flash as the encoder max (there is no second confirmation — hold it open
   before pressing Enter), followed by a 10 Hz live readout so you can eyeball the result.

Output looks like:

```text
================================================================
  TacCap leader-gripper encoder calibration
================================================================
  requested    : left  (resolved by side)
  firmware SN  : TCGU01A28Z0031m
  side         : Left
  mcu serial   : 5C96089694
  mcu device   : /dev/serial/by-id/usb-1a86_USB_Dual_Serial_5C96089694-if02
  visible      : TCGU01A28Z0032m (Right), TCGU01A28Z0031m (Left)

Step 1/2: hold the gripper FULLY CLOSED.
  → press [Enter] when held closed:
  post-latch reading: raw=+0.0058 rad (+0.33°)   cooked=+0.0058
  ✓ zero latched OK (|raw post-zero| ≤ 0.010 rad)

Step 2/2: open the gripper to its MECHANICAL LIMIT.
  → press [Enter] when fully open:
  fully-open reading: +1.1486 rad  (+65.81°)
  ✓ stored: max_rad = 1.1486 rad (65.81°)
```

A unit that was calibrated before also gets an `existing span: … — will be overwritten` line in
the header.

!!! warning "Hold it at the opening before pressing Enter"
    The firmware latches the raw count at the instant it receives the command. The jaw must
    **already** be at the target opening (fully closed for step 1, fully open for step 2) when you
    press Enter — moving it afterwards wastes the calibration.

!!! tip "Closed is always 0"
    The zero lives in firmware; there is **no `gripper_closed_rad` config**. `position_rad` keeps
    reporting raw radians — normalisation adds a `position` field rather than changing what was
    already there.

### 4.1.3 Confirm it took effect {#413}

**One: the connect log.** When the calibration is in effect, each side prints:

```text
[left]  Jaw normalised by the firmware's encoder-max calibration
```

If that line is missing there is nothing more to check — **an uncalibrated leader does not connect
at all**; the program exits with the calibration command in the error.

**Two: the curve in Rerun.** Run with `--display_data=true` and find `gripper.pos` in the scalar
panel:

| Action | Expected |
| --- | --- |
| Fully open | reaches **1.0** |
| Fully closed | drops to **0.0** |

Connecting at all means the travel span is in effect, so this step checks whether the stored
value is still *accurate*.

!!! warning "Clearly short of 1.0 wide open → recalibrate that unit"
    If it connects but a fully open jaw only reaches around `0.8`, the span stored in flash no
    longer matches this gripper's real travel — usually after the encoder was refitted or a limit
    stop moved. The program does **not** raise for this: it knows a value was stored, not whether
    that value is still right, so this one is only visible to you in the preview.

    Just run the calibration again; the command is identical to the first time.

### 4.1.4 Scope

- **Manual calibration is leader-only.** The two-step procedure is a leader capability; a
  follower rejects the command — it relies on the firmware's **power-on auto-calibration**
  instead (see the note below). During collection a follower's `gripper.pos` is still normalised
  by `gripper_open_rad`, unchanged from the older behaviour.
- **Firmware command set >= V2.1 required** — that is build leader **>= 1.2.0** / follower **>= 1.1.0**; see
  [the three version numbers](versions.md#v21). Older firmware does not support it:
  `calibrate.py` **exits without changing anything**, and at collection time a leader fails at
  connect with a pointer to the OTA update. Any gripper
  below V2.1 **must be upgraded**; the images ship with the SDK →
  [Firmware OTA upgrade](versions.md#ota). **Update the SDK before the firmware** — the other
  order runs into an old bug where a failed update reported success.
- **Calibration is one-off.** The values live in MCU flash: they survive power cycles and moving
  to another host. Only redo it after removing or refitting the encoder, changing the mechanical
  limit, or erasing the firmware.

!!! note "A follower's power-on auto-calibration does not replace this"
    **Followers** have supported power-on auto-calibration since V1.9: on power-up they close to
    stall for the zero and open to stall for the travel span. **Leaders do not have it**, and
    collection uses leaders — so the manual calibration in this section is still required.

## 4.2 Pico4 Ultra Enterprise tracker self-check

!!! note "There is no tracker calibration step — this command only reads"
    The tracker needs no calibration — the mount transform is **built in** (below) and the side is
    matched from the SN. This command **writes nothing**: it prints the pose for you to look at.
    This section is purely about confirming the chain is live and the hardware is mounted correctly.

```bash
python -m lerobot.robots.taccap_gripper.check_tracker
# Pin a specific tracker SN (format in 3.3, e.g. PC2310MLL3200496G):
python -m lerobot.robots.taccap_gripper.check_tracker <tracker SN>
# Apply that side's built-in tracker->TCP mount transform:
python -m lerobot.robots.taccap_gripper.check_tracker --side right
```

Prints `raw` (the tracker's own pose) and `ee` (the TCP after the rigid mount transform) at 10 Hz.
Wave the gripper: `raw xyz` should move smoothly and the SN should match what you expect.

!!! note "The mount transform is built in — you do not measure it"
    The tracker is bolted to the gripper, so what it reports is the **tracker's** pose, not the TCP
    we want to record. The rigid offset between them is built into
    `ee_transform.tracker_to_tcp` (measured off the CAD assembly), **with each side measured
    separately** — the two are close to mirror images but not exactly (0.03° apart in rotation,
    1.27 mm in translation), so the left value is not the right one mirrored.

    `--side` picks which built-in value to apply; **without `--side` the transform is identity**
    and `ee` simply follows `raw`. Override only if you need to (a re-machined mount, say) via
    `--robot.tracker_to_ee_pos` / `--robot.tracker_to_ee_quat`; the two are independent, so you can
    pin just the translation and keep the built-in rotation.

!!! tip "Pivot check: verify the mount transform with no extra hardware"
    Rest the gripper's **two-finger midpoint** on a fixed point and, holding the handle, sweep
    through as many orientations as it allows. **`ee xyz` should stay put while `raw xyz` swings
    widely** — that is the whole test, and whatever drift you see is the transform's error.
    **Test both sides**; a left value mirrored the wrong way shows up as `ee` moving about twice
    as far as it should.

If the quaternion flips hemisphere (sign jumps), the pose path already applies a continuity
fix — **please file a bug if you still see jumps**.

Once calibration and the self-checks pass, you are ready to collect.

Next → [5. Data collection](05-data-collection.md)
