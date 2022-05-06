# Grafana　アラート設定

>以下の設定は、Grafana Ver.8以降に対応したものになります。  
>Grafana Ver.7以前にアラート設定をされていた方は、一度既存のアラート設定をすべて削除することで以下の設定が可能になります。
>
>Grafanaのバージョンは、左下の？マークにマウスカーソルを乗せることで確認可能です。

![a](https://user-images.githubusercontent.com/69729884/167123041-fef2d449-6df7-44cf-b726-0a00adfca169.png)


___
## 1,
左メニューの＋マークから**Folder**を選択し、**Folder name**に適当な名前を入れ、**Create**を押します。 

![b](https://user-images.githubusercontent.com/69729884/167106893-ec26c6de-2e02-4b53-b5d1-ef04fae0f52a.png)


___
## 2,
左メニューの鈴マークから**Alert rules**を選択し、**New alert rule**を押します。
![c](https://user-images.githubusercontent.com/69729884/167109229-a2fb373d-58d9-4f57-af95-f35995f0acf8.png)


___
## 3,
まずBPのアラートを作成します。
  
  
A,**Rule name**に `BP Alert`　を入力します。（任意変更可能）

B,**Rule type**は `Grafana managed alert` を選択してください。

C,**Folder**は、最初に作成したフォルダを選択してください。

D,**Metrics browser**の欄に  
```cardano_node_metrics_slotInEpoch_int{instance="<BPのIP>:12798", job="prometheus", type="cardano-node"}```  
↑を <BPのIP> 部分を置き換えてから入力してください。

E,**Legend**の欄に  
```{{alias}}```  
を入力してください。

F,**instant**を切り替えてオンにしてください。（ツマミが右）

G,該当箇所を　`IS BELOW 0` に置き換えてください。

H,**Configure no data and error handling**を展開し、**Alert state if no data or all values are null**を`Alerting`に変更してください。

I,右上の**Save and exit**を選択してください。
![d](https://user-images.githubusercontent.com/69729884/167117913-4fa5950b-3ccd-45d3-aa25-c1fe1102394d.png)


___
## 4,
**New alert rule**を選択し、同じ要領でリレーのアラートを作成します。
![e](https://user-images.githubusercontent.com/69729884/167115017-3abc90d7-6113-4ff4-aa51-55beb7a3d944.png)

A,**Rule name**に `Relay Alert`　を入力（任意変更可能）、

D,**Metrics browser**の欄に  
```cardano_node_metrics_slotInEpoch_int{instance="localhost:12798", job="prometheus", type="cardano-node"}```  
↑をそのまま入力してください。

後はBPのアラート作成と同じ流れになります。


___
## 5,
**Contact points**を選択し**New contact point**を押します。
![f](https://user-images.githubusercontent.com/69729884/167119874-fca808dd-62cc-4e39-88ef-93ad54e34860.png)


___
## 6,
アラートをどのアプリに飛ばすかを設定します。（Emailは特別な設定が必要なので非推奨とさせていただきます。）

**Name**には適当な名前を入力してください。

Webhook URLなど、各アプリで要求されるデータの入手方法はマニュアルの**ブロック生成ステータス通知設定手順**の中にまとめられています。
https://dev.xstakepool.com/block_notify#2-apurinolinediscordtelegramslackni

アラート状態が回復したという通知がいらない方は**Notification setting**の**Disable resolved message**にチェックを入れてください。

入力後、**Save contact point**を選択します。 
![g](https://user-images.githubusercontent.com/69729884/167127345-fac37479-0f33-4134-a6d3-22d1395340a7.png)

___
## 7,
**Notification policies**に移動し、**New specific policy**を選択します。
![h](https://user-images.githubusercontent.com/69729884/167121003-a5f91553-1316-4991-b220-4c777fabb6af.png)


___
## 8, 
**Contact point**を先ほど作成したアラート設定に変更し、**Save policy**を選択します。
![i](https://user-images.githubusercontent.com/69729884/167121198-e232a5a6-c23f-4bf2-aba6-0bb71ba94f68.png)


___
## 9,
設定完了です。ブロック生成予定の無いタイミングでノードを停止し、6分ほど待ってアラートがアプリに届くか確認してください。
