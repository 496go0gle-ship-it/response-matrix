# ブランチ `atf2-ext-orbit-analysis` の変更内容（response-matrix）

ベース：`flight-simulator-framework/response-matrix` の `master`（`bd710ba`）
英語版：[CHANGES_atf2-ext-orbit-analysis.en.md](CHANGES_atf2-ext-orbit-analysis.en.md)

| コミット | 概要 |
|---|---|
| `8760772` | 応答行列 GUI のデータフォルダと選択ファイルの扱いを改善 |
| `0df71b8` | 使える軌道データがないとき、補正電磁石のエラーではなく原因を表示する |

変更したファイル：`ComputeResponseMatrix_GUI.py`、`ComputeResponseMatrix_QM_GUI.py`、`response_matrix/ResponseMatrix_DFS_WFS.py`、`response_matrix/UI files/ComputeResponseMatrix_GUI.ui`

このブランチは、`machine-interfaces` の同じ名前のブランチと組み合わせて使います。あちらのブランチで、ATF2 Ext の軌道が再び使えるようになり（BPM 強度が記録されない問題）、ビームラインの並びに BPM 名が記録されるようになります。

---

## データフォルダの選択（`ComputeResponseMatrix_GUI.py`）
- **最初に開くフォルダ：** フォルダ選択画面は、`~/CERN-Flight_Simulator-Data`、リポジトリと同じ階層の `CERN-Flight_Simulator-Data`、現在のデータフォルダ、作業ディレクトリのうち、最初に存在するものを開きます。これまでは常に `~/CERN-Flight_Simulator-Data` を開いていました。
- **データのないフォルダを選んだとき：** 選んだフォルダに `DATA*.pkl` がなければ、データを含むサブフォルダ（`ATF2_Ext_*_Orbit` など）の一覧を警告で表示し、補正電磁石と BPM の一覧を空にします。これまでは、メッセージも出ずに、前に読み込んだフォルダの機器が一覧に残ったままでした。
- データフォルダを読み込んだとき、最初の状態を `initial_state` として保持します。これにより、選択ファイルを選ばなかったときも **Load…** が既定の選択を使えます。`ComputeResponseMatrix_QM_GUI.py` でも同様です。

## 補正電磁石と BPM の選択ファイル（`ComputeResponseMatrix_GUI.py`）
- `correctors.txt` と `bpms.txt` を、SysID が書き出すのと同じデータフォルダに保存し、そこから読み込みます。
- 補正電磁石と BPM の **Save** ボタンと **Clear** ボタンが動くようになりました。これまでは処理自体はあってもボタンにつながっておらず、Save の処理は `SaveOrLoad` に存在しない `_save_correctors()` / `_save_bpms()` を呼んでいました。
- 一覧が空のとき（データフォルダを読み込む前）は、**Load** を押しても何もしません。

## ビームライン上の位置が分からない機器（`response_matrix/ResponseMatrix_DFS_WFS.py`）
- `_compute_response_matrix()` で、記録された並びにない BPM と補正電磁石は、一覧を表示したうえで無視します。これまでは `sequence.index()` でエラーになっていました。
- 選んだ BPM が1台も並びにないときは、分かりやすい `RuntimeError` を出します。

## 見た目（`response_matrix/UI files/ComputeResponseMatrix_GUI.ui`）
- 機器・補正電磁石・BPM の枠は、枠線に `palette(mid)` を使い、固定の薄いグレーの背景をやめました。システムのライト／ダークテーマに合わせて表示されます。

---

## 確認結果
- 変更した Python ファイルはコンパイルが通ります（`python -m py_compile`）。
- GUI を実際に開いての動作確認はしていません。

## 使える軌道データがないときのエラー文（`0df71b8`、`response_matrix/ResponseMatrix_DFS_WFS.py`）
- 使える BPM に軌道がなく、すべての DATA ペアが飛ばされたとき、`_compute_response_matrix()` は「No DATA pair has orbit data at the usable BPMs」というエラーを出します。使えた BPM の一覧と、`python -m machine_interfaces_atf2.repair_ext_sequence` での修復方法も表示します。
- これまでは、その後の処理で「No complete corrector readbacks available for response-matrix.」と表示され、補正電磁石に問題があるように見えていました。
- 2026-11-19 の ATF2 Ext データで起きていた現象です。記録された並びには `MB2X` と `MSF1FF` しかなく、選んだ補正電磁石より後ろの BPM に絞ると、値が0の `MSF1FF` しか残りませんでした。

## 並びの修正前に記録された ATF2 Ext データを使うには
先に並びを作り直し（`machine-interfaces` のブランチにあるツール）、GUI では `_sequence_fixed` のコピーの中のフォルダを選んでください。

```bash
python -m machine_interfaces_atf2.repair_ext_sequence ~/CERN-Flight_Simulator-Data
```

修復した 2026-11-19 のデータで、`ATF2_Ext_20261119_112700_Orbit` について `ZH1X` と `ZV1X` の応答行列が50台の BPM で計算できました（Python から確認。GUI では未確認）。
