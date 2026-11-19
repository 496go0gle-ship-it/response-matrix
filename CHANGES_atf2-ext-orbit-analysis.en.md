# Changes on branch `atf2-ext-orbit-analysis` (response-matrix)

Base: `flight-simulator-framework/response-matrix` `master` at `bd710ba`.
Japanese version: [CHANGES_atf2-ext-orbit-analysis.ja.md](CHANGES_atf2-ext-orbit-analysis.ja.md)

| Commit | Summary |
|---|---|
| `8760772` | Improve data folder and selection handling in the response matrix GUI |
| `0df71b8` | Explain why no orbit data is usable instead of a readback error |
| `fc62455` | Skip quadrupole-mover pairs in the corrector response matrix |
| `48e06ae` | Find quadrupole movers in the sequence for the triangular cut |
| (this commit) | Compute the quadrupole-mover response per setpoint, not per encoder reading |

Files changed: `ComputeResponseMatrix_GUI.py`, `ComputeResponseMatrix_QM_GUI.py`, `response_matrix/ResponseMatrix_DFS_WFS.py`, `response_matrix/UI files/ComputeResponseMatrix_GUI.ui`.

This branch goes together with the branch of the same name in `machine-interfaces`, which makes ATF2 Ext orbits usable again (BPM intensity not recorded) and records BPM names in the beamline sequence.

---

## Data folder selection (`ComputeResponseMatrix_GUI.py`)
- **Default folder.** The folder dialog opens in the first folder that exists among `~/CERN-Flight_Simulator-Data`, the `CERN-Flight_Simulator-Data` folder next to the repositories, the current data folder, and the working directory. Before, it always opened `~/CERN-Flight_Simulator-Data`.
- **Folder without data.** If the chosen folder has no `DATA*.pkl`, a warning lists the subfolders that do contain them (for example the `ATF2_Ext_*_Orbit` folders), and the corrector and BPM lists are cleared. Before, the lists silently kept the devices of the previously loaded folder.
- When a data folder is loaded, its first state is kept as `initial_state`, so **Load…** has a fallback selection when no selection file is chosen. The same is done in `ComputeResponseMatrix_QM_GUI.py`.

## Corrector and BPM selection files (`ComputeResponseMatrix_GUI.py`)
- `correctors.txt` and `bpms.txt` are saved to and loaded from the data folder, where SysID writes them.
- The **Save** and **Clear** buttons for correctors and BPMs are now connected. Before, their handlers existed but were not wired, and the save handlers called `_save_correctors()` / `_save_bpms()`, which `SaveOrLoad` does not define.
- **Load** does nothing while the list is empty (no data folder loaded yet).

## Devices without a beamline position (`response_matrix/ResponseMatrix_DFS_WFS.py`)
- In `_compute_response_matrix()`, BPMs and correctors that are not in the recorded sequence are ignored, with a printed list, instead of failing on `sequence.index()`.
- If none of the selected BPMs is in the recorded sequence, a clear `RuntimeError` is raised.

## Appearance (`response_matrix/UI files/ComputeResponseMatrix_GUI.ui`)
- The device, corrector and BPM group boxes use `palette(mid)` for their border and no fixed light-grey background, so they follow the system light/dark theme.

---

## Verification
- The changed Python files compile (`python -m py_compile`).
- The GUI has not been opened to test these changes interactively.

## Clearer error when no orbit data is usable (`0df71b8`, `response_matrix/ResponseMatrix_DFS_WFS.py`)
- When every DATA pair is skipped because the usable BPMs have no orbit, `_compute_response_matrix()` now raises "No DATA pair has orbit data at the usable BPMs", lists the BPMs that were usable, and points to `python -m machine_interfaces_atf2.repair_ext_sequence`.
- Before, it failed later with "No complete corrector readbacks available for response-matrix.", which pointed at the correctors.
- This happened with the 2026-11-19 ATF2 Ext data: its recorded sequence holds only `MB2X` and `MSF1FF`, and after keeping the BPMs downstream of the selected correctors only `MSF1FF` (reading 0) was left.

## Folders from the combined corrector + mover SysID (`response_matrix/ResponseMatrix_DFS_WFS.py`)
- In corrector mode, `_compute_response_matrix()` skips quadrupole-mover pairs (`DATA_<quad>_<x|y>_...`), which `SysID_corrector_and_mover_GUI.py` (branch `atf2-ext-orbit-analysis` of system-identification) writes next to the corrector pairs.
- This does not change the result: in a mover pair the correctors keep their value and the orbit moves by +/- the same amount, which cancels in the least-squares fit (checked on a simulated folder: identical matrix, difference 6e-16). It avoids reading those files and keeps them out of the "no usable orbit" count.

## Triangular cut of the quadrupole-mover matrix (`response_matrix/ResponseMatrix_DFS_WFS.py`, `_compute_qm_response_matrix`)
- **Problem:** with `triangular=True` (always used by `BBA_QM_GUI.py`), the response at BPMs upstream of a mover is set to zero. The mover's position was looked up by its own name (`QM15FF`), but the recorded sequence holds the BPM on it (`MQM15FF`, ATF2 Ext machine data) or its slices (`QM15FF_1`, RF-Track). Not found, the mover counted as infinitely far downstream, so every BPM was "upstream" and the whole matrix became zero. This affects the existing QM BBA too.
- **Change:** the position is taken from `QM15FF`, `QM15FF_1` or `MQM15FF`, whichever the sequence has. A mover found under none of them keeps all its BPMs (a message is printed) instead of losing its column.
- **Checked** on a simulated folder: each mover column is non-zero only downstream of the mover, with the same maximum as without the cut (e.g. `QM15FF` 0.194, `QD0FF` 0.0020).

## Using ATF2 Ext data recorded before the sequence fix
Rebuild the sequence first (tool in the `machine-interfaces` branch), then select the folders in the `_sequence_fixed` copy:

```bash
python -m machine_interfaces_atf2.repair_ext_sequence ~/CERN-Flight_Simulator-Data
```

With the repaired 2026-11-19 data, the response matrix of `ATF2_Ext_20261119_112700_Orbit` is computed for `ZH1X` and `ZV1X` with 50 BPMs (checked from Python, not in the GUI).

## Quadrupole-mover response per setpoint (this commit, `_compute_qm_response_matrix`)
- The mover change is taken from the setpoint (`xdes`/`ydes`) instead of the encoder (`xact`/`yact`); the encoder is used only when there is no setpoint.
- Why: the BBAs (QM, and correctors + movers) command the setpoints, so the response must be per setpoint. In the 2026-11-19 machine data the x encoders of `QF1FF` and `QD0FF` count opposite to the setpoint (ratio -1.00), so with the encoder those two columns corrected in the wrong direction. For the other 19 movers and y the ratio is 1.00 +/- 0.01, so their results hardly change.
