# ブランチ `atf2-ext-orbit-analysis` の変更内容（response-matrix）

ベース：`flight-simulator-framework/response-matrix` の `master`（`bd710ba`）
英語版：[CHANGES_atf2-ext-orbit-analysis.en.md](CHANGES_atf2-ext-orbit-analysis.en.md)

| コミット | 概要 |
|---|---|
| `8760772` | 応答行列 GUI のデータフォルダと選択ファイルの扱いを改善 |
| `0df71b8` | 使える軌道データがないとき、補正電磁石のエラーではなく原因を表示する |
| `fc62455` | 補正電磁石用の応答行列で、四極ムーバーのペアを読み飛ばす |
| `48e06ae` | 三角化のために、並びの中から四極ムーバーの位置を見つける |
| （このコミット） | 四極ムーバーの応答を、エンコーダではなく設定値あたりで計算する |

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

## 補正電磁石とムーバーをまとめて測った SysID のフォルダ（`response_matrix/ResponseMatrix_DFS_WFS.py`）
- 補正電磁石用の計算で、`_compute_response_matrix()` は四極ムーバーのペア（`DATA_<四極>_<x|y>_...`）を飛ばします。system-identification の `atf2-ext-orbit-analysis` ブランチにある `SysID_corrector_and_mover_GUI.py` が、補正電磁石のペアと同じフォルダに書くファイルです。
- 計算結果は変わりません。ムーバーのペアでは補正電磁石の値が変わらず、軌道が ± に同じだけ動くので、最小二乗の計算で打ち消し合うためです（シミュレーションのフォルダで確認：行列は同一、差は 6×10⁻¹⁶）。これらのファイルを読み込まずに済み、「使える軌道データがない」の判定にも数えなくなります。

## 四極ムーバーの行列の三角化（`response_matrix/ResponseMatrix_DFS_WFS.py`、`_compute_qm_response_matrix`）
- **問題：** `triangular=True`（`BBA_QM_GUI.py` は常にこれを使います）では、ムーバーより上流の BPM の応答を0にします。ムーバーの位置をその名前（`QM15FF`）で探していましたが、記録された並びにあるのは、その上の BPM（`MQM15FF`、ATF2 Ext の実機データ）か分割された要素（`QM15FF_1`、RF-Track）です。見つからないとムーバーが無限に下流にある扱いになり、すべての BPM が「上流」と判定されて、行列全体が0になっていました。既存の QM 版 BBA にも影響します。
- **変更：** 位置を `QM15FF`、`QM15FF_1`、`MQM15FF` のうち並びにあるもので決めます。どれでも見つからないムーバーは、列を失う代わりに全 BPM を残します（メッセージを表示）。
- **確認：** シミュレーションのフォルダで、各ムーバーの列がムーバーより下流でだけ0以外になり、最大値が三角化なしの場合と一致しました（例：`QM15FF` 0.194、`QD0FF` 0.0020）。

## 並びの修正前に記録された ATF2 Ext データを使うには
先に並びを作り直し（`machine-interfaces` のブランチにあるツール）、GUI では `_sequence_fixed` のコピーの中のフォルダを選んでください。

```bash
python -m machine_interfaces_atf2.repair_ext_sequence ~/CERN-Flight_Simulator-Data
```

修復した 2026-11-19 のデータで、`ATF2_Ext_20261119_112700_Orbit` について `ZH1X` と `ZV1X` の応答行列が50台の BPM で計算できました（Python から確認。GUI では未確認）。

## 四極ムーバーの応答は設定値あたり（このコミット、`_compute_qm_response_matrix`）
- ムーバーの変化量に、エンコーダ（`xact`/`yact`）ではなく設定値（`xdes`/`ydes`）を使います。設定値が無いときだけエンコーダを使います。
- 理由：BBA（QM 版と補正電磁石＋ムーバー版）はムーバーを設定値で動かすので、応答も設定値あたりでないと合いません。2026-11-19 の実機データでは、`QF1FF` と `QD0FF` の x エンコーダが設定値と逆向き（比 −1.00）に数えていて、エンコーダ基準の応答ではこの2列の補正の向きが逆になっていました。他の19台と y の比は 1.00 ± 0.01 なので、それらの結果はほぼ変わりません。
