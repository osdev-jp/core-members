# 資料集

OS開発をする上で参考になる資料

These documents will help your operating system development.

## Intel CPU

- [Intel Software Developer’s Manual](http://www.intel.co.jp/content/www/jp/ja/processors/architectures-software-developer-manuals.html) [1]

  Intel CPUの詳細な仕様、命令セットや割り込みの説明などが載っている。OS自作で一歩踏み込む際に必読の書。

- [MultiProcessor Specification](http://www.intel.com/design/archives/processors/pro/docs/242016.htm) [2]

  マルチコア、ハイパースレッディングに関する仕様が乗っている。MPを実装するにはこれに加えてAPICサポートが必須。

- [Intel® 82093AA I/O Advanced Programmable Interrupt Controller (I/O APIC) Datasheet](http://www.intel.com/design/chipsets/datashts/290566.htm) [3]

  I/O APICの仕様書。（Local APICについてはSDM参照）

## Intel Chipset

- [Intel® I/O Controller Hub 8 (Intel® ICH8) Family Datasheet](http://www.intel.co.jp/content/www/jp/ja/io/intel-io-controller-hub-8-datasheet.html) [4]

  各種I/Oデバイスのレジスタマップ等が記載されている。

## Intel NIC

- [Intel 8254x, 8257x](http://draft.scyphus.co.jp/osdev/e1000.html) [5]

  基本的なIntel E1000 NICに関する話。仕様書もリンクから辿れる

- [Intel® I/O Controller Hub 8/9/10 and 82566/82567/82562V Software Developer’s Manual](http://www.intel.com/content/www/us/en/embedded/products/networking/i-o-controller-hub-8-9-10-82566-82567-82562v-software-dev-manual.html) [6]

  チップセット統合型NICの仕様書

- [Intel® I/O Controller Hub 8 LAN NVM: Map and Information Guide](http://www.intel.com/content/www/us/en/ethernet-controllers/i-o-controller-hub-8-lan-nvm-map-appl-note.html) [7]

  NICが参照するNVMに関する資料。ich8 NIC等を実装するときに必要

## PCI

- [PCI Local Bus Specification Revision 3.0](http://www.xilinx.com/Attachment/PCI_SPEV_V3_0.pdf) [8]

  PCIの仕様書

- [PCI Express Base Specification Revision 3.0](http://composter.com.ua/documents/PCI_Express_Base_Specification_Revision_3.0.pdf) [9]

  PCI Expressの仕様書。基本的にPCIとの差異が載っている感じなので、PCIの仕様書も読む必要あり。

- [PCI-to-PCI Bridge Architecture Specification](https://cds.cern.ch/record/551427/files/cer-2308933.pdf) [10]

## ACPI

- [ACPI Spec](http://www.acpi.info/spec.htm) [11]

  ACPIは電源管理のみならず、MP TablesやPCI、各種デバイスの情報を取る事ができる。

## HPET Timer

- [IA-PC HPET (High Precision Event Timers) Specification](http://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/software-developers-hpet-spec-1-0a.pdf) [12]

  高精度タイマー。単体で使っても良いし、Local APIC Timerのような、バスの周波数で動くタイマーの精度測定に使っても良い

## UEFI

- [UEFI Specifications](http://www.uefi.org/specifications) [13]

  BIOSの後継となるファームウエアインターフェイス。UEFI搭載PCでブートローダを書く際には必須

## ハードウエア全般

- [OADG テクニカルリファレンス（ハードウエア）](http://web.archive.org/web/20090815135508/http://www.oadg.or.jp/techref/oadghwd.pdf) [14]

  古いx86マシンのハードウエア全般に関する情報が記載されている。キーボードの情報は有用。
