# LED Display

LEDの表示板です。
<br>
USB接続によるシリアル通信で様々なエフェクトを使用した文字表示をすることができます。


### データフォーマット

LED Displayを使用するにはフォーマットに合わせる必要があります。

下記のヘッダーデータ、本文データ、フッターデータを１セットで送信することで、LED Displayに表示させることができます。

ヘッダーデータ

|No |設定例| 説明 |
|--|-- |--|
|1|0xA5 | 開始文字符号　固定|
|2|0x68 | 固定|
|3|0x32 | 固定
|4|0xff | LEDコントローラーの番号指定0xff:番号に関わらず応答|
|5|0x7b | 固定|
|6|0x01 | LEDコントローラー確認応答　0x00:なし0x01:あり|
|7|0x00〜 | 文字サイズ(length)下位バイト文字サイズ
|8|0x00〜 | 文字サイズ(length)上位バイト文字サイズ
|9|0x00〜 | パケット番号
|10|0x00〜 | 最終パケット番号


本文

|No |設定例| 説明 |
|--|-- |--|
|1|0x02 | 文字送信の符号固定
|2|0x00 | 送信ウィンドウ0x00〜0x07
|3|0x0A | エフェクト<br> 0x00:静止<br>0x01…左から開く<br>0x02:右から開く<br>0x03:中央から水平方向に開く<br>0x04:中央から垂直方向に開く<br>0x05:ストライプ<br>0x06:右からスライドイン<br>0x07:左からスライドイン<br>0x08:下からスライドイン<br>0x09:上からスライドイン<br>0x0A:下から上へスクロール<br>0x0B:右から左へスクロール<br>0x0C:左から右へスクロール
|4|0x02 | 文字揃え0x00:左揃え,0x01:中央揃え,0x02:右揃え
|5|0x50 | エフェクト速度0x00〜0x64
|6|0x00 | 静止時間(秒)の上位バイト
|7|0x03 | 静止時間(秒)の下位バイト
|8|0x12|送信文字1バイト目の上位4ビットは文字の色　0x1または0x5:赤0x2または0x6:緑0x3または0x7:黃下位4ビットは文字サイズ0x2(16px)
|9|0x00|文字コード1(半角文字の場合は2バイト目は0x00固定)
|10|0x52|文字コード２
|11-|-|No8〜No10の内容を送信する文字数分追加する|
||0x00|終了文字１ 送信する文字を全て設定後、〜空の文字を設定　0x00を固定|
||0x00|終了文字２ 0x00固定|
||0x00|終了文字３ 0x00固定|

フッターデータ

|No |設定値| 説明 |
|--|-- |--|
|1|0x00〜|チェックサムの下位バイト|
|2|0x00〜|チェックサムの上位バイト|
|3|0xae |終了文字|


## Setup
ここではMAC、Node.jsを使用した通信方法を記載します。

* Nodeのインストール

下記より環境にあったものを選択しダウンロード後インストールを行う
<br>
https://nodejs.org/en/download/

* USB接続

端末のUSBポートとLED displayのシリアル変換ボードをUSBケーブルで接続

## Sample Code

Node.jsのファイルを作成し、下記のコードを入力し保存。

led_display.js

