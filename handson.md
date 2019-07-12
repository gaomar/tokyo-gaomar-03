---
id: dist
{{if .Meta.Status}}status: {{.Meta.Status}}{{end}}
{{if .Meta.Summary}}summary: {{.Meta.Summary}}{{end}}
{{if .Meta.Author}}author: {{.Meta.Author}}{{end}}
{{if .Meta.Categories}}categories: {{commaSep .Meta.Categories}}{{end}}
{{if .Meta.Tags}}tags: {{commaSep .Meta.Tags}}{{end}}
{{if .Meta.Feedback}}feedback link: {{.Meta.Feedback}}{{end}}
{{if .Meta.GA}}analytics account: {{.Meta.GA}}{{end}}

---

# M5StickC + LINE Things + Amazon Connect連携ハンズオン

## 問い合わせフローを作ろう！
### Amazon Connect電話番号取得

すでにAmazon Connectの電話番号を取得している前提で開始します。
まだ未取得な方はこちらから番号を取得しておいてください。

[https://ac-handson-00.netlify.com](https://ac-handson-00.netlify.com)

### 1-1. 問い合わせフローの作成
左側メニューのルーティングから「問い合わせフロー」をクリックします。

![s100](images/s100.png)

［問い合わせフローの作成］をクリックします。

![s101](images/s101.png)

名前を「M5Stick-AmazonConnect」と入力します。

![s102](images/s102.png)

設定カテゴリにある「音声の設定」ブロックをドラッグアンドドロップして、ドロップしたブロックをクリックします。

![s103](images/s103.png)

言語は「日本語」でお好きな音声を選択してください。

![s104](images/s104.png)

エントリポイントと音声の設定ブロックを繋げます。

![s105](images/s105.png)

設定カテゴリにある「問い合わせ属性の設定」をドラッグアンドドロップします。

![s106](images/s106.png)

属性の設定を行います。「属性を使用する」を選択してください。

|項目|値|
|:--|:--|
|宛先キー|message|
|タイプ|ユーザー定義|
|属性|message|

![s107](images/s107.png)

ブロックを繋げます。

![s108](images/s108.png)

操作カテゴリにある「プロンプトの再生」をドラッグアンドドロップします。

![s109](images/s109.png)

属性の設定を行います。「テキスト読み上げ機能(アドホック)」を選択してください。

|項目|値|
|:--|:--|
|テキスト読み上げ機能(アドホック)|動的に入力する|
|タイプ|ユーザー定義|
|属性|message|

![s110](images/s110.png)

ブロックを繋げます。

![s111](images/s111.png)

ブランチカテゴリにある「ループ」をドラッグアンドドロップします。
ループ回数はお好きな数を指定してください。

![s112](images/s112.png)

ブロックを繋げます。

![s113](images/s113.png)

ループとプロンプトの再生を繋げます。

![s114](images/s114.png)

終了 / 転送カテゴリにある「切断/ハングアップ」をドラッグアンドドロップします。

![s115](images/s115.png)

まだ繋いでいない部分を全て「切断/ハングアップ」に繋ぎます。

![s116](images/s116.png)

右上の［保存］と「公開」ボタンを順番にクリックします。

![s117](images/s117.png)

### 1-2. IDをメモしておく
問い合わせフローの名前の下に「追加のフロー情報の表示」という項目があるので、それを展開します。展開するとARNの情報が表示されるのでinstanceのIDとconstact-flowのIDをそれぞれメモしておきます。

![s118](images/s118.png)

## Lambda関数を作成しよう！
### 2-1. Lambda関数を作成する

Lambdaから新規で関数を作成します。［関数の作成］ボタンをクリックします。

![s130](images/s130.png)

関数は以下の通り入力して、［関数の作成］ボタンをクリックします。

| 項目       |       値 |
|:-----------------|:------------------|
|①関数名|M5StickC-AmazonConnect|
|②ランタイム|Node.js 10.x|
|③実行ロール|新しいロールを作成|
|④ロール名|M5StickC-AmazonConnect-Role|
|⑤ポリシーテンプレート|基本的なLambda@Edgeのアクセス権限|

![s131](images/s131.png)

### 2-2. Amazon Connectアクセス権限を追加する
M5StickC-AmazonConnect-Roleロールを表示をクリックします。

![s132](images/s132.png)

［インラインポリシーの追加］をクリックします。

![s133](images/s133.png)

サービスを展開して、検索窓に「Connect」と入れて検索します。出てきた［Connect］をクリックします。

![s134](images/s134.png)

アクションのアクセスレベルにある「書き込み」部分を展開して、その中にある`StartOutboundVoiceContact`のチェックを入れます。

![s135](images/s135.png)

すべてのリソースを選択して、右下の［ポリシーの確認］ボタンをクリックします。

![s136](images/s136.png)

ポリシー名を入力します。`M5Stick-AmazonConnect-Policy`としました。右下の［ポリシーの作成］ボタンをクリックします。

![s137](images/s137.png)

Lambda画面に戻り、画面更新するとAmazon Connectの権限が追加されます。

![s138](images/s138.png)

### 2-3. プログラムを書き込む
index.jsを開き、下記プログラムをコピペしてください。
LINE Thingsからリクエストが飛んでくるので、bodyから対象値を取得します。base64で格納されているので、デコード処理が必要です。

```javascript:index.js
const Util = require('./util.js');

exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    const thingsData = body.events[0].things.result;

    const response = {
        statusCode: 200,
        body: JSON.stringify('Hello from M5StickC!'),
    };

    if (thingsData.bleNotificationPayload) {
        // LINE Thingsから飛んでくるデータを取得
        const blePayload = thingsData.bleNotificationPayload;
        var buffer1 = new Buffer(blePayload, 'base64');
        var stickData = buffer1.toString('ascii');  //Base64をデコード
        console.log("M5StickC-Payload=" + stickData);
        
        const sendMessage = `エムファイブスティックの値は「${stickData}」です。`;

        // Amazon Connect送信
        await Util.callMessageAction(sendMessage);

    }  
    return response;
};
```

新規ファイルを作成します。

![s150](images/s150.png)

下記コードをコピペしてください。

```javascript:util.js
'use strict';
const AWS = require('aws-sdk');
var connect = new AWS.Connect();

// 電話をかける処理
module.exports.callMessageAction = async function callMessageAction(message) {
    return new Promise(((resolve, reject) => {

        // Attributesに発話する内容を設定
        var params = {
            Attributes: {"message": message},
            InstanceId: process.env.INSTANCEID,
            ContactFlowId: process.env.CONTACTFLOWID,
            DestinationPhoneNumber: process.env.PHONENUMBER,
            SourcePhoneNumber: process.env.SOURCEPHONENUMBER
        };

        // 電話をかける
        connect.startOutboundVoiceContact(params, function(err, data) {
            if (err) {
                console.log(err);
                reject();
            } else {
                resolve(data);
            }
        });
    }));
};
```

保存する際はファイル名を「util.js」にしてください。

![s151](images/s151.png)

### 2-4. 環境変数を設定する
Amazon Connectと連携するための環境変数を設定します。

| キー名       |       値 |
|:-----------------|:------------------|
|INSTANCEID|1-2でメモした<span style="color: red; ">instance</span>のID|
|CONTACTFLOWID|1-2でメモした<span style="color: blue; ">contact-flow</span>のID|
|PHONENUMBER|ご自身の携帯電話番号 ※+81を先頭につけて数字のみにします<br>例)090-1234-5678 👉+819012345678|
|SOURCEPHONENUMBER|Amazon Connectで取得した電話番号 ※+81を先頭につけて数字のみにします|

