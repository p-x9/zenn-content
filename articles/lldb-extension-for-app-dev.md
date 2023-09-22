---
title: "LLDBのカスタムコマンドを実装してアプリ開発を快適に"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["iOS", "LLDB"]
published: true
---
# 目次

- [LLDBとは](#lldbとは)
- [LLDBで使用可能なコマンド](#lldbで使用可能なコマンド)
- [LLDBのカスタムコマンド](#lldbのカスタムコマンド)
  - [例1: オブジェクトのプロパティ情報を表示するコマンド](#例１-オブジェクトのプロパティ情報を表示するコマンド)
  - [例2: 実行中のアプリのBundleパスをFinderで開く（Simulator限定）](#例2-実行中のアプリのbundleパスをfinderで開くsimulator限定)
    - [準備](#準備)
      - [1. import](#1-import)
      - [2. 関数の定義](#2-関数の定義)
      - [3. Bundleパスの取得](#3-bundleパスの取得)
      - [4. FinderでBundleパスを開く](#4-finderでbundleパスを開く)
      - [5. LLDBにカスタムコマンドを読み込み](#5-lldbにカスタムコマンドを読み込み)
      - [実装したPythonによるカスタムコマンドの全貌](#実装したpythonによるカスタムコマンドの全貌)
- [アプリ開発のためのLLDBカスタムコマンド](#アプリ開発のためのlldbカスタムコマンド)
  - [UIの階層を表示する機能](#uiの階層を表示する機能)
  - [ファイルの階層を表示する](#ファイルの階層を表示する)
  - [シミュレータで実行中のアプリのパスをFinderで開く](#シミュレータで実行中のアプリのパスをfinderで開く)
  - [その他](#その他)
- [終わりに](#終わりに)

# LLDBとは

LLDBは、Xcodeに付属されているデバッグツールの一部です。
アプリケーションのデバッグや解析に役立つツールです。
特に、バグの原因を突き止める場合には大いに役立ちます。

LLDBでは例えば、以下のようなことを行うことが可能です。

- **ブレークポイント**:
コード内の特定の行に到達した際にプログラムの実行を一時停止して、現在の変数の状態を確認したりできます。

- **ステップ実行**:
コードを一行ずつ実行し、変数の値やスタックトレースを確認しながらデバッグできます。

- **ウォッチポイント**:
特定の変数やメモリアドレスの値が変更された際にプログラムを一時停止。

- ...

# LLDBで使用可能なコマンド

Xcodeでは、UIによってブレークポイントを設定したり、ステップ実行を行ったりすることができるようになっています。
が、もちろんコマンドからも操作可能です。

一部のコマンドはXcodeからでもよく使います。

- **プロパティの表示**
  プロパティの内容を表示します。

  ```sh
  po <プロパティ名>
  ```

  関数の実行もできます。

  ```sh
  po print("hello")
  ```

- **ブレークポイントを抜ける**
  ブレークポイント停止中に、プログラムの実行を再開する

  ```sh
  c
  ```

- **ステップ実行（ステップオーバー）**
- 1行ずつ実行する。
  関数が呼ばれている場合も、関数内に入らず(関数内で止まらず)1行ずつ進める。

  ```sh
  n
  ```

- **ステップ実行（ステップイン）**
  1行ずつ実行する。
  関数が呼ばれている場合は、関数内に入ってさらに1行ずつ進める。

  ```sh
  s
  ```

- 他にも色々...

# LLDBのカスタムコマンド

LLDBでは、ユーザが新たに定義したカスタムコマンドを追加することができるようになっています。
ここでは、例としていくつか簡単なカスタムコマンドを定義してみましょう。

## 例１: オブジェクトのプロパティ情報を表示するコマンド

SwiftではリフレクションのためにMirrorという構造体が用意されています。
以下のようなコードで、オブジェクトのプロパティ情報を取得することができます。

```swift
let object = <何かしらオブジェクト>
let mirror = Mirror(refrecting: object)
mirror?.children.forEach {
  print("\($0.label): $0.value")
}
```

これを以下のようなコマンドで簡単に実行できるようにしてみましょう。

```
mirror object
```

lldbで以下のようなコマンドを実行すると、上記のような「mirror」というカスタムコマンドが使用できるようになります。

```sh
command regex mirror 's/(.+)/po Mirror(refrecting: %0)?.children.enumerated().forEach { print("\($1.label ?? "[($0)]"): \($1.value)") }/'
```

長いですが、内容はシンプルです。
まず、最初に示した、swiftコードを1行にまとめました。

```swift
Mirror(refrecting: %0)?.children.enumerated().forEach { print("\($1.label ?? "[\($0)]"): \($1.value)") }
```

「%0」はLLDBのカスタムコマンドに与えられた第一引数を表します。
例えば、「mirror test」と実行されれば、「%0」は「test」を表します。

「command regex」は、以下のフォーマットで、入力されたコマンドを別のコマンドに置き換えるものです。
正規表現「(.+)」は任意の文字列を表します。
よって、先ほどカスタムコマンド追加のために実行したコマンドは、

```sh
mirror 引数
```

と入力された場合、

```sh
po Mirror(refrecting: 引数)?.children.enumerated().forEach { print("\($1.label ?? "[\($0)]"): \($1.value)") }
```

に置き換えて実行するというものというわけです。

結局長いコマンドのエイリアスを作っているという感じですね。
(厳密には「command alias」というものもあるのでやや語弊はあるかもしれませんが。。)

もし、LLDB起動のたびに毎回使用したいのであれば、~/.lldbinitというファイルに同様の内容を書いておけば良いです。
~/.lldbinitはLLDBの起動のたびに読み込まれるファイルで、自分で「command regex mirror ~」のコマンドを実行せずともすぐに「mirrorカスタムコマンドが使えるようになります。

「mirror」というコマンド名はやや長いので、以下のように別名をつけておくのもアリかもしれませんね

```sh
command alias mr mirror
```

## 例2： 実行中のアプリのBundleパスをFinderで開く（Simulator限定）

例１では比較的簡単な処理のカスタムコマンドの実装を行いました。
でも、もっと高度な処理を行うコマンドを実装してみたい場合はどうでしょうか。
LLDBには、Pythonを利用したカスタムコマンドの実装方法についても用意されています。

ここでは、実行中のシミュレータでアプリのBundleパスをMacのFinderで開くというコマンドを作ってみましょう。

```sh
openbundle
```

### 準備

まず実装の前に準備が必要です。
コマンドの実装では、pythonの「lldb」モジュールを使用するのですが、このままではimportできません。
shellで以下のコマンドを実行して、モジュールの場所を探します。

```bash
lldb -P
```

私は以下の場所に存在しました。

```sh
/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Resources/Python
```

このパスを以下のようにPYTHONPATHに追加してimportできるようにしておきます。

```sh
PYTHONPATH=/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Resources/Python
```

### 実装

#### 1. import

まずは、lldbをインポート。
ここでエラーが出る場合は、PYTHONPATHの設定がうまくできていないので、もう一度見直しましょう。

```python
import lldb
```

#### 2. 関数の定義

以下のように五つの引数を受け取る関数を定義しておきます。
`@lldb.command`というデコレータを付加しておくと、このpythonファイルのimport時に自動でコマンド登録が行われるようになります。

```python
@lldb.command('openbundle', doc='実行中のアプリのBundleパスをFinderで開きます')
def handle_command(
        debugger: lldb.SBDebugger,
        command: str,
        exe_ctx: lldb.SBExecutionContext,
        result: lldb.SBCommandReturnObject,
        internal_dict: dict) -> None:
    pass
```

#### 3. Bundleパスの取得

Swiftでは以下のようなコードでBundleパスが取得できますね。

```swift
Bundle.main.bundlePath
```

これをPythonでLLDBを経由して呼び出します。

```python
#　実行したいSwiftスクリプト
script = """
import Foundation
Bundle.main.bundlePath
"""

# フレームを取得
frame: lldb.SBFrame = (
    debugger.GetSelectedTarget()
    .GetProcess()
    .GetSelectedThread()
    .GetSelectedFrame()
)

#　スクリプトを実行して値を取得
value: lldb.SBValue = frame.EvaluateExpression(script)

# パスを文字列として取得
path = value.GetObjectDescription()
```

#### 4. FinderでBundleパスを開く

shellでは以下のフォーマットで、パスをFinderで開くことが可能です。

```sh
open -R <パス>
```

pythonでは、`subprocess`を利用して実行します。

```python
import subprocess

shell = f"open -R {path}"
subprocess.run(shell, shell=True)
```

#### 5. LLDBにカスタムコマンドを読み込み

LLDBで以下のコマンドを実行すると読み込めます。

```sh
command script import <実装したPythonファイルまでのパス>
```

#### 実装したPythonによるカスタムコマンドの全貌

改善点はたくさんありますが、こんな感じです。

```python
import lldb
import subprocess

@lldb.command('openbundle', doc='実行中のアプリのBundleパスをFinderで開きます')
def handle_command(
        debugger: lldb.SBDebugger,
        command: str,
        exe_ctx: lldb.SBExecutionContext,
        result: lldb.SBCommandReturnObject,
        internal_dict: dict) -> None:

    #　実行したいSwiftスクリプト
    script = """
    import Foundation
    Bundle.main.bundlePath
    """

    #　スクリプトを実行して値を取得
    value: lldb.SBValue = frame.EvaluateExpression(script)

    # パスを文字列として取得
    path = value.GetObjectDescription()

    # パスをFinderで開く
    shell = f"open -R {path}"
    subprocess.run(shell, shell=True)
```

# アプリ開発のためのLLDBカスタムコマンド

アプリ開発をより快適にするためのLLDBのカスタムコマンドの機能を色々考えてみました。
実装したものがこちらです。

<https://github.com/p-x9/iLLDB>

## UIの階層を表示する機能

`ui tree`でアプリのUIの階層構造を表示します。

![KeyWindow](https://github.com/p-x9/iLLDB/raw/main/resources/keyWindow-simple.png)

以下のようにさまざまな指定方法に対応しています。

```sh
# 特定のview
ui tree --view {viewのプロパティ名}
ui tree --view {viewのアドレス(0x600000c7e0a0)}

# 特定のViewController
ui tree --vc {vcのプロパティ名}
ui tree --vc {vcのアドレス(0x600000c7e0a0)}

# 特定のWidnow
ui tree --window {windowのプロパティ名}
ui tree --window {windowのアドレス(0x600000c7e0a0)}

# 特定のCALayer
ui tree --layer {layerのプロパティ名}
ui tree --layer {layerのアドレス(0x600000c7e0a0)}
```

実装は[ここ](https://github.com/p-x9/iLLDB/blob/main/src/ui.py)にあります。
<https://github.com/p-x9/iLLDB/blob/main/src/ui.py>

## ファイルの階層を表示する

`file tree`でアプリのファイル構造を表示します。

![FileTree](https://github.com/p-x9/iLLDB/raw/main/resources/file-tree.png)

こんな感じで実行します。

```sh
file tree --bundle
file tree --library
file tree --documents
file tree {path}
```

実装は[ここ](https://github.com/p-x9/iLLDB/blob/78081cc356cc5df44035b34dead5b3e80e38b930/src/file.py#L64-L88)にあります。
<https://github.com/p-x9/iLLDB/blob/78081cc356cc5df44035b34dead5b3e80e38b930/src/file.py#L64-L88>

## 　　シミュレータで実行中のアプリのパスをFinderで開く

`file open`でアプリに関するディレクトリをFinderで開きます。
以下のように実行します。

```sh
file open --bundle
file open --library
file open --documents
file open {path}
```

実装は[ここ](https://github.com/p-x9/iLLDB/blob/78081cc356cc5df44035b34dead5b3e80e38b930/src/file.py#L91-L116)にあります。
<https://github.com/p-x9/iLLDB/blob/78081cc356cc5df44035b34dead5b3e80e38b930/src/file.py#L91-L116>

## その他

他にもUserDefaultsの簡易操作、アプリやデバイス情報の表示、HTTP Cookieの簡易操作などさまざまな機能を実装しています。

# 終わりに

今回はLLDBのカスタムコマンドの実装について書きました。
何かいい機能のアイデアあれば[Issue](https://github.com/p-x9/iLLDB/issues)や[プルリク](https://github.com/p-x9/iLLDB/pulls)ください。

スターください。