```js
//
// FaBo Sample
//
// led_display
//
var serial = require('serialport').SerialPort;
serial = new serial('/dev/cu.usbserial-A4001fN1',{ // 接続先は環境により変更させる　　「ls /dev/tty.usb*」等で検索
    baudrate:115200}
);

serial.on('open',function(){
    serial.on('data',function(data){
        for(i=0; i<data.length; i++) {
         console.log(i+': ' + data[i].toString(16));
        }
    });

    // 変数定義
    var checkAndPush , i , j , k , _i , _len , td , sd;

    // 文字チェック&設定用(対応した場所以外で0xA5->開始判定文字,0xAE->終了判定文字,0xAA-> エスケープ文字がある場合、それぞれ対応が必要になる)
    checkAndPush = function(arr, data) {
        if (data === 0xA5 || data === 0xAA || data === 0xAE) {
            return arr.push(0xAA, data & 0x0F);
        } else {
            return arr.push(data);
        }
    };    

    // 表示するデータ
    var text = "SAMPLE.";

    // 送信文字のオプション  1バイト目の上位4ビットは文字の色　0x1または0x5:赤 0x2または0x6:緑 0x3または0x7:黃  下位4ビットは文字サイズ  0x2(16px)固定
    var fontSetting = 0x22;  // 色:2->緑　サイズ:2->16px 

    // 本文データ格納用
    var textData = [];

    // チェックサム(チェックに使用するデータ:送信データの0x68から本文終了までの合算値を格納)
    var checkSum;

    // 送信用データ格納
    var sendData = [0xA5, //開始文字符号　固定
                    0x68, //固定
                    0x32, //固定
                    0xff, //LEDコントローラーの番号指定 0xff:番号に関わらず応答
                    0x7b, //固定
                    0x01] //LEDコントローラー確認応答　0x00:なし 0x01:あり

    /*
    / 本文データの作成
    */
    
    // 本文の開始符号設定 0x02固定
    textData.push(0x02);

    // 送信ウィンドウ指定 0x00〜0x07
    textData.push(0x00);

    // エフェクトの指定
      // 0x00…静止 ,0x01…左から開く ,0x02…右から開く ,0x03…中央から水平方向に開く ,0x04…中央から垂直方向に開く ,0x05…ストライプ ,0x06…右からスライドイン
      // 0x07…左からスライドイン ,0x08…下からスライドイン ,0x09…上からスライドイン ,0x0A…下から上へスクロール ,0x0B…右から左へスクロール ,0x0C…左から右へスクロール
    textData.push(0x0A);

    // 文字の揃え方の指定
    textData.push(0x02); //文字揃え 0x00:左揃え, 0x01:中央揃え, 0x02:右揃え

    // エフェクト速度 0x00〜0x64 小さいほど速くなる
    textData.push(0x50);

    // 1度のエフェクトごとの静止時間(秒)の上位バイト
    textData.push(0x00);

    // 1度のエフェクトごとの静止時間(秒)の下位バイト
    textData.push(0x03);

    // テキストのコード変換
    for (j = 0 ; j<text.length; j++){
        // 1バイト目：文字オプション(0x12、文字コード1バイト目(半角の場合は0x00固定)、文字2バイト目を設定
        textData.push( fontSetting , 0x00 , text.charCodeAt(j));
    }
    // 終了文字設定(全て0x00を設定)
    textData.push( 0x00 , 0x00 , 0x00);


    /*
    / 送信用データへの設定
    */
    
    // 文字サイズを送信用データに設定
    textSize = textData.length;
    sendData.push( textSize & 0x00ff);         //文字サイズ下位バイト
    sendData.push( (textSize & 0xff00) >> 8);  //文字サイズ上位バイト

    //パケット番号、最終パケット番号設定 1パケットのみで処理するので0x00固定
    sendData.push(0x00,0x00);

    // 送信文字情報を送信用変数に格納
    for (k = 0 ; k < textData.length ; k++){
        td = textData[k];
        sendData.push(td);
    }

    // チェックサム初期化
    checkSum = 0;

    // 送信データを格納
    for(i=1; i<sendData.length; i++) {
        sd = sendData[i];
        checkSum += sd;
    }

    // チェックサムを出力用データに設定
    checkAndPush(sendData,checkSum & 0x00ff);
    checkAndPush(sendData,(checkSum & 0xff00) >> 8);

    // 終了文字設定
    sendData.push(0xae);

    // シリアルデータ書き込み
    serial.write(sendData, function(err, results) {
        console.log('err ' + err);
        console.log('results ' + results);
    });
});
```
<br>
ターミナルし、上記のファイルを格納した場所に移動し、下記のコマンドを実行

```$ node led_display```

Ctrl+Cキーを押下で終了