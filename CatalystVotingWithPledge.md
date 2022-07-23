# プール誓約金(payment.addr)を使用したCatalyst登録方法

>以下の手順では、作業中stake.skeyをサーバー上へ置いておく必要が出てきます。
>リスクを承知の上で自己責任で行ってください。
>(paymentキーは使用しないので、payment.addr内の誓約金は危険にさらされません。)

___
## 1、手数料支払い用のアドレスを生成します。

```mkdir $HOME/CatalystVoting
cd $HOME/catalystVoting
cardano-cli address key-gen \
    --verification-key-file catalystpayment.vkey
    --signing-key-file catalystpayment.skey
```

___
## 2、手数料支払い用アドレスに、2ADAを送金します。
支払先アドレス表示コマンド
```cardano-cli address build \
    --payment-verification-key-file catalystpayment.vkey \
    --out-file catalystpayment.addr \
    --mainnet

cat catalystpayment.addr
```