![s152](images/s152.png)

### 2-5. API Gatewayを設定する
LINE ThingsからアクセスするためのURLを発行します。
［トリガーを追加］をクリックします。

![s153](images/s153.png)

トリガーの設定は下記を指定します。最後に［追加］ボタンをクリックします。

| 項目       |       値 |
|:-----------------|:------------------|
|トリガー|API Gateway|
|API|新規APIの作成|
|セキュリティ|オープン|

![s154](images/s154.png)

APIエンドポイントのURLをメモしておきましょう

![s155](images/s155.png)

右上の保存ボタンをクリックします。

![s156](images/s156.png)

## LINE Botを作ろう！
### 3-1. チャネルの作成
下記にアクセスしてログインしてください。
[https://developers.line.biz/ja/](https://developers.line.biz/ja/)

プロバイダーがまだ無い方は作成お願いします。

新規チャネルを作成します。

![s160](images/s160.png)

Messaging APIをクリック

![s161](images/s161.png)

［同意する］をクリック

![s162](images/s162.png)

2つのチェックを入れてから［作成］をクリック

![s163](images/s163.png)

| 項目       |       値 |
|:-----------------|:------------------|
|①アプリ名|M5StickC連携ハンズオン|
|②アプリ説明|M5StickC連携ハンズオン|
|③大業種|旅行・エンタメ・レジャー|
|④小業種|その他娯楽・エンタメ|
|⑤メールアドレス|ご自身のメールアドレス|

![s164](images/s164.png)

［入力内容を確認する］ボタンをクリック

![s165](images/s165.png)

作成したチャネルをクリックします

![s166](images/s166.png)

### 3-2. アクセストークンを発行する
このチャネルにアクセスするためのトークンを発行します。

［再発行］をクリックします。

![s167](images/s167.png)

[再発行]をクリック

![s168](images/s168.png)

発行されたアクセストークンをメモしておきましょう。

![s169](images/s169.png)

### 3-3. Webhookを設定
Webhookのアクセス先URLを指定します。

![s170](images/s170.png)

2-5で作成したAPI GatewayのURLを貼り付けます。「https://」は既にあるので、省略して貼り付けます。［更新］ボタンをクリックします。

![s171](images/s171.png)

Webhook送信を利用するに変えます

![s172](images/s172.png)

利用するを選択して［更新］ボタンをクリックします。
稀に利用するに変わらないことがあるので、画面をリロードして確認してください。

![s173](images/s173.png)

### 3-4. LINE Botと友だちになる
下にスクロールするとQRコードがあるので、LINEアプリからQRを読み取って、
Botと友だちになっておいてください。

![s173-1](images/s173-1.png)

### 3-5. LIFFを設定する
LIFFアプリを設定します。

![s174](images/s174.png)

| 項目       |       値 |
|:-----------------|:------------------|
|①名前|M5Stick-AmazonConnect|
|②サイズ|Tall|
|③エンドポイントURL|https://www.google.com/|
|④オプション|必ずONにする|

![s175](images/s175.png)

## LINE Thingsの設定をしよう！
### 4-1. シナリオを作成する

[のびすけ](https://twitter.com/n0bisuke)さんが作ったツールをありがたく使います。
下記にアクセスしてください。

[https://n0bisuke.github.io/linethingsgen/](https://n0bisuke.github.io/linethingsgen/)

左側メニューのSettingをクリックして、3-2で生成したアクセストークンを貼り付けます。
［保存］ボタンをクリックします。

![s177](images/s177.png)

左側メニューのCreate Productをクリックして、プルダウンメニューから3-5で作成したLIFFアプリを指定します。
トライアルプロダクトの名前を入力して［保存］ボタンをクリックします。

![s178](images/s178.png)

左側メニューのCreate Scenarioをクリックして、プルダウンメニューから選択。
そのままの状態で［シナリオセットの登録/更新］ボタンをクリックします。

![s179](images/s179.png)

発行されたserviceUuidをメモしておきます。

![s180](images/s180.png)

### 4-2. LINEアプリのLINE Things有効化
既に有効化している方はこの手順をスキップしてください。
以下のURLを読み込んで、LINE Thingsを有効化してください。

![s181](images/s181.png)

設定ページにLINE Thingsの項目が増えます

![s182](images/s182.png)

## M5Stickにコードを書き込む
### 5-1. コードを書き込む
Arduino IDEを開いて下記コードを入力します。
11行目のサービスUUIDは4-1で生成した値を入力してください。

コードはこちらからもコピペできます

[https://github.com/gaomar/tokyo-gaomar-03/raw/master/files/M5Stick-AmazonConnect/M5Stick-AmazonConnect.ino](https://github.com/gaomar/tokyo-gaomar-03/raw/master/files/M5Stick-AmazonConnect/M5Stick-AmazonConnect.ino)


```c
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <M5StickC.h>

// Device Name: Maximum 30 bytes
#define DEVICE_NAME "M5Stick"

// あなたのサービスUUIDを貼り付けてください
#define USER_SERVICE_UUID "＜あなたのサービスUUID＞"
// Notify UUID: トライアル版は値が固定される
#define NOTIFY_CHARACTERISTIC_UUID "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"
// PSDI Service UUID: トライアル版は値が固定される
#define PSDI_SERVICE_UUID "E625601E-9E55-4597-A598-76018A0D293D"
// PSDI CHARACTERISTIC UUID: トライアル版は値が固定される
#define PSDI_CHARACTERISTIC_UUID "26E2B12B-85F0-4F3F-9FDD-91D114270E6E"

BLEServer* thingsServer;
BLESecurity* thingsSecurity;
BLEService* userService;
BLEService* psdiService;
BLECharacteristic* psdiCharacteristic;
BLECharacteristic* notifyCharacteristic;

bool deviceConnected = false;
bool oldDeviceConnected = false;

class serverCallbacks: public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) {
   deviceConnected = true;
  };

  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;
  }
};

void setup() {
  Serial.begin(115200);

  BLEDevice::init("");
  BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_NO_MITM);

  // Security Settings
  BLESecurity *thingsSecurity = new BLESecurity();
  thingsSecurity->setAuthenticationMode(ESP_LE_AUTH_BOND);
  thingsSecurity->setCapability(ESP_IO_CAP_NONE);
  thingsSecurity->setInitEncryptionKey(ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK);

  setupServices();
  startAdvertising();

  // M5Stick LCD Setup
  M5.begin(true, false, false);

  // 横向き
  M5.Lcd.setRotation(3);

  // ボタン初期化
  pinMode(M5_BUTTON_HOME, INPUT);  
  pinMode(M5_BUTTON_RST, INPUT);

  M5.Lcd.fillScreen(TFT_BLACK);
  M5.Lcd.setTextColor(YELLOW);
  M5.Lcd.setCursor(0, 0);   
  M5.Lcd.print("Ready to Connect");
  Serial.println("Ready to Connect");
}

int btnValue = 0;

void loop() {

  if(digitalRead(M5_BUTTON_RST) == LOW) {
    // カウントアップ
    btnValue++;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Count");
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(30, 40);
    M5.Lcd.printf("%d", btnValue);

    // ボタンが離されるまでループ
    while(digitalRead(M5_BUTTON_RST) == LOW);
  }

  if(digitalRead(M5_BUTTON_HOME) == LOW) {
    // LINE Botに紐づくWebhookにデータ送信
    const char *newValue=((String)btnValue).c_str();
    notifyCharacteristic->setValue(newValue);
    notifyCharacteristic->notify();

    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Send!!");

    // ボタンが離されるまでループ
    while(digitalRead(M5_BUTTON_HOME) == LOW);

    // カウント初期化
    btnValue = 0;
    delay(3000);
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Count");
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(30, 40);
    M5.Lcd.printf("%d", btnValue);    
  }

  // Disconnection
  if (!deviceConnected && oldDeviceConnected) {
    delay(500); // Wait for BLE Stack to be ready
    thingsServer->startAdvertising(); // Restart advertising
    oldDeviceConnected = deviceConnected;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(YELLOW);
    M5.Lcd.setCursor(0, 0);     
    M5.Lcd.print("Ready to Connect");
  }
  // Connection
  if (deviceConnected && !oldDeviceConnected) {
    oldDeviceConnected = deviceConnected;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(0, 0);       
    M5.Lcd.print("Connected");
  }
}

// サービス初期化
void setupServices(void) {
  // Create BLE Server
  thingsServer = BLEDevice::createServer();
  thingsServer->setCallbacks(new serverCallbacks());

  // Setup User Service
  userService = thingsServer->createService(USER_SERVICE_UUID);

  // Notifyセットアップ
  notifyCharacteristic = userService->createCharacteristic(NOTIFY_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  notifyCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  BLE2902* ble9202 = new BLE2902();
  ble9202->setNotifications(true);
  ble9202->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  notifyCharacteristic->addDescriptor(ble9202);

  // Setup PSDI Service
  psdiService = thingsServer->createService(PSDI_SERVICE_UUID);
  psdiCharacteristic = psdiService->createCharacteristic(PSDI_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_READ);
  psdiCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);

  // Set PSDI (Product Specific Device ID) value
  uint64_t macAddress = ESP.getEfuseMac();
  psdiCharacteristic->setValue((uint8_t*) &macAddress, sizeof(macAddress));

  // Start BLE Services
  userService->start();
  psdiService->start();
}

void startAdvertising(void) {
  // Start Advertising
  BLEAdvertisementData scanResponseData = BLEAdvertisementData();
  scanResponseData.setFlags(0x06); // GENERAL_DISC_MODE 0x02 | BR_EDR_NOT_SUPPORTED 0x04
  scanResponseData.setName(DEVICE_NAME);

  thingsServer->getAdvertising()->addServiceUUID(userService->getUUID());
  thingsServer->getAdvertising()->setScanResponseData(scanResponseData);
  thingsServer->getAdvertising()->start();
}
```

![s185](images/s185.png)

LINEアプリを開いて、20190722ハンズオンをタップします

![s183](images/s183.png)

［今すぐ利用］をタップします

![s184](images/s184.png)

横のボタンをクリックしてカウントをアップさせて、中央のボタンをクリックするとAmazon Connectから値を教えてくれます。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">7/22のM5StickC + LINE Things + Amazon Connect連携ハンズオンはこれを作ります！押したボタンの値を教えてくれます。<a href="https://t.co/HIdM5EDfGw">https://t.co/HIdM5EDfGw</a> <a href="https://twitter.com/hashtag/M5StickC?src=hash&amp;ref_src=twsrc%5Etfw">#M5StickC</a> <a href="https://twitter.com/hashtag/AmazonConnect?src=hash&amp;ref_src=twsrc%5Etfw">#AmazonConnect</a> <a href="https://t.co/lKGUZPb7m2">pic.twitter.com/lKGUZPb7m2</a></p>&mdash; がおまる@スマートスピーカーアプリ開発入門発売中！ (@gaomar) <a href="https://twitter.com/gaomar/status/1146329025687670784?ref_src=twsrc%5Etfw">July 3, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## M5Stackにコードを書き込む
### 6-1. コードを書き込む
M5Stackをお持ちの方はこちらです。

Arduino IDEを開いて下記コードを入力します。
11行目のサービスUUIDは4-1で生成した値を入力してください。

コードはこちらからもコピペできます

[https://github.com/gaomar/tokyo-gaomar-03/raw/master/files/M5Stack-AmazonConnect/M5Stack-AmazonConnect.ino](https://github.com/gaomar/tokyo-gaomar-03/raw/master/files/M5Stack-AmazonConnect/M5Stack-AmazonConnect.ino)

```c
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <M5Stack.h>

// Device Name: Maximum 30 bytes
#define DEVICE_NAME "M5Stack"

// あなたのサービスUUIDを貼り付けてください
#define USER_SERVICE_UUID "＜あなたのサービスUUID＞"
// Notify UUID: トライアル版は値が固定される
#define NOTIFY_CHARACTERISTIC_UUID "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"
// PSDI Service UUID: トライアル版は値が固定される
#define PSDI_SERVICE_UUID "E625601E-9E55-4597-A598-76018A0D293D"
// PSDI CHARACTERISTIC UUID: トライアル版は値が固定される
#define PSDI_CHARACTERISTIC_UUID "26E2B12B-85F0-4F3F-9FDD-91D114270E6E"

BLEServer* thingsServer;
BLESecurity* thingsSecurity;
BLEService* userService;
BLEService* psdiService;
BLECharacteristic* psdiCharacteristic;
BLECharacteristic* notifyCharacteristic;

bool deviceConnected = false;
bool oldDeviceConnected = false;

class serverCallbacks: public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) {
   deviceConnected = true;
  };

  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;
  }
};

void setup() {
  Serial.begin(115200);

  BLEDevice::init("");
  BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_NO_MITM);

  // Security Settings
  BLESecurity *thingsSecurity = new BLESecurity();
  thingsSecurity->setAuthenticationMode(ESP_LE_AUTH_BOND);
  thingsSecurity->setCapability(ESP_IO_CAP_NONE);
  thingsSecurity->setInitEncryptionKey(ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK);

  setupServices();
  startAdvertising();

  // M5Stack LCD Setup
  M5.begin(true, false, false);
  M5.Lcd.clear(BLACK);
  M5.Lcd.setTextColor(YELLOW);
  M5.Lcd.setTextSize(2);
  M5.Lcd.setCursor(0, 0);
  M5.Lcd.println("Ready to Connect");
  Serial.println("Ready to Connect");

}

int btnValue = 0;

void loop() {

  M5.update();

  if (M5.BtnA.wasPressed()) {
    // カウントアップ
    btnValue++;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Count");
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(30, 40);
    M5.Lcd.printf("%d", btnValue);

  }

  if (M5.BtnB.wasPressed()) {
    // LINE Botに紐づくWebhookにデータ送信
    const char *newValue=((String)btnValue).c_str();
    notifyCharacteristic->setValue(newValue);
    notifyCharacteristic->notify();

    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Send!!");

    // カウント初期化
    btnValue = 0;
    delay(3000);
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setCursor(0, 0);
    M5.Lcd.print("Count");
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(30, 40);
    M5.Lcd.printf("%d", btnValue);    
  }

  // Disconnection
  if (!deviceConnected && oldDeviceConnected) {
    delay(500); // Wait for BLE Stack to be ready
    thingsServer->startAdvertising(); // Restart advertising
    oldDeviceConnected = deviceConnected;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(YELLOW);
    M5.Lcd.setCursor(0, 0);     
    M5.Lcd.print("Ready to Connect");
  }
  // Connection
  if (deviceConnected && !oldDeviceConnected) {
    oldDeviceConnected = deviceConnected;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setCursor(0, 0);       
    M5.Lcd.print("Connected");
  }
}

// サービス初期化
void setupServices(void) {
  // Create BLE Server
  thingsServer = BLEDevice::createServer();
  thingsServer->setCallbacks(new serverCallbacks());

  // Setup User Service
  userService = thingsServer->createService(USER_SERVICE_UUID);

  // Notifyセットアップ
  notifyCharacteristic = userService->createCharacteristic(NOTIFY_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  notifyCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  BLE2902* ble9202 = new BLE2902();
  ble9202->setNotifications(true);
  ble9202->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  notifyCharacteristic->addDescriptor(ble9202);

  // Setup PSDI Service
  psdiService = thingsServer->createService(PSDI_SERVICE_UUID);
  psdiCharacteristic = psdiService->createCharacteristic(PSDI_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_READ);
  psdiCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);

  // Set PSDI (Product Specific Device ID) value
  uint64_t macAddress = ESP.getEfuseMac();
  psdiCharacteristic->setValue((uint8_t*) &macAddress, sizeof(macAddress));

  // Start BLE Services
  userService->start();
  psdiService->start();
}

void startAdvertising(void) {
  // Start Advertising
  BLEAdvertisementData scanResponseData = BLEAdvertisementData();
  scanResponseData.setFlags(0x06); // GENERAL_DISC_MODE 0x02 | BR_EDR_NOT_SUPPORTED 0x04
  scanResponseData.setName(DEVICE_NAME);

  thingsServer->getAdvertising()->addServiceUUID(userService->getUUID());
  thingsServer->getAdvertising()->setScanResponseData(scanResponseData);
  thingsServer->getAdvertising()->start();
}

```

書き込むボードの設定は以下の通り

|項目|値|
|:--|:--|
|ボード|M5Stack-Core-ESP32|
|Upload Speed|921600|
|シリアルポート|/dev/cu.SLAB_USBtoUART|


![s186](images/s186.png)
