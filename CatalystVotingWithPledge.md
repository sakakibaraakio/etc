# プール誓約金(payment.addr)を使用したCatalyst登録方法
参考先：
https://github.com/gitmachtl/scripts/blob/master/SPO_Pledge_Catalyst_Registration.md


>以下の手順では、作業中stake.skeyをサーバー上へ移動する必要が出てきます。
>リスクを承知の上で自己責任で行ってください。
>(paymentキーは使用しないので、payment.addr内の誓約金は危険にさらされません。)

以下の作業はBPにて行います。
___
## 1、手数料支払い用のアドレスを生成します。

```
mkdir $HOME/CatalystVoting
cd $HOME/CatalystVoting
cardano-cli address key-gen \
    --verification-key-file catalystpayment.vkey \
    --signing-key-file catalystpayment.skey

```

___
## 2、手数料支払い用アドレスに、2ADAを送金します。
支払先アドレス表示コマンド
```
cardano-cli address build \
    --payment-verification-key-file catalystpayment.vkey \
    --out-file catalystpayment.addr \
    --mainnet

echo "$(cat catalystpayment.addr)"

```

残高確認コマンド
```
cardano-cli query utxo \
    --address $(cat catalystpayment.addr) \
    --mainnet

```
___
## 3、jormungandrとVoter-toolを導入します。
jormungandrの導入
```
wget https://github.com/input-output-hk/jormungandr/releases/download/$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name)/jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

tar -xf jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

```
Voter-toolの導入
```
wget https://hydra.iohk.io/build/9209906/download/1/voter-registration.tar.gz

tar -xf voter-registration.tar.gz

```
___
## 4、登録メタデータを生成します。
エアギャップマシンの$NODE_HOME/stake.skeyを、BPの$HOME/CatalystVotingにコピーします。

登録メタデータを生成します。
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')

voter-registration \
    --rewards-address $(cat $NODE_HOME/stake.addr) \
    --vote-public-key-file catalyst-vote.pkey \
    --stake-signing-key-file stake.skey \
    --slot-no $currentSlot \
    --json > voting-registration-metadata.json

```

メタデータ生成後、stake.skeyを削除します。
```
rm $HOME/CatalystVoting/stake.skey
```
___
## 5、トランザクションを作成、送信します。

最新スロット番号を取得します。
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')
echo Current Slot: $currentSlot

```

catalystpayment.addrの残高を算出します。
```
cardano-cli query utxo \
    --address $(cat $HOME/CatalystVoting/catalystpayment.addr) \
    --mainnet > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr | sed -e '/lovelace + [0-9]/d' > balance.out

cat balance.out

```

UTXOを算出します
```
tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}

```

build-raw transactionコマンドを実行します。
```
cardano-cli transaction build-raw \
    ${tx_in} \
    --tx-out $(cat catalystpayment.addr)+${total_balance} \
    --invalid-hereafter $(( ${currentSlot} + 10000)) \
    --fee 0 \
    --metadata-json-file voting-registration-metadata.json \
    --out-file tx.tmp

```

最低手数料を出力します。
```
fee=$(cardano-cli transaction calculate-min-fee \
    --tx-body-file tx.tmp \
    --tx-in-count ${txcnt} \
    --tx-out-count 2 \
    --mainnet \
    --witness-count 1 \
    --byron-witness-count 0 \
    --protocol-params-file $NODE_HOME/params.json | awk '{ print $1 }')
echo fee: $fee

```

変更出力を計算します。
```
txOut=$((${total_balance}-${fee}-${amountToSend}))
echo Change Output: ${txOut}

```

トランザクションファイルを構築します。
```
cardano-cli transaction build-raw \
    ${tx_in} \
    --tx-out $(cat catalystpayment.addr)+${txOut} \
    --invalid-hereafter $(( ${currentSlot} + 10000)) \
    --fee ${fee} \
    --metadata-json-file voting-registration-metadata.json \
    --out-file tx.raw

```

トランザクションに署名します。
```
cardano-cli transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file catalystpayment.skey \
    --mainnet \
    --out-file tx.signed

```

トランザクションを送信します。
```
cardano-cli transaction submit \
    --tx-file tx.signed \
    --mainnet

```
___
## 6、投票登録に使用するQRコードを作成します。

catalyst-toolboxを導入します。
```
cd $HOME/CatalystVoting
wget https://github.com/input-output-hk/catalyst-toolbox/releases/download/$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name)/catalyst-toolbox-$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu.tar.gz
tar -xf catalyst-toolbox-$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu.tar.gz
cp $(find . -name catalyst-toolbox -executable -type f) ~/cardano/.

```

QRコードを作成します。
```
./catalyst-toolbox qr-code --pin <4桁コード> --input catalyst-vote.skey --output catalyst-qrcode.png img

```
> <4桁コード> の部分を任意の4桁数字に置き換えてから入力してください。


表示されたqrコードと設定した4桁pinコードを使用して、スマホアプリの"Catalyst Voting"にて登録を行います。


