# Changes on branch `atf2-ext-orbit-analysis` (response-matrix)

Base: `flight-simulator-framework/response-matrix` `master` at `bd710ba`.
Japanese version: [CHANGES_atf2-ext-orbit-analysis.ja.md](CHANGES_atf2-ext-orbit-analysis.ja.md)

| Commit | Summary |
|---|---|
| `8760772` | Improve data folder and selection handling in the response matrix GUI |

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

## Known issue
- With the 2026-11-19 ATF2 Ext data, the response matrix still fails with "No complete corrector readbacks available for response-matrix." The recorded sequence of that data contains only two BPMs (`MB2X`, `MSF1FF`), both reading 0, so every selected BPM is ignored. The real cause is the missing BPM names in the sequence, which the `machine-interfaces` branch fixes for new recordings; the error message itself is misleading in this case.
