---
title: "Ubuntuで複数・最新のPythonのバージョンを管理する (それぞれパスを通す)"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['python', 'ubuntu']
published: true
---

# やること

1. ソースコードからコンパイル
2. パスを通す

:::details 検証環境

```bash
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="20.04.3 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.3 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

:::

# ソースコードからコンパイル

最新の Python のバージョンはこちらで確認できます。
執筆日時点では、3.7〜3.10 がメンテナンス中のようです。

https://www.python.org/downloads/

最新のバージョンに追従する確実な方法は、**Python 公式から配布されているソースコードをビルドすること**のようです。
一連の作業をスクリプト化できる、非常に便利なサイトが用意されています。

https://www.build-python-from-source.com/

個人的には、 "POST INSTALLATION STUFF" や "OS RELATED STUFF" は全てオンに、それ以外はデフォルトのままが良いのかなと思っていたりします。

生成されたコマンド群をコピーし、実行します。
実行の仕方は諸説ありそうですが、例えばこのように実行できます。

```bash
$ code install-py.sh  # VS Codeにコピーしたスクリプトを貼り付け、保存
$ bash install-py.sh
$ rm install-py.sh
```

正常にインストールされているか確認します。
例えば、執筆日時点の最新 3.10.2 の場合は次のようになります。

```bash
$ /opt/python/3.10.2/bin/python -V
Python 3.10.2
```

ただし、 "add some soft links for comfortable usage" をオンにしていない場合は、 `/opt/python/3.10.2/bin/python3.10` を実行する必要があります。

# パスを通す

apt でインストールされた python は `/usr/bin/python3.x` に実行ファイルを用意されているようです。
これにならい、シンボリックリンクを張ります。

そうすれば、 `python3.x` で任意のバージョンを実行できます。

```bash
$ ln -s /opt/python/3.10.2/bin/python /usr/bin/python3.10

$ python3.10 -V
Python 3.10.2  # 🎉
```

:::message
pip や venv といったモジュールを使う場合は、 `-m` オプションから実行してください。
こうすることで、実行したい Python バージョンに適したモジュールが実行されます。
`python3.10 -m pip install pandas` や `python3.10 -m venv .venv` といった具合です。
:::

# おまけ

## 古いバージョンをアンインストールする

作成した関連ファイルを削除します。

```bash
$ sudo rm -r /opt/python/3.10.2
$ sudo rm /usr/bin/python3.10
```

## `update-alternatives` 使わないの？

検討しましたが、次の理由から使いませんでした。

いずれも、ちょっと引っかかるな〜ぐらいのノリです。
`update-alternatives` を否定するわけではありません。

### 都度、バージョンを指定するのが手間

実際に Python のバージョンを指定するのは、 venv の作成時といった限られた機会だけかとは思います。
しかし、そのたびに

- あのコマンドなんだっけな？
- ああ、 `update-alternatives` か、どういう用法だっけな？
- これか、`sudo update-alternatives --config python` っと...

とするのは結構手間だと思います。

わざわざ root 権限でバージョンを切り替えるのも、ちょっとモヤっとします。
既知のコマンドの上書きで、セキュリティ上の懸念が生じうるのでしょうがないのでしょうか。

### 戻さないと戻らない

`update-alternatives` の設定値は実行した bash プロセスだけでなく、システム全体に影響を与えるようです。
そのため、設定を変更して作業した後は、 `sudo update-alternatives --auto python` で規定値に変更する必要があります。

`.bash_logout` に書いちゃえばいいのかもしれないですが、複数窓開いていたときに困りそうです。

### VS CodeのPython Interpreterに選択肢として表示されない

要は、パスが通っていないことによる弊害です。
このようなちょっとした弊害が散らばっていそうな気がしました。

（パスを通したり設定をいじったりすればいいという話ですが...）
