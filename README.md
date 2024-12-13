# CSMBus Documents
2024世代のとよはし☆ロボコンズが実際に使用していたライブラリの一部を公開します。
フォーマット等はほとんど行っていないため、読みづらい部分もあると思いますがご了承ください。

## ライブラリへのリンク
- [csmbus for ROS2](https://github.com/Ayrton2718/csmbus/tree/main)  
  CSMBusのROS2パッケージ。Ethernet基盤とUDPパケットの送受信をする。
- [CSMBus-GatewayF107RC](https://github.com/Ayrton2718/CSMBus-GatewayF107RC)  
  Ethernet基盤上で動作するCSMBusライブラリ。ROS2からのUDPパケットをCANパケットに変換する。
- [CSMBus-DriverG431](https://github.com/Ayrton2718/CSMBus-DriverG431)  
  CANの末端基盤のプログラム。Ethernet基盤とCANパケットの送受信をする。
- [BlackBox](https://github.com/Ayrton2718/blackbox)  
  ROS2上で動作するデバッグ用のROS2パッケージ。
- [robocon-os](https://github.com/Ayrton2718/robocon-os)  
  CSMBus、BlackBoxの環境構築を行えるROS2のワークスペース。
- [ABU2024_DriverG431](https://github.com/Ayrton2718/ABU2024_DriverG431)  
  2024世代の豊橋で実際に使用していたDriverのプログラム。ブランチと各種基盤のプログラムが対応している。

## Documents
CSMBusのプロトコルやライブラリの使用方法についてまとめてあります。
- [プロトコル](docs/protocol.md)
- [カスタムタスクの作成方法](docs/custom_task.md)

## Acknowledgments
- **[HIYryuni](https://github.com/HIYryuni)**, [TUT_RC]  
  The circuit design used in this project is based on their innovative work.

- **[eyr1n](https://github.com/eyr1n)**, [TUT_RC]  
  The creator of the **tutrc_ament** library.

- **[Syunima](https://github.com/Syunima)**, [NIT_OITA]  
  For inspiring the concept behind the circuit design in this project. 
