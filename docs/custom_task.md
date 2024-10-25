
## ライブラリの使用方法
### ROS2-Gateway間で動作するカスタムタスク作成方法
#### ROS2側
参考元：https://github.com/Ayrton2718/csmbus/blob/main/src/csmbus/robomas.cpp

送受信したいパケットは、このように書いて、ROS2とGateway両方で宣言しておく
```c++
typedef struct{
    uint8_t mode;
    int16_t cur;
    int16_t rpm;
    int64_t ang;
}__attribute__((__packed__)) Robomas_power_t;
```

初期化:  
appid（アプリIDを指定して、Socketを宣言する。appidは、enumなので自分でROS2の追記する。
```c++
static ECSocket_t g_sock; // グローバル宣言等

g_sock = ECSocket_connect(ECEther_appid_ROBOMAS);
```

送信:  
`ECSocket_addr_t`に送信先のGatewayのID, portを格納し、データ識別用のレジスタ番号を格納する。  
`ECSocket_send`と`ECSocket_sendAck`があり、`ECSocket_sendAck`はパケットドロップ時の再送機能がついているが、ACKを待つ時間が発生するため、出力値は`ECSocket_send`、パラメータは`ECSocket_sendAck`などの使い分けが必要。
```c++
void send_param(ECId_t gw_id, ECPort_t port, id_t number, Robomas_param_t* param)
{
    ECSocket_addr_t addr;
    addr.id = gw_id;
    addr.port = port;
    addr.reg = (ECReg_t)((int)ECReg_8 + (int)number);

    // パラメータなのでACK付きで送信
    ECSocket_sendAck(g_sock, addr, param, sizeof(Robomas_param_t));
}
```

受信スレッド:  
`ECSocket_recv`はブロッキング関数で、受信すると`true`を返す。
`ECSocket_addr_t`には、送信元のGatewayのIDやポート番号、レジスタ番号が格納されている。  
レジスタ番号は、パケットの種類を区別したいときに使える。  
```c++
while(1)
{
    // パケットの受信
    ECSocket_addr_t addr;
    uint8_t buff[ECTYPE_PACKET_MAX_SIZE];
    size_t len;
    if(ECSocket_recv(g_sock, &addr, buff, &len))
    {
        ECId_t id = addr.id;        // 送信元GatewayのID
        ECPort_t port = addr.port;  // 送信元Gatewayのポート番号
        ECReg_t reg = addr.reg;     // 送信元Gatewayのレジスタ番号

        // レジスタ番号の確認とデータサイズの確認
        if(ECReg_0 == reg && (len == sizeof(Robomas_sensor_t) * 6))
        {
            // 型変換
            Robomas_sensor_t* sens = (Robomas_sensor_t*)buff;
        }
    }
}
```

#### Gateway側
参考元:https://github.com/Ayrton2718/CSMBus-GatewayF107RC/blob/main/Core/Src/app/robomas.hpp  

初期化：
```c++
// AppBaseを継承する
class Robomas : public AppBase
{
public:
    // AppBaseにロボマスインスタンスのポート番号を登録する
    Robomas(ECPort_t port) : AppBase(port)
    {
    }

    // init関数をオーバライドして、setup_callbacks関数を読ぶ
    void init(void)
    {
        this->setup_callbacks(ECEther_appid_ROBOMAS);
    }
};
```

送信： 
```c++
// 送信データの作成
Robomas_sensor_t send_data[6];
for(size_t i = 0; i < 6; i++)
{
    _mot[i].filt_cur += ROBOMAS_CUR_LP_COEFF * (_mot[i].now_cur - _mot[i].filt_cur);
    _mot[i].sens.cur = (int16_t)_mot[i].filt_cur;

    send_data[i] = _mot[i].sens;
    _mot[i].sens.is_received = 0;
}

// データ識別用のレジスタ番号と、送信データ、送信サイズを指定する。
this->ether_send(ECReg_0, send_data, sizeof(Robomas_sensor_t) * 6);
```

受信:
```c++
// eth_callback関数をオーバーライドすることで、パケット受信時に呼び出される
void eth_callback(ECReg_t reg, const void* data, size_t len)
{
    // レジスタの確認やパケットサイズの確認をする
    if(ECReg_8 <= reg && reg <= ECReg_13 && len == sizeof(Robomas_param_t))
    {
        uint8_t num = reg - ECReg_8;
        _mot[num].param = *((const Robomas_param_t*)data);
    }
}
```

### ROS2-CAN小基盤上で動作するカスタムタスク作成方法
#### ROS2
参考元：https://github.com/Ayrton2718/csmbus/blob/main/src/csmbus/switch.hpp, https://github.com/Ayrton2718/csmbus/blob/main/src/csmbus/led_tape.hpp

送受信したいパケットは、このように書いて、ROS2と小基盤両方で宣言しておく（最大8バイトなので注意）。
```c++
typedef struct{
    uint8_t list1;
}__attribute__((__packed__)) switch_t;
```

初期化：  
```c++
// can_csmbus::Deviceを継承する
class Switch : protected can_csmbus::Device
{
public:
    // GatewayのID、GatewayのCANポート、CAN ID
    void init(ECId_t gw_id, ECPort_t port, id_t id)
    {
        // can_csmbus::Deviceの初期化
        this->dev_init(gw_id, port, id);
    }
};
```

受信:
```c++
private:
    // recvレジスタの宣言（受信したいレジスタ番号、型名）
    RecvRegister<CCReg_0, switch_t> _switch_reg;

    void init(ECId_t gw_id, ECPort_t port, id_t id)
    {
        // _switch_regの初期化
        switch_t sw;
        for(size_t i = 0; i < 7; i++)
        {
            set_sw(&sw.list1, i, false);
        }
        _switch_reg.init(sw);
    }

    // オーバライドする
    virtual void can_callback(CCReg_t reg, uint8_t len, const uint8_t* data)
    {
        // _switch_regを呼び出す
        _switch_reg.can_cb(reg, len, data);
    }
```

送信:
```c++
private:
    // sendレジスタの宣言（送信したいレジスタ番号、型名）
    send_reg_t<ECReg_0, led_tape_t> _rgb_reg;

    
    void send(void)
    {
        led_tape_t rgb;
        // thisポインタと送信データ
        _rgb_reg.send(this, rgb);
    }
```

#### CAN小基盤
参考:https://github.com/tutrobo/ABU2024_DriverG431/blob/dev-SWITCH/Core/Src/user_task.cpp

参考:https://github.com/tutrobo/ABU2024_DriverG431/blob/dev-WS2812/Core/Src/user_task.cpp

## その他
### 基盤のID管理
`Gateway`と`CAN Device`のIDのcsmbusライブラリは、IDの変更・保存を行う機能が搭載されている。  
基板上にID変更用のタクトスイッチがあり、ロボマスモータのように、長押しのあとに押された回数がIDとなる。IDは、STM32の内蔵フラッシュに保存され、再起動後もIDを記憶する。  
`Gateway`ではIPアドレスとMACアドレスが変化し、`CAN Device`では受け取るCAN IDが変化する。

https://github.com/Ayrton2718/CSMBus-GatewayF107RC/blob/main/Core/Src/eth_csmbus/ec_id.c
https://github.com/Ayrton2718/CSMBus-DriverG431/blob/main/Core/Src/can_csmbus/cc_id.c

