## SSH 接続用 config ファイルの設定方法

以下の手順で、SSH の `config` ファイルを設定してください。

### 1. `.ssh` ディレクトリに移動

まず、ターミナルを開いて次のコマンドを実行します。

```bash
cd ~/.ssh
```

※ `.ssh` ディレクトリが存在しない場合は、以下で作成してください。

```bash
mkdir ~/.ssh
chmod 700 ~/.ssh
```

---

### 2. `config` ファイルを作成・編集

次に、`config` ファイルをエディタで開きます（なければ新規作成されます）。

```bash
nano ~/.ssh/config
```

または、VS Code を使っている場合：

```bash
code ~/.ssh/config
```

---

### 3. 以下の内容を `config` に記述

```sshconfig
Host hokushin
     HostName hokushin.fro-local.kitasato-u.ac.jp
     Port 22
     User ＜自分のユーザー名＞
     IdentitiesOnly yes
     IdentityFile ＜自分の秘密鍵ファイルへのパス＞
```

#### 各項目の説明

* `Host hokushin`
  → 接続時に使うショートカット名です（この名前は共通でOK）

* `HostName hokushin.fro-local.kitasato-u.ac.jp`
  → 接続先サーバのアドレス（変更しない）

* `Port 22`
  → SSH のポート番号（通常は 22）

* `User ＜自分のユーザー名＞`
  → **各自に割り当てられたユーザー名に変更してください**

* `IdentityFile ＜自分の秘密鍵ファイルへのパス＞`
  → **自分の環境にある秘密鍵ファイルのパスを指定してください**
  例：

  ```text
  /Users/yourname/.ssh/id_rsa_hokushin
  ```

---

### 4. ファイルの権限を設定

設定後、以下のコマンドを実行してください。

```bash
chmod 600 ~/.ssh/config
```

---

### 5. 接続確認

以下のコマンドで接続できるか確認します。

```bash
ssh hokushin
```

うまく設定できていれば、ユーザー名やホスト名を毎回入力せずに接続できます。

---

### ⚠ 注意事項

* 秘密鍵（`IdentityFile`）は**他人に見せないでください**
* エラーが出る場合は、ユーザー名・鍵ファイルのパスが正しいかを確認してください
