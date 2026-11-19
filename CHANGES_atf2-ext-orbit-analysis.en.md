# Changes on branch `atf2-ext-orbit-analysis` (response-matrix)

Base: `flight-simulator-framework/response-matrix` `master` at `bd710ba`.
Japanese version: [CHANGES_atf2-ext-orbit-analysis.ja.md](CHANGES_atf2-ext-orbit-analysis.ja.md)

| Commit | Summary |
|---|---|
| `8760772` | Improve data folder and selection handling in the response matrix GUI |
| `0df71b8` | Explain why no orbit data is usable instead of a readback error |

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

## Using ATF2 Ext data recorded before the sequence fix
Rebuild the sequence first (tool in the `machine-interfaces` branch), then select the folders in the `_sequence_fixed` copy:

```bash
python -m machine_interfaces_atf2.repair_ext_sequence ~/CERN-Flight_Simulator-Data
```

With the repaired 2026-11-19 data, the response matrix of `ATF2_Ext_20261119_112700_Orbit` is computed for `ZH1X` and `ZV1X` with 50 BPMs (checked from Python, not in the GUI).
