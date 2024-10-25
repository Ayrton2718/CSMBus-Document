# CSMBusのプロトコル
CSMBusのライブラリは、図のように単一のPCから、複数の`Gateway`基盤（Ethernet to CAN）が接続し、その`Gateway`基盤に複数のCANデバイスが接続することを想定している。`ROS`-`Gateway`間の通信をライブラリで隠蔽することで、ユーザがPC-CANデバイス間の通信をEthernetの通信プログラムを書かずに直接行えるようになっている。

![overview](export_svg/connection_example.svg)


## プロトコルスタック
### ROS2
PC上で動作するcsmbusのプロトコルスタックを解説する。  
`Gateway`との通信にはUDPを用いている（[UDPパケットの詳細](#udp-packet)）。  
ROS2ノードは、`robomas`, `odrive`, `encoder`, `gyro`などをインスタンス化して宣言でき、インスタンスは各CANノードと１対１対応または、多対1対応（複数のインスタンスから単一センサの値を取得する場合）するようになっている。

- `ether csmbus ctrl`: `Gateway`基盤との通信状況確認、セーフティ信号などを扱っている。
- `ether csmbus socket`: UDPパケットの作成・上位層へのパケットのルーティングや、再送制御などを行っている。
- `robomas`: ROS2ノードからの指令値からパケットを作成し、`Gateway`基盤上で動いているロボマスモータの制御プログラムと通信を行う。
- `odrive`: ROS2ノードからの指令値からパケットを作成し、`Gateway`基盤上で動いているOdriveの制御プログラムと通信を行う。
- `user custom app`: `Gateway`上で動いている自作のカスタムアプリと通信を行いたい場合、`ether csmbus socket`の通信インスタンスを作成することで、そのアプリのパケットのみを送受信できる。詳細は[「ROS2-Gateway間のカスタムアプリの作り方」](custom_task.md)。
- `can csmbus io`: `encoder`や`gyro`のCANの通信パケットを集約して通信パケットを作成し、`Gateway`基盤上で動いている`can csmbus`と送受信する。
- `encoder`: encoderの`CAN Device`から送られてくるエンコーダのCANパケットを受信し、ROS2ノードに返す。
- `gyro`: gyroの`CAN Device`から送られてくるジャイロのCANパケットを受信し、ROS2ノードに返す。
- `user custom board`: `can csmbus io`の通信クラスを継承することで、独自のCANパケットを用いてカスタム基盤との通信を行うことができる。詳細は[「ROS2-CAN小基盤上で動作するカスタムタスク作成方法」](custom_task.md)。

- `ether csmbus backdoor`: `Gateway`基盤のエラーや、ROS2を起動するためのパネル基盤のデータなどは、ROS2との通信ポートとは別のポートを用いて取得することができる。

![overview](export_svg/csmbus_ros2_stack.svg)


### Gateway
`Gateway`基盤上で動作するcsmbusのプロトコルスタックを解説する。  
`Gateway`基盤の役割は、Ethernet-CAN間のプロトコル変換と、PC側では間に合わないモータのPID制御などを行うことである。`robomas`や`odrive`、`can csmbus`などのアプリを、使用目的に応じて各CANポートごとにインスタンス化して使用する。

- `ether csmbus ctrl`: ROS2との通信状況確認、セーフティ信号などを扱っている。ROS2との通信が一定期間途切れると各アプリのループを停止し、CANバス上にも停止信号を送信する。
- `ether csmbus socket`: UDPパケットの作成・上位層へのパケットのルーティングを行う。
- `robomas`: ROS2からの出力パケットを受取り、ロボマスモータへのCANパケットを作成する。また、受信したモータのセンサデータをROS2に送信する。制御周期が必要な速度PIDと角度PIDの制御器が各モータに対して動いおり、ROS2からパラメータの変更、制御モードの切り替えを行える。ストール検知してROS2に接続エラーを送信する。
- `odrive`: ROS2からの出力パケットを受取り、OdriveへのCANパケットを作成する。また、受信したモータのセンサデータをROS2に送信する。Odriveのエラーを自動で解消し、原因をROS2に送信する。
- `can csmbus`: ROS2からの通信パケットから、CANパケットを取り出してバスに流す。CANバスから受信したパケットを集約してROS2に送る（[CANパケットの詳細](#can-csmbus-packet)）。
- `ether csmbus can socket`: CANの送受信処理を行う。送信バッファを備えている。
- `user custom app`: `robomas`や`odrive`のように、`Gateway`上で動作させたいプログラムがある場合、`ether csmbus socket`や`ether csmbus can socket`の通信クラスを継承し、各CANポートごとにバインドして使用できるようになっている。詳細は[「ROS2-Gateway間のカスタムアプリの作り方」](custom_task.md)

- `ether csmbus backdoor`: `Gateway`基盤内で発生したエラーやパネル基盤の情報など、ROSが起動していない間もPC側が必要なデータを送信する。

![overview](export_svg/csmbus_gateway_stack.svg)


### CAN Device
CANの小基板上で動作するcsmbusのプロトコルスタックを解説する（[CANパケットの詳細](#can-csmbus-packet)）。  
- `can csmbus io`: 受け取るCAN IDの識別、自分のCAN IDの送信をを行う。Gatewayから定期的に送られてくるセーフティ信号の監視も行っており、エラーやタイムアウトが発生したら、`user task`の`unsafe`関数を実行する。詳細は[「ROS2-CAN小基盤上で動作するカスタムタスク作成方法」](custom_task.md)。

![overview](export_svg/csmbus_device_stack.svg)


## 通信パケット

### UDP Packet
通信プロトコルは記述されている[ec_type.h](https://github.com/Ayrton2718/csmbus/blob/main/src/csmbus/eth/ec_type.h)

To do

### CAN CSMBus Packet
このCAN Packetとは、`can csmbus`専用のパケットで、`robomas`や`odrive`で扱っているCANパケットとは別です。

通信プロトコルは記述されている[cc_type.h](https://github.com/Ayrton2718/csmbus/blob/main/src/csmbus/can/cc_type.h)

To do