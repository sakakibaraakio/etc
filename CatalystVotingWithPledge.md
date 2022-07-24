# プール誓約金(payment.addr)を使用したCatalyst登録方法
参考先：
https://github.com/gitmachtl/scripts/blob/master/SPO_Pledge_Catalyst_Registration.md

以下の作業はBPにて実行してください。
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
## 2、手数料支払い用アドレスに、1.5ADAを送金します。
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

キーファイルを作成します。
```
./jcli key generate --type ed25519extended > catalyst-vote.skey
./jcli key to-public < catalyst-vote.skey > catalyst-vote.pkey

```
___
## 4、登録メタデータを生成します。
BPの$HOME/CatalystVotingにある、  
・catalyst-vote.pkey  
・voter-registration  
の2つを、エアギャップマシンの$NODE_HOMEにコピーします。

BPにて最新スロットを取得し、戻りの数字をメモします。
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')
echo Current Slot: $currentSlot

```

エアギャップにて以下コマンドを入力し、登録メタデータを生成します。
```
cd $NODE_HOME
./voter-registration \
    --rewards-address $(cat $NODE_HOME/stake.addr) \
    --vote-public-key-file catalyst-vote.pkey \
    --stake-signing-key-file stake.skey \
    --slot-no <スロット番号> \
    --json > voting-registration-metadata.json

```
> <スロット番号> の箇所を、先ほどメモした数字に置き換えてから入力してください。  
  
  
エアギャップの$NODE_HOMEに生成された"voting-registration-metadata.json"を、BPの$HOME/CatalystVotingに移動します。
>エアギャップの
>・catalyst-vote.pkey
>・voter-registration
>は削除してもらっても大丈夫です。
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

UTXOを算出します。
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
txOut=$((${total_balance}-${fee}))
echo txOut: ${txOut}

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

```

QRコードを作成します。
```
./catalyst-toolbox qr-code encode --pin <4桁コード> --input catalyst-vote.skey --output catalyst-qrcode.png img

```
> <4桁コード> の部分を任意の4桁数字に置き換えてから入力してください。

 

$HOME/CatalystVoting の中に"catalyst-qrcode.png"というファイルが作成されるので、DLします。
___ 
DLしたQRコードと設定した4桁pinコードを使用して、スマホアプリの"Catalyst Voting"にて登録を行えば完了です。


