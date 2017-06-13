2017/06/13 by uchan

# EDK II で UEFI アプリケーションを作る

この記事は UEFI アプリケーションを EDK II 上で作り、QEMU で動かすまでを解説します。動作確認は Ubuntu 16.04 でやっていますが、Linux ならどれも同じ方法で動作すると思います。

<!--more-->

## UEFI アプリを作る 3 つの方法

UEFI アプリを作るには、使う SDK によって主に 3 つの方法があります。参考記事：[UEFIのSDK事情 - syuu1228's blog](http://syuu1228.hatenablog.com/entry/20130130/1359552753)

- SDK を使わない
- gnu-uefi
- EDK II

SDK を使わないで作る方法は [ツールキットを使わずに UEFI アプリケーションの Hello World! を作る - 品川高廣（東京大学）のブログ](http://d.hatena.ne.jp/shina_ecc/20140819/1408434995) が参考になります。Hello World アプリを作るだけなら、恐らくこの方法が最も簡単だと思います。新しく買った UEFI 搭載の組み込みボードの動作をサクッと確認したい、みたいな用途にはちょうどいいでしょう。もちろん、必要となるすべての構造体を自分で定義する必要があったり、頼れるライブラリもないので、発展性は著しく低いですが。

gnu-uefi は UEFI API を叩くアプリケーションを作るためのものです。使い方が結構シンプルなようで、これを用いたサンプルを多く見かけます。ただ、命名規則 や ABI が UEFI の公式とはちょっと違ったり、UEFI で規定されている API のうちよく使うものしか実装されてなかったりして、使いこなすうちに物足りなくなることもあると教えてもらいました。

[EDK II](http://www.tianocore.org/edk2/) は UEFI の開発に深くかかわっている Intel が元々作っていた SDK です。gnu-efi が「UEFI アプリケーション」専用なのに対し、EDK II は UEFI アプリはもちろん、周辺のライブラリや、UEFI ファームウェアそのものを開発するための SDK という役割も持っており、超高機能です。高機能ゆえに UEFI アプリを作るという目的のためには複雑すぎてとっつきにくい印象があります。が、フル装備の SDK ですから、慣れておけば後々困ることもないと思います。ということで、この記事では EDK II への入門を目指します。

## EDK II の入手とセットアップ

EDK II SDK は GitHub から入手します。参照：[EDK II 公式サイト](http://www.tianocore.org/edk2/)

    $ git clone https://github.com/tianocore/edk2.git

入手できたら edk2 ディレクトリに移動し、edksetup.sh を読み込みます。

    $ cd edk2
    $ source edksetup.sh

`source edksetup.sh BaseTools` を実行するように書いてある解説ブログなどがあるのですが、現在のバージョンの EDK II では `BaseTools` 付けないのが正しいです。（付けたとしても単に無視されます。）

edksetup.sh が読み込まれると幾つかの環境変数が設定された旨が表示されます。以降のビルド作業などは、edksetup.sh を読み込んだシェル上で行ってください。

## EDK II のファイル構造

ここで EDK II のファイル構造を眺めておきましょう。この知識はアプリを作るときに役立つはずです。（なお、以下で登場する `$WORKSPACE` は git clone した edk2 ディレクトリへのパスを指します。）

    $WORKSPACE/
      Conf/
        target.txt      ビルド設定
        tools_def.txt   ツールチェーンの設定
      Build/            ビルド成果物が出力されるディレクトリ
      MdePkg/           EDK の中心的ライブラリのパッケージディレクトリ
      ...Pkg/           その他のパッケージディレクトリ
      edksetup.sh       環境変数設定用スクリプト

Conf ディレクトリにある target.txt は `build` コマンドで何がビルドされるかを決定するファイルです。詳しい説明は後述します。

Build ディレクトリは `build` コマンドによるビルド成果物が出力されるディレクトリです。例えば AppPkg をビルドすると Build/AppPkg/DEBUG_GCC5/X64/Hello.efi のような場所に目的の EFI アプリが出力されます。

Conf と Build を除くと、その他のディレクトリは「パッケージ」ごとに分かれています。多くのパッケージは別のパッケージの機能を利用して実装されており、パッケージ間で依存関係を持っています。ただし MdePkg だけは他のパッケージに依存しておらず、その他のパッケージから利用される基本的なライブラリとして構成されています。

次に、AppPkg の構造を見てみます。

    AppPkg/
      AppPkg.dec        パッケージ宣言（declaration）ファイル
      AppPkg.dsc        パッケージ記述（description）ファイル
      Applications/
        Hello/          Hello モジュールディレクトリ
          Hello.c
          Hello.inf     モジュール定義ファイル

パッケージとして構成するのに必要なのはパッケージ宣言ファイルとパッケージ記述ファイルです。パッケージ宣言ファイルは、パッケージ名を定義したり、パッケージ内のソースコードから利用する定数を定義したりするのに使います。パッケージ記述ファイルは、出力ディレクトリ名やサポートされるアーキテクチャ、サポートされるビルドターゲットなど、ビルドに関する設定を含みます。これら 2 つのファイルについて、詳しくは後述します。

Applications ディレクトリは UEFI アプリケーションを格納するためのディレクトリです。名前は慣習的なもので、AppPkg.dsc の Components セクションのパスを修正すれば別の名前のディレクトリでも問題ないはずです。

Hello.inf は Hello モジュールの定義ファイルです。このファイルにはモジュールの名前やモジュールの種別（ライブラリ、ドライバ、アプリ）、そのモジュールを構成するソースコード名のリストなどが書かれています。このファイルについても詳しくは後述します。

## target.txt

target.txt を開くといくつかの設定項目があることが分かります。重要な設定項目を説明します。

| 設定項目 | 設定値 |
|---------|--------|
| `ACTIVE_PLATFORM` | ビルド対象のパッケージの .dsc ファイル。 |
| `TARGET` | DEBUG、RELEASE、NOOPT、UserDefined のいずれか。NOOPT は最適化をせずビルドするオプションらしい。 |
| `TARGET_ARCH` | どのアーキテクチャ向けのバイナリを作るか。IA32 とか X64 とか ARM とか。 |
| `TOOL_CHAIN_TAG` | tools_def.txt の "Supported Tool Chains" からビルドに用いるツールチェインを選ぶ。VS2015 とか GCC5 とか。 |

## 独自パッケージの作成

さて、いよいよ自作の UEFI アプリの作り方を説明します。新しいアプリを作るには主に 2 つの方法があります。1 つは AppPkg/Applications にディレクトリを増やすことで作る方法、もう 1 つは独自のパッケージを作る方法です。ここでは後者を説明します。

参考記事：[技術者見習いの独り言: UEFIアプリケーション/ドライバー開発の話](http://orumin.blogspot.jp/2014/01/uefi.html)

独自パッケージを作るには新しいディレクトリと .dec/.dsc ファイルが必要です。また、UEFI アプリを構成するモジュールを作る必要があり、.inf ファイルが必要です。既存のパッケージやモジュールのファイルを参考に書いていただければいいのですが、慣れないと難しいのでそれぞれの書き方を説明します。

## .dec : パッケージ宣言ファイル

パッケージ名とその GUID を設定するのが主目的のファイルです。ほぼ最小のファイルは次のようになります。

    [Defines]
      DEC_SPECIFICATION              = 0x00010005
      PACKAGE_NAME                   = MyPkg
      PACKAGE_GUID                   = d6a78c81-0d7f-4b46-af1d-e6ed44e5ffaa
      PACKAGE_VERSION                = 0.1

`PACKAGE_GUID` は `uuidgen` コマンドで生成したものに置き換えてください。`DEC_SPECIFICATION` は多分、.dec ファイルのフォーマットのバージョン番号だろうと思います。他のパッケージの設定値と同じにしておけば問題ないと思います。

`PACKAGE_NAME` と `PACKAGE_VERSION` はご自由にどうぞ。

## .dsc : パッケージ記述ファイル

主にパッケージのビルド設定を書くファイルです。ほぼ最小のファイルは次のようになります。

    [Defines]
      PLATFORM_NAME                  = MyPkg
      PLATFORM_GUID                  = 75dd174f-907b-4f8f-8afa-f0029d98ed5a
      PLATFORM_VERSION               = 0.1
      DSC_SPECIFICATION              = 0x00010005
      OUTPUT_DIRECTORY               = Build/MyPkg$(ARCH)
      SUPPORTED_ARCHITECTURES        = IA32|X64
      BUILD_TARGETS                  = DEBUG|RELEASE|NOOPT

      DEFINE DEBUG_ENABLE_OUTPUT     = FALSE

    [LibraryClasses]
      # Entry point
      UefiApplicationEntryPoint|MdePkg/Library/UefiApplicationEntryPoint/UefiApplicationEntryPoint.inf

      # Common Libraries
      BaseLib|MdePkg/Library/BaseLib/BaseLib.inf
      BaseMemoryLib|MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf
      PcdLib|MdePkg/Library/BasePcdLibNull/BasePcdLibNull.inf
      UefiBootServicesTableLib|MdePkg/Library/UefiBootServicesTableLib/UefiBootServicesTableLib.inf
      !if $(DEBUG_ENABLE_OUTPUT)
        DebugLib|MdePkg/Library/UefiDebugLibConOut/UefiDebugLibConOut.inf
        DebugPrintErrorLevelLib|MdePkg/Library/BaseDebugPrintErrorLevelLib/BaseDebugPrintErrorLevelLib.inf
      !else   ## DEBUG_ENABLE_OUTPUT
        DebugLib|MdePkg/Library/BaseDebugLibNull/BaseDebugLibNull.inf
      !endif  ## DEBUG_ENABLE_OUTPUT

      UefiLib|MdePkg/Library/UefiLib/UefiLib.inf
      PrintLib|MdePkg/Library/BasePrintLib/BasePrintLib.inf
      MemoryAllocationLib|MdePkg/Library/UefiMemoryAllocationLib/UefiMemoryAllocationLib.inf
      UefiRuntimeServicesTableLib|MdePkg/Library/UefiRuntimeServicesTableLib/UefiRuntimeServicesTableLib.inf
      DevicePathLib|MdePkg/Library/UefiDevicePathLib/UefiDevicePathLib.inf

    [Components]
      MyPkg/Hello/Hello.inf

最小といいつつ結構大きいですね。各部分について説明していきます。

### Defines セクション

Defines セクションはビルドの全体的な設定です。

たくさん出てくる「プラットフォーム」とは、どうやら「パッケージ」とほぼ同じ意味らしいです。取り敢えず筆者はパッケージ＝プラットフォームという認識で居ますが、今のところ困っていません。

`PLATFORM_GUID` は例のごとく `uuidgen` コマンドで生成した値を設定してください。`DSC_SPECIFICATION` は他の .dsc ファイルを参考に。`PLATFORM_NAME` と `PLATFORM_VERSION` はご自由に。

`OUTPUT_DIRECTORY` はどこに成果物を書き出すかを指定します。`$WORKSPACE` を基準としたパスを書きます。

`SUPPORTED_ARCHITECTURES` と `BUILD_TARGETS` はそれぞれ、このパッケージがサポートするアーキテクチャとビルドターゲットの選択肢を列挙します。この選択肢から選んだ値を target.txt の `TARGET_ARCH` と `TARGET` に設定することになります。

### LibraryClasses セクション

LibraryClasses セクションでは、ライブラリ名と実際に使用するライブラリ実体のマッピングを定義します。同じような機能を提供するライブラリが複数存在する場合、ここのマッピングを切り替えることでどちらのライブラリを使用するかを選択できます。例えば

    DebugLib|MdePkg/Library/UefiDebugLibConOut/UefiDebugLibConOut.inf

であれば、DebugLib という名前で MdePkg/Library/UefiDebugLibConOut/UefiDebugLibConOut.inf というライブラリを指定しています。パッケージ内のモジュール（今回の例で言えば Hello モジュール）が DebugLib という名前でライブラリを参照したとき、実際には UefiDebugLibConOut.inf ライブラリが使用される、というわけです。

上記の代わりに次のように指定したとします。

    DebugLib|MdePkg/Library/BaseDebugLibNull/BaseDebugLibNull.inf

すると、モジュールは相変わらず DebugLib を参照していても、その実体は BaseDebugLibNull.inf に切り替わります。

例の .dsc ファイルでは、このマッピングの仕組みと条件式を組み合わせて使っています。`DEBUG_ENABLE_OUTPUT` が真の場合、DebugLib は本来のデバッグライブラリを指し、偽の場合は空のライブラリに切り替わります。`DEBUG_ENABLE_OUTPUT` 自体は Defines セクションで定義された定数です。

LibraryClasses セクションには、そのパッケージ内のモジュールが必要とするライブラリを、依存関係も含めてすべて列挙する必要があります。そのため、Hello World でさえ多くのライブラリの列挙が必要になってしまっています。慣れましょう。

### Components セクション

ここは単にパッケージに含まれるモジュール（ライブラリ、ドライバ、アプリ）の .inf ファイルを列挙します。列挙したものがビルドされる、という理解で良いと思います。

## .inf : モジュール定義ファイル

ENTRY_POINT UefiMain, EfiMain 挙動が違う話
