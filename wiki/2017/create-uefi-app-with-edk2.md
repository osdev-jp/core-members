2017/06/15 by uchan

# EDK II で UEFI アプリケーションを作る

この記事は UEFI アプリケーションを EDK II 上で作り、QEMU で動かすまでを解説します。動作確認は Ubuntu 16.04 でやっていますが、Linux ならどれも同じ方法で動作すると思います。

<!--more-->

## UEFI アプリを作る 3 つの方法

UEFI アプリを作るには、使う SDK によって主に 3 つの方法があります。参考記事：[UEFIのSDK事情 - syuu1228's blog](http://syuu1228.hatenablog.com/entry/20130130/1359552753)

- SDK を使わない
- gnu-efi
- EDK II

SDK を使わないで作る方法は [ツールキットを使わずに UEFI アプリケーションの Hello World! を作る - 品川高廣（東京大学）のブログ](http://d.hatena.ne.jp/shina_ecc/20140819/1408434995) が参考になります。Hello World アプリを作るだけなら、恐らくこの方法が最も簡単だと思います。新しく買った UEFI 搭載の組み込みボードの動作をサクッと確認したい、みたいな用途にはちょうどいいでしょう。しかし、必要となるすべての構造体を自分で定義する必要があったり、頼れるライブラリもないので、発展性は低いです。

gnu-efi は UEFI API を叩くアプリケーションを作るためのものです。使い方が結構シンプルなようで、これを用いたサンプルを多く見かけます。ただ、命名規則 や ABI が UEFI の公式とはちょっと違ったり、UEFI で規定されている API のうちよく使うものしか実装されてなかったりして、使いこなすうちに物足りなくなることもあると教えてもらいました。gnu-efi については id:tnishinaga さんの記事 [gnu-efiを使ってAARCH64/ARM64のUEFIサンプルアプリを動かしてみる](http://tnishinaga.hatenablog.com/entry/2017/06/20/191955) が参考になるでしょう。

[EDK II](http://www.tianocore.org/edk2/) は元々、UEFI の開発に深くかかわっている Intel が作っていた SDK です。gnu-efi が「UEFI アプリケーション」専用なのに対し、EDK II は UEFI アプリはもちろん、周辺のライブラリや、UEFI ファームウェアそのものを開発するための SDK という役割も持っており、超高機能です。高機能ゆえに UEFI アプリを作るという目的のためには複雑すぎてとっつきにくい印象があります。が、フル装備の SDK ですから、慣れておけば後々困ることもないと思います。ということで、この記事では EDK II への入門を目指します。

## EDK II の入手とセットアップ

EDK II SDK は GitHub から入手します。参照：[EDK II 公式サイト](http://www.tianocore.org/edk2/)

    $ git clone https://github.com/tianocore/edk2.git

入手できたら edk2 ディレクトリに移動し、edksetup.sh を読み込みます。

    $ cd edk2
    $ source edksetup.sh

`source edksetup.sh BaseTools` を実行するように書いてある解説ブログなどがあるのですが、現在のバージョンの EDK II では `BaseTools` を付けないのが正しいです（付けたとしても単に無視されます）。

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

```eval_rst
===================  ======
設定項目             設定値
===================  ======
``ACTIVE_PLATFORM``  ビルド対象のパッケージの .dsc ファイル。
``TARGET``           DEBUG、RELEASE、NOOPT、UserDefined のいずれか。NOOPT は最適化をせずビルドするオプションらしい。
``TARGET_ARCH``      どのアーキテクチャ向けのバイナリを作るか。IA32 とか X64 とか ARM とか。
``TOOL_CHAIN_TAG``   tools_def.txt の "Supported Tool Chains" からビルドに用いるツールチェインを選ぶ。VS2015 とか GCC5 とか。
===================  ======
```

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

個々のモジュールの名前、モジュールの種別、構成するソースコードなどを指定するファイルです。Hello World モジュールの例を示します。

    [Defines]
      INF_VERSION                    = 0x00010006
      BASE_NAME                      = Hello
      FILE_GUID                      = 430e7e71-c5af-408d-aa22-795ee53ae750
      MODULE_TYPE                    = UEFI_APPLICATION
      VERSION_STRING                 = 0.1
      ENTRY_POINT                    = UefiMain

    [Sources]
      Hello.c

    [Packages]
      MdePkg/MdePkg.dec

    [LibraryClasses]
      UefiLib
      UefiApplicationEntryPoint

Defines セクションはまあ今まで見てきたのから類推できると思います。`BASE_NAME` はモジュール名を指定します。`FILE_GUID` は例にもれず `uuiegen` で生成します。

`MODULE_TYPE` はモジュールの種別を設定します。edk2 のリポジトリを眺めてみると、取り得る値として `UEFI_APPLICATION`, `DXE_DRIVER`, `UEFI_DRIVER`, `BASE`, `PEIM` などがあることが分かります。それぞれどんな意味なのか筆者はよく分かっていませんが、UEFI アプリを作りたい場合は `UEFI_APPLICATION` にしておけばいいはずです。

`ENTRY_POINT` にはメイン関数となる関数名を設定します。ここに設定した名前の関数がアプリケーションの開始点となります。基本的にどんな名前でもいいのですが、`EfiMain` だけは使えないようです。それ以外の名前にしてください。ここではよくサンプルで見かける `UefiMain` という名前にしてみました。

Souces セクションにはこのモジュールを構成するソースコードを列挙します。

Packages セクションでは、このモジュールが依存するパッケージを列挙します。ここに列挙されたパッケージからヘッダファイルが検索されてモジュールのコンパイルに利用されます。

LibraryClasses セクションでは、このモジュールが依存するライブラリを列挙します。Packages セクションと役割は若干似ていますが、LibraryClasses セクションの情報はリンクするライブラリを特定するのに使われます。

ここでは `UefiLib` と `UefiApplicationEntryPoint` をリンクするよう指示しています。前者は `Print` 関数を利用するために必要なライブラリです。後者はスタンドアロンの UEFI アプリを構成するのに必要なライブラリです。

スタンドアロンの UEFI アプリは UEFI Shell が起動する前に起動することができます。OS のブートローダーを作る場合などはスタンドアロンとして構成します。ところで `UefiApplicationEntryPoint` と似たようなライブラリに `ShellCEntryLib` があり、例えば AppPkg/Applications/Hello/Hello.inf で使用されています。`UefiApplicationEntryPoint` の代わりにこのライブラリをリンクすると、作成したアプリは UEFI Shell から起動するタイプのアプリとなります。この 2 つのライブラリは同名の関数を提供しているので、どちらかしかリンクすることはできません。

## Hello.c

おっと、Hello World を作るのに最後の大事なピースを紹介し忘れるところでした。ソースコードです。

    #include  <Uefi.h>
    #include  <Library/UefiLib.h>

    EFI_STATUS
    EFIAPI
    UefiMain (
        IN EFI_HANDLE ImageHandle,
        IN EFI_SYSTEM_TABLE *SystemTable
        )
    {
      Print(L"Hello EDK II.\n");
      while (1);
      return 0;
    }

スタンドアロンの UEFI アプリのエントリポイントは一般的な C 言語プログラムと違い、`EFI_HANDLE` と `EFI_SYSTEM_TABLE *` を引数として受け取ります。EDK II に含まれる CryptoPkg/Application/Cryptest/Cryptest.c はスタンドアロンのアプリのサンプルソースであり、エントリポイントの引数の型と戻り値の型は同じですね。

`Print` 関数を使うには `Library/Uefilib.h` のインクルードと UefiLib のリンクが必要です。品川先生のブログを読むと、`Print` 関数を使わないで文字列を表示する方法が紹介されています。実験してみると面白いかもしれません。

## ビルド

さて、ここまできたら target.txt に設定してビルドすることができます。Ubuntu 16.04 であれば、次のような設定値にすれば良いでしょう。

```eval_rst
===================  ======
設定項目             設定値
===================  ======
``ACTIVE_PLATFORM``  MyPkg
``TARGET``           RELEASE
``TARGET_ARCH``      X64
``TOOL_CHAIN_TAG``   GCC5
===================  ======
```

この設定が完了したら、edksetup.sh を読み込んだシェル上で `build` コマンドを実行します。

    $ build
    Build environment: Linux-4.4.0-66-generic-x86_64-with-Ubuntu-16.04-xenial
    Build start time: 08:24:03, Jun.14 2017

    WORKSPACE        = /home/uchan/workspace/github.com/tianocore/edk2
    ECP_SOURCE       = /home/uchan/workspace/github.com/tianocore/edk2/EdkCompatibilityPkg
    EDK_SOURCE       = /home/uchan/workspace/github.com/tianocore/edk2/EdkCompatibilityPkg
    EFI_SOURCE       = /home/uchan/workspace/github.com/tianocore/edk2/EdkCompatibilityPkg
    EDK_TOOLS_PATH   = /home/uchan/workspace/github.com/tianocore/edk2/BaseTools
    CONF_PATH        = /home/uchan/workspace/github.com/tianocore/edk2/Conf


    Architecture(s)  = X64
    Build target     = RELEASE
    Toolchain        = GCC5

    Active Platform          = /home/uchan/workspace/github.com/tianocore/edk2/MyPkg/MyPkg.dsc

    ...中略...

    - Done -
    Build end time: 08:24:05, Jun.14 2017
    Build total time: 00:00:03

最後が Done となればビルド完了です。失敗時は Failed となります。

## テスト実行

ビルド成果物は Build/MyPkgX64/RELEASE_GCC5/X64/Hello.efi にあるはずです。これを QEMU で起動させる方法をご紹介します。

### ファームウェア

QEMU で UEFI アプリを動作させるには、OVMF と呼ばれる UEFI ファームウェアを入手する必要があります。幸いなことに EDK II は OVMF を含んでいるので、我々はすぐにビルドすることができます。

OVMF をビルドするには、target.txt で `ACTIVE_PLATFORM` に `OvmfPkg/OvmfPkgX64.dsc` を指定します。それで `build` コマンドを実行するだけです。ビルドが終わると Build/OvmfX64/RELEASE_GCC5/FV/OVMF.fd に目的の OVMF ファームウェアが生成されているはずです。

Build/OvmfX64/RELEASE_GCC5/FV ディレクトリには OVMF.fd の他に OVMF_CODE.fd と OVMF_VARS.fd があります。この 3 つのファイルについて説明します。

UEFI の設定画面を見たことがあると思いますが、いろいろな設定項目がありますよね。それらの設定値はマザーボードの不揮発性メモリ（NVRAM）に保存されますが、その保存領域と UEFI のプログラム部分を一緒くたにしたファイルが OVMF.fd です。分離したものが OVMF_CODE.fd と OVMF_VARS.fd となります。

OVMF_VARS.fd が分離されていることで、UEFI_CODE.fd は共通化しつつ、OVMF_VARS.fd を複数用意してデバッグ対象別に UEFI の設定を変えるなどということが可能です。OVMF_VARS.fd は OVMF_CODE.fd に比べてとても小さいので、ディスク容量の節約になりますね。（と言っても OVMF_CODE.fd は 2MB 程度なので、それほど大きいわけでもないのですけどね）

UEFI に詳しい人によると、分離された形式のほうが新しい形式なので、OVMF.fd という古いものを使うよりオススメということでした。こだわりがなければ従っておきましょう（笑）

### ディスクイメージ

さて、これでファームウェアの準備はできました。次は自作した UEFI アプリを含む、FAT でフォーマットされたディスクイメージを作成します。といっても、実際にディスクイメージを作るのはちょっと面倒なので、ここでは QEMU の便利な機能を使います。それは、あるディレクトリ以下を仮想的にディスクとして扱えるようにする機能です。

ディスクのルートディレクトリとして振る舞うディレクトリを用意します。名前はなんでもいいですが、ここでは image とします。そして、自作の UEFI アプリを自動起動させるためには、/EFI/BOOT/BOOTX64.efi としてその UEFI アプリを配置する必要があります。具体的なコマンドを示します。

    $ mkdir -p image/EFI/BOOT
    $ cp Build/MyPkgX64/RELEASE_GCC5/X64/Hello.efi image/EFI/BOOT/BOOTX64.efi

これで、QEMU に与えるディスク（として振る舞うディレクトリ）の準備ができました。次に OVMF_VARS.fd をコピーしておきます。なぜなら、UEFI の設定変更で書き換わってしまうからです。

    $ cp Build/OvmfX64/RELEASE_GCC5/FV/OVMF_VARS.fd ./

ここまでできたら QEMU を起動させてみましょう。

    $ qemu-system-x86_64 \
    -drive if=pflash,format=raw,readonly,file=Build/OvmfX64/RELEASE_GCC5/FV/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=./OVMF_VARS.fd \
    -drive if=ide,file=fat:image,index=0,media=disk

drive 指定はちょっと複雑ですね。詳しくは `man qemu-system-x86_64` としてマニュアルを読んでみてください。取り得る値など詳しく書いてあります。簡単に説明するとこんな感じです。

- if: インターフェースを指定します。
    - pflash は「パラレルフラッシュ」のことです。要は基板上に実装されたファームウェアを入れるフラッシュメモリのことです。
    - ide は IDE インターフェースと言って、2010 年ごろまで使われていた古い規格です。ハードディスクや CD-ROM をパラレルケーブルで接続します。
    - 他には scsi や floppy なども指定できるみたいです。
- format: ディスクイメージのフォーマットを指定します。
    - raw は単純なディスクイメージであることを示します。ディスクの中身がそのままファイルになったものです。
    - 他には qcow2 などのフォーマットもあります。大容量のディスクイメージ用途などによく使われます。
    - QEMU はディスクイメージからフォーマットを推測する機能があるので、raw 以外の場合は指定を省略することが多い印象です。
    - オプションはカンマ区切りで複数指定できます。OVMF_CODE.fdは書き換わることはないので読み取り専用としています。
- file: ディスクイメージのファイル名を指定します。
    - 基本的にはファイルへのパスを指定します。
    - `fat:` オプションを指定すると、あるディレクトリ以下を FAT フォーマットのディスクであるかのように扱うことができます。
    - `fat:rw:` というように読み書き可能オプションを加えると、指定したディレクトリ以下に書き込みができるようになります。
    - `fat:image` であれば、image ディレクトリの内容を FAT ディスクとして使うという意味になります。
- index: ディスクをつなぐポート番号を指定します。
    - `if=ide` のディスクの場合、index 0, 1, 2, 3 はマスターとスレーブ、プライマリとセカンダリの違いになるのだと思います。（この辺の用語は聞かなくなって久しいので、若い人は知らないかもしれません）
    - `if=floppy` の場合、index 0 は A ドライブ、1 は B ドライブとなります。
- media: disk か cdrom を指定します。

さて、上記のコマンドを実行すると QEMU が立ち上がります。画面に "TianoCore" というロゴが表示され、しばらく待つと `Print` 関数に指定したメッセージが表示されるはずです。これでめでたく EDK II による UEFI アプリの作成と QEMU による実験のやり方が分かりました。おめでとうございます！

## UEFI Shell

UEFI を実験する場合に UEFI Shell を使いたくなることがあります。UEFI Shell とは UEFI に組み込まれているシェルで、基本的には `ShellCEntryLib` をリンクした UEFI アプリを起動するためのものです。また、接続されたディスクのファイルを確認したり、システムのメモリマップを取得したり、パーティションプログラムを実行したりなど、いろいろな機能が備わっています。

QEMU を起動させて何もしないでいると、自動で BOOTX64.efi が起動してしまいます。UEFI Shell を使用するためには、QEMU 起動直後に F2 を連打し、ブートメニューを表示させ、"Boot Manager" メニューを選び、"EFI Internal Shell" を選びます。

UEFI Shell の使い方については [UEFI シェル - ArchWiki](https://wiki.archlinuxjp.org/index.php/Unified_Extensible_Firmware_Interface#UEFI_.E3.82.B7.E3.82.A7.E3.83.AB) が詳しいです。

## 参考になる情報

EDK II は UEFI の規格に基づいて作られていますから，各種 API の仕様などは UEFI 規格書を見ると詳しく載っています。 https://uefi.org/specifications から "UEFI Specification Version X.Y" をダウンロードして読んでください。
