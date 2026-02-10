# GPUクラスタ講習：Singularity + venv でPython環境を拡張する

この講習では、**Singularityコンテナを土台にして、venvでPythonパッケージを追加する方法**を学びます。

これはHPC環境で非常によく使われる構成です。

構成のイメージ：

```
GPUクラスタ
 ├─ コンテナ（PyTorch + CUDA）
 │
 └─ venv（ユーザー追加パッケージ）
```

役割分担：

| 層           | 役割                        |
| ----------- | ------------------------- |
| Singularity | CUDA / PyTorch / Python本体 |
| venv        | 追加Pythonパッケージ             |

---

# 1. 作業ディレクトリ

ログインノードで：

```bash
mkdir -p ~/vgg_intro
cd ~/vgg_intro
```

---

# 2. Singularityを読み込む

```bash
module load singularity/4.0.3
```

---

# 3. PyTorchコンテナを取得（1回だけ）

```bash
singularity pull pytorch-cu118.sif \
docker://pytorch/pytorch:2.2.2-cuda11.8-cudnn8-runtime
```

`.sif` は「実行環境が丸ごと入ったファイル」です。

---

# 4. venv をコンテナのPythonで作る

重要ポイント：

> venvは「実際に使うPython」で作る

```bash
singularity exec -B "$PWD:/work" ./pytorch-cu118.sif bash -lc '
cd /work
python -m venv .venv
'
```

これで作られる：

```
~/vgg_intro/.venv
```

---

# 5. 追加パッケージをインストール（例：biopython）

```bash
singularity exec -B "$PWD:/work" ./pytorch-cu118.sif bash -lc '
cd /work
source .venv/bin/activate
python -m pip install -U pip
python -m pip install biopython
'
```

確認：

```bash
singularity exec -B "$PWD:/work" ./pytorch-cu118.sif bash -lc '
cd /work
source .venv/bin/activate
python -c "import Bio; print(Bio.__version__)"
'
```

---

# 6. PBSジョブスクリプト

`JobScript.sh`

```bash
#!/bin/bash
#PBS -N "VGG16_intro"
#PBS -q DebugA
#PBS -l select=1:ncpus=2:ngpus=1:mem=16gb
#PBS -l walltime=00:10:00
#PBS -j oe

cd "$PBS_O_WORKDIR"
module load singularity/4.0.3

export TORCH_HOME="$PBS_O_WORKDIR/.cache/torch"
mkdir -p "$TORCH_HOME"

singularity exec --nv \
  -B "$PBS_O_WORKDIR:/work" \
  ./pytorch-cu118.sif \
  bash -lc '
    cd /work
    source .venv/bin/activate
    python vgg_intro.py --image test.jpg
  '
```

---

# 7. ジョブ投入

```bash
qsub JobScript.sh
```

---

# なぜこの構成を使うのか

HPCでは次の制約があります：

* root権限が無い
* OSパッケージを入れられない
* CUDA依存が壊れやすい
* ノード間で環境が違う

この構成はそれを解決します。

---

## コンテナの役割

固定環境を提供：

* Python
* PyTorch
* CUDA
* cuDNN

再現性の核になります。

---

## venvの役割

ユーザーパッケージ管理：

* biopython
* scikit-learn
* transformers
* etc

root不要で追加できます。

---

# 重要な理解

このコマンド：

```bash
singularity exec image.sif bash -lc "..."
```

は

```
コンテナ内でbashを起動してコマンド実行
```

を意味します。

構造：

```
ホスト
  └─ singularity
        └─ bash
              └─ python
```

---

# よくある間違い

## venvをホストPythonで作る

NG例：

```bash
python -m venv .venv
```

必ずコンテナのPythonで作ること：

```bash
singularity exec image.sif python -m venv .venv
```

---

## venvをactivateし忘れる

ジョブでは必須：

```bash
source .venv/bin/activate
```

---

# ディレクトリ構成（完成形）

```
vgg_intro/
├── pytorch-cu118.sif
├── vgg_intro.py
├── test.jpg
├── JobScript.sh
├── .venv/
└── .cache/
```

---

# まとめ

この講習で学んだ構成：

```
Singularity = OS + CUDA + PyTorch
venv        = 追加Pythonパッケージ
PBS         = 実行管理
```

HPCでの標準的な開発スタイルです。

どれに発展させますか？
