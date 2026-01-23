# vgg16-gpu-hpc-tutorial
# GPUクラスタ講習：PBS + Singularityで VGG16 をGPU実行する（入門）


* **ジョブシステム（PBS）**の基本
* **Singularityコンテナ**で深層学習を動かす方法
* 有名CNNモデル **VGG16** の推論（画像分類）を **GPUで実行**
* [ssh configファイルの作成](sshconfig.md)
* [vscodeでのremote sshの使用](remotessh.md)

を、最短で体験します。

---

## 目標（ゴール）

最終的に、以下のような出力を得ます。

* `device = cuda`（GPUを使っている）
* GPU名が表示される（例：RTX A6000）
* 画像の分類結果 Top-5 が表示される

---

## 1. 重要ルール：ログインノードと計算ノード

GPUクラスタには大きく2種類のノードがあります。

### ログインノード（例：hokushin01）

**やってよいこと**

* ファイル作成・編集（スクリプトを書く）
* ジョブスクリプト作成
* ジョブ投入（`qsub`）
* ファイルの確認（`ls`, `cat` など）

**やってはいけないこと**

* 学習や推論などの計算
* GPUを使う処理
* 大きなダウンロードや重い `pip install`

👉 計算は他のユーザーにも影響するため、**実行は計算ノードで行う**。

---

### 計算ノード（GPUノード）

**ジョブを投入するとPBSが割り当ててくれる**ノードです。
ここでGPUを使った計算を実行します。

---

## 2. 今日の方針：なぜ Singularity を使うのか？

初学者がクラスタでPython環境を作ると、よく次の問題が起きます。

* ノードによってPythonのバージョンが違う
* CUDAやPyTorchの組み合わせが合わない
* venvを作っても計算ノードで動かない

そこでこの講習では **Singularity（コンテナ）**を使います。

### Singularityの良い点

* Python / PyTorch / CUDA環境が **丸ごと揃った状態**で使える
* ノード差による「動かない」を避けられる
* HPCでよく使われる（研究でそのまま使える）

---

## 3. ディレクトリ（作業場所）を作る

ログインノードで以下を実行：

```bash
mkdir -p ~/vgg_intro
cd ~/vgg_intro
```

### 何をしている？

* `mkdir -p`：ディレクトリが無ければ作る（既にあってもエラーにならない）
* `cd`：その場所に移動

---

## 4. 実行するPythonプログラムを作る（VGG16推論）

### `vgg_intro.py` を作成

以下を `vgg_intro.py` として保存してください。

```python
import argparse
import torch
from torchvision import models
from PIL import Image

def main():
    # 1) コマンドライン引数（画像ファイル名）を受け取る
    parser = argparse.ArgumentParser()
    parser.add_argument("--image", required=True)
    args = parser.parse_args()

    # 2) GPUがあればcuda、なければcpuを選ぶ
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    print(f"[INFO] device = {device}")
    if device.type == "cuda":
        print(f"[INFO] cuda device = {torch.cuda.get_device_name(0)}")

    # 3) 学習済みVGG16モデルを読み込む
    weights = models.VGG16_Weights.DEFAULT
    model = models.vgg16(weights=weights).to(device)
    model.eval()  # 推論モード（学習ではない）

    # 4) VGG16の入力用に画像を前処理する（リサイズ、正規化など）
    transform = weights.transforms()
    img = Image.open(args.image).convert("RGB")
    x = transform(img).unsqueeze(0).to(device)  # (1,3,224,224)

    # 5) 推論して確率を出す
    with torch.no_grad():  # 勾配計算をしない（推論なので高速）
        y = model(x)
        prob = torch.softmax(y, dim=1)[0]
        top5 = torch.topk(prob, 5)

    # 6) 予測ラベルを表示する
    labels = weights.meta["categories"]
    print("[RESULT] Top-5 predictions:")
    for score, idx in zip(top5.values, top5.indices):
        print(f"  - {labels[idx]}: {score:.4f}")

if __name__ == "__main__":
    main()
```

---

## 5. テスト画像を用意する

`test.jpg` を `~/vgg_intro` に置きます。

* 自分の写真
* ネットで拾ったサンプル画像
* 何でもOK（jpg/png）

例：

```bash
ls
# vgg_intro.py  test.jpg
```

---

## 6. Singularityイメージを準備する（1回だけ）

### コマンド

```bash
module load singularity/4.0.3
cd ~/vgg_intro/
singularity pull pytorch-cu118.sif docker://pytorch/pytorch:2.2.2-cuda11.8-cudnn8-runtime
```

### 何をしている？

#### `module load singularity/4.0.3`

クラスタでは、ソフトウェアが `module` で管理されています。
このコマンドは **Singularity を使える状態にする**ものです。

#### `singularity pull ... docker://...`

* `docker://...` は **Dockerイメージの場所**
* それをダウンロードし、Singularity用の **`.sif` ファイル**に変換します

`.sif` は「実行環境が丸ごと入った1ファイル」です。
一度作れば、次回以降ダウンロードは不要です。

---

## 7. PBSジョブスクリプトを作る（計算ノードで実行するため）

### `JobScript.sh` を作成

```bash
#!/bin/bash
#PBS -N "VGG16_intro" 
#PBS -q DebugA                 
#PBS -l select=1:ncpus=2:ngpus=1:mem=16gb   
#PBS -l walltime=00:10:00                   
#PBS -j oe                                  

cd "$PBS_O_WORKDIR"                         
module load singularity/4.0.3               

# PyTorchモデルの重みをキャッシュする場所（毎回ダウンロードしないため）
export TORCH_HOME="$PBS_O_WORKDIR/.cache/torch"
mkdir -p "$TORCH_HOME"

# --nv : GPUをコンテナ内から使えるようにするオプション
# -B   : ホストの作業ディレクトリをコンテナ内/workに見せる
singularity exec --nv \
  -B "$PBS_O_WORKDIR:/work" \
  ./pytorch-cu118.sif \
  bash -lc "cd /work && python vgg_intro.py --image test.jpg"
```

---

## 8. ジョブ投入（ログインノードでやる）

```bash
qsub JobScript.sh
```

### 何をしている？

PBSに「このJobScriptを計算ノードで実行してください」と依頼しています。

---

## 9. ジョブ状態の確認

```bash
qstat -u $USER
```

| 表示  | 意味        |
| --- | --------- |
| Q   | 待機中（順番待ち） |
| R   | 実行中       |
| 消えた | 終了（成功/失敗） |

---

## 10. 出力ログを見る

ジョブが終わると、同じディレクトリにログファイルができます。

例：

```bash
ls *.o*
cat VGG16_intro.o12345
```

### 成功例（期待する出力）

```text
[INFO] device = cuda
[INFO] cuda device = NVIDIA RTX A6000
[RESULT] Top-5 predictions:
  - ...
```

---

## 11. よくある質問：CUDA module を使っていないの？

この講習では、CUDA module（`module load cuda/...`）は基本使いません。

理由：

* CUDA/PyTorch環境は **コンテナ内に揃っている**
* GPUドライバだけホストから借りればよい
* `singularity exec --nv` が GPU使用設定を自動で行う

ただし、GPUドライバ自体はサーバー側に依存します。

---

## 12. まとめ（今日の到達点）

✅ PBSでGPUジョブを投げられる
✅ Singularityで深層学習環境を再現できる
✅ VGG16で画像分類をGPUで実行できた

---

