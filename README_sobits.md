<a name="readme-top"></a>

[← Hunyuan3D-2 README](README.md)

# Hunyuan3D-2mv Setup Guide

複数方向の画像から3Dメッシュを生成する`Hunyuan3D-2mv`のセットアップガイドです。
このフォークでは、低VRAM環境でshapeモデルをCPUメモリに待機させ、推論対象だけを
NVIDIA GPUへ移すCPU offloadに対応しています。推論処理自体はCUDA GPUで実行されます。

<details>
<summary>目次</summary>

- [機能](#機能)
- [環境](#環境)
- [依存パッケージ](#依存パッケージ)
- [インストール](#インストール)
- [実行方法](#実行方法)
- [画像と生成設定](#画像と生成設定)
- [メッシュの書き出し](#メッシュの書き出し)
- [3Dプリント](#3dプリント)
- [トラブルシューティング](#トラブルシューティング)
- [参考資料](#参考資料)

</details>

## 機能

- Front / Back / Left / Rightの1～4画像から3D形状を生成
- NVIDIA GPUによる推論
- 低VRAM向けshapeモデルCPU offload
- 背景除去
- 浮遊片・不正面の除去
- GLB / OBJ / PLY / STL出力
- Gradio Web UI

本ガイドはshape生成を対象とします。テクスチャ生成は公式目安で16GB VRAMが必要なため、
低VRAM環境では`--disable_tex`を指定します。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 環境

| 項目 | 使用環境 | 備考 |
| --- | --- | --- |
| OS | Ubuntu 24.04 | ROS 2 Jazzy環境でも利用可能 |
| Python | 3.12 | venvを使用 |
| CUDA Toolkit | 12.8 | `nvcc --version`で確認 |
| PyTorch | 2.11.0+cu128 | CUDA 12.8版 |
| GPU | NVIDIA RTXシリーズ | 6GB以上推奨 |
| 低VRAM | 約4GB | CPU offload使用、設定を下げる必要あり |
| 空き容量 | 20GB以上推奨 | モデルキャッシュは約10GB |

`nvidia-smi`に表示されるCUDA Versionはドライバが対応する上限です。
PyTorchとCUDA拡張の互換性確認には`nvcc --version`と`torch.version.cuda`を使用します。

<details>
<summary>ディレクトリ構成</summary>

```text
Hunyuan3D-2/
├── README.md
├── README_sobits.md
├── gradio_app.py
├── setup.py
├── hy3dgen/
│   ├── shapegen/
│   └── texgen/
└── .venv/
    ├── bin/python
    └── lib/python3.12/site-packages/
```

</details>

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 依存パッケージ

公式`requirements.txt`はバージョンが固定されていないため、本環境では互換性を確認した
バージョンを個別に導入します。

| 分類 | パッケージ |
| --- | --- |
| GPU | torch 2.11.0+cu128, torchvision 0.26.0+cu128 |
| 基本 | numpy 1.26.4, opencv-python 4.11.0.86, pillow |
| モデル | transformers 4.57.6, diffusers 0.35.2, accelerate 1.14.0, safetensors 0.8.0 |
| 3D | trimesh 4.12.2, pymeshlab 2025.7.post1, pygltflib 1.16.5, xatlas 0.0.11 |
| Web UI | gradio 5.49.1, fastapi 0.139.2, uvicorn 0.51.0 |
| 背景除去 | rembg 2.0.69, onnxruntime 1.27.0 |
| その他 | einops, omegaconf, PyYAML, tqdm, pybind11, ninja |

venvはROS 2やUbuntuのシステムPythonから依存関係を分離します。
`sudo pip`、`pip --user`、`--break-system-packages`は使用しません。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## インストール

<details>
<summary>インストールコマンドを開く</summary>

システムパッケージを導入します。

```bash
sudo apt update
sudo apt install -y python3-venv python3-dev build-essential ninja-build git git-lfs
```

このフォークを取得し、venvを作成します。すでにclone済みなら`cd`以降を実行します。

```bash
cd ~/colcon_ws/src
git clone https://github.com/TeamSOBITS/Hunyuan3D-2.git
cd Hunyuan3D-2

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
```

CUDA 12.8版PyTorchを導入します。

```bash
python -m pip install \
  "torch==2.11.0+cu128" \
  "torchvision==0.26.0+cu128" \
  --index-url https://download.pytorch.org/whl/cu128
```

その他の依存関係とHunyuan3D本体を導入します。

```bash
python -m pip install \
  "numpy==1.26.4" \
  "opencv-python==4.11.0.86" \
  "transformers==4.57.6" \
  "diffusers==0.35.2" \
  "gradio==5.49.1" \
  "accelerate==1.14.0" \
  "safetensors==0.8.0" \
  "einops==0.8.2" \
  "omegaconf==2.3.1" \
  "PyYAML==6.0.3" \
  "tqdm==4.69.0" \
  "trimesh==4.12.2" \
  "pymeshlab==2025.7.post1" \
  "pygltflib==1.16.5" \
  "xatlas==0.0.11" \
  "pybind11==3.0.4" \
  "ninja==1.13.0" \
  "fastapi==0.139.2" \
  "uvicorn==0.51.0" \
  "rembg==2.0.69" \
  "onnxruntime==1.27.0"

python -m pip install --no-deps -e .
```

確認します。

```bash
python -m pip check

python - <<'PY'
import torch
import hy3dgen

print("PyTorch:", torch.__version__)
print("PyTorch CUDA:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
print("GPU:", torch.cuda.get_device_name(0))
print("hy3dgen:", hy3dgen.__file__)
PY
```

</details>

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 実行方法

<details>
<summary>GPU・低VRAMモードで起動する</summary>

```bash
cd ~/colcon_ws/src/Hunyuan3D-2
source .venv/bin/activate

python gradio_app.py \
  --model_path tencent/Hunyuan3D-2mv \
  --subfolder hunyuan3d-dit-v2-mv \
  --host 0.0.0.0 \
  --port 7860 \
  --device cuda \
  --low_vram_mode \
  --disable_tex
```

</details>

ブラウザで`http://localhost:7860`を開きます。

初回起動時にHugging FaceからHunyuan3D-2mvモデル約10GBと、
rembgの`u2net.onnx`約176MBを取得します。2回目以降はローカルキャッシュを利用します。

低VRAMモードでは`Low VRAM mode: enabling shape model CPU offload to cuda`と表示されます。

モデルはCPUメモリに待機しますが、Diffusion SamplingとVolume DecodingはCUDA GPUで実行されます。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 画像と生成設定

マルチビュー画像は同じ物体・姿勢・縮尺・上下方向で用意し、カメラだけを移動します。
不正確な画像を追加すると結果が悪化するため、Front 1枚から始めてBack、Left、Rightを順に追加します。

| 設定 | 動作確認 | 低VRAM推奨 | 品質優先 |
| --- | ---: | ---: | ---: |
| Inference Steps | 5 | 10 | 30 |
| Octree Resolution | 64 | 128 | 256 |
| Guidance Scale | 5 | 5 | 5 |
| Number of Chunks | 1000 | 1000 | 1000～2000 |
| Randomize Seed | OFF | OFF | OFF |

- Stepsを増やすと形状品質と処理時間が上がる
- Octree Resolutionを増やすと細部とメモリ使用量が増える
- Number of Chunksを下げるとメモリ使用量は減るが処理時間が増える
- 背景がある画像では`Remove Background`を有効にする

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## メッシュの書き出し

`Export`タブで形式を選び、`Transform`を押してから`Download`します。
`Transform`では小さな浮遊メッシュと不正な面を除去します。

| 形式 | 内容 | 主な用途 |
| --- | --- | --- |
| GLB | メッシュ・マテリアル・テクスチャを1ファイルに格納 | Blender編集、保管、Web表示 |
| OBJ | メッシュ・UV・法線。材質は別ファイルの場合あり | 3D編集ソフト間の受け渡し |
| PLY | メッシュ・頂点色・法線 | 3Dスキャン、頂点カラー |
| STL | 三角形メッシュのみ | FDM・光造形3Dプリント |

`Simplify Mesh`は必要な場合だけ有効にします。人物やフィギュアでは、
Target Face Number 50,000～100,000を目安にし、細部を残す場合は無効にします。

大きな欠損、接続したノイズ、耳や手足の補修にはBlenderなどのメッシュ編集ソフトが必要です。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 3Dプリント

単色FDM・光造形ではSTLを推奨します。編集・保管用の`model_original.glb`と、
スライサー用の`model_print.stl`を両方保存すると安全です。

STLには単位がないため、Cura、PrusaSlicer、OrcaSlicer、Bambu Studioなどで
実寸をmm単位に設定します。造形前に穴、非多様体、浮遊片、肉厚、接地面、
サポート材を確認してください。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## トラブルシューティング

<details>
<summary>よくある警告と対処</summary>

### CUDA out of memory

`--low_vram_mode`を指定し、Octree ResolutionとNumber of Chunksを下げます。
GPUを使用する他のプロセスも終了してください。

### TBB threading layer warning

NumbaのCPU並列バックエンドに関する警告です。CUDA推論には影響しません。

### フォント・manifestの404

Gradioの静的ファイルに関する警告です。画像アップロードや3D生成には影響しません。

### モデル取得が止まって見える

モデルは`~/.cache/huggingface/hub/`へ保存されます。空き容量とキャッシュサイズを確認します。

```bash
df -h ~
du -sh ~/.cache/huggingface 2>/dev/null
```

</details>

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>

## 参考資料

- SOBITS fork: https://github.com/TeamSOBITS/Hunyuan3D-2
- Upstream repository: https://github.com/Tencent-Hunyuan/Hunyuan3D-2
- Hunyuan3D-2mv model: https://huggingface.co/tencent/Hunyuan3D-2mv
- Hunyuan3D-2 model collection: https://huggingface.co/tencent/Hunyuan3D-2
- Technical report: https://arxiv.org/abs/2501.12202
- PyTorch installation: https://pytorch.org/get-started/locally/
- Blender: https://www.blender.org/
- License: [LICENSE](LICENSE)

本リポジトリの利用条件とモデルの利用条件を必ず確認してください。

<p align="right">(<a href="#readme-top">上に戻る</a>)</p>
