# プール誓約金(payment.addr)を使用したCatalyst登録方法
参考先：
https://github.com/gitmachtl/scripts/blob/master/SPO_Pledge_Catalyst_Registration.md

***このページの方法はFund10以降では使用できません。Fund10での登録方法はこちら（検証不十分）***
https://github.com/sakakibaraakio/etc/blob/main/Fund10%E5%AF%BE%E5%BF%9C%E3%80%80%E3%83%97%E3%83%BC%E3%83%AB%E8%AA%93%E7%B4%84%E9%87%91%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9FCatalyst%E7%99%BB%E9%8C%B2%E6%96%B9%E6%B3%95.md
*<details><summary>Fund9以前の方法</summary>*
___
## 1、jormungandrとVoter-toolを導入します。
jormungandrを導入します。

`BP`
```
mkdir $HOME/CatalystVoting
cd $HOME/CatalystVoting
wget https://github.com/input-output-hk/jormungandr/releases/download/$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name)/jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

tar -xf jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

```
Voter-toolを導入します。

`BP`
```
cd $HOME/CatalystVoting
wget https://hydra.iohk.io/build/9209906/download/1/voter-registration.tar.gz

tar -xf voter-registration.tar.gz

```

キーファイルを作成します。

`BP`
```
cd $HOME/CatalystVoting
./jcli key generate --type ed25519extended > catalyst-vote.skey
./jcli key to-public < catalyst-vote.skey > catalyst-vote.pkey

```
___
## 2、登録メタデータを生成します。
BPの$HOME/CatalystVotingにある、  
・catalyst-vote.pkey  
・voter-registration  
の2つを、エアギャップマシンの$NODE_HOMEにコピーします。  
  
`BP → catalyst-vote.pkey/voter-registration → エアギャップ`

　  
　   
BPにて最新スロットを取得し、戻りの数字をメモします。

`BP`
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')
echo Current Slot: $currentSlot

```
  
  
エアギャップにて以下コマンドを入力し、登録メタデータを生成します。  
<スロット番号> の箇所を、先ほどメモした数字に置き換えてから入力してください。

`エアギャップ`
```
cd $NODE_HOME
./voter-registration \
    --rewards-address $(cat $NODE_HOME/stake.addr) \
    --vote-public-key-file catalyst-vote.pkey \
    --stake-signing-key-file stake.skey \
    --slot-no <スロット番号> \
    --json > voting-registration-metadata.json

```


　  
エアギャップの$NODE_HOMEに生成された"voting-registration-metadata.json"を、BPの$HOME/CatalystVotingに移動します。  
  
`エアギャップ → voting-registration-metadata.json → BP`
___
## 3、トランザクションを作成、送信します。

最新スロット番号を取得します。

`BP`
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')
echo Current Slot: $currentSlot

```

payment.addrの残高を算出します。

`BP`
```
cd $NODE_HOME
cardano-cli query utxo \
    --address $(cat $NODE_HOME/payment.addr) \
    --mainnet > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr | sed -e '/lovelace + [0-9]/d' > balance.out

cat balance.out

```

UTXOを算出します。

`BP`
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

`BP`
```
cardano-cli transaction build-raw \
    ${tx_in} \
    --tx-out $(cat $NODE_HOME/payment.addr)+${total_balance} \
    --invalid-hereafter $(( ${currentSlot} + 10000)) \
    --fee 0 \
    --metadata-json-file $HOME//CatalystVoting/voting-registration-metadata.json \
    --out-file tx.tmp

```

最低手数料を出力します。

`BP`
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

`BP`
```
txOut=$((${total_balance}-${fee}))
echo txOut: ${txOut}

```

トランザクションファイルを構築します。

`BP`
```
cardano-cli transaction build-raw \
    ${tx_in} \
    --tx-out $(cat $NODE_HOME/payment.addr)+${txOut} \
    --invalid-hereafter $(( ${currentSlot} + 10000)) \
    --fee ${fee} \
    --metadata-json-file $HOME//CatalystVoting/voting-registration-metadata.json \
    --out-file tx.raw

```
　  
>BPのtx.raw をエアギャップマシンのcnodeディレクトリにコピーします。  


　  

トランザクションに署名します。

`エアギャップ`
```
cd $NODE_HOME
cardano-cli transaction sign \
    --tx-body-file tx.raw \
    --signing-key-file payment.skey \
    --mainnet \
    --out-file tx.signed

```
　  
   
>エアギャップマシンのtx.signed をBPのcnodeディレクトリにコピーします。  


　  
トランザクションを送信します。

`BP`
```
cardano-cli transaction submit \
    --tx-file $NODE_HOME/tx.signed \
    --mainnet

```
___
## 4、投票登録に使用するQRコードを作成します。

catalyst-toolboxを導入します。

`BP`
```
cd $HOME/CatalystVoting
wget https://github.com/input-output-hk/catalyst-toolbox/releases/download/$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name)/catalyst-toolbox-$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu.tar.gz
tar -xf catalyst-toolbox-$(curl -s https://api.github.com/repos/input-output-hk/catalyst-toolbox/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu.tar.gz

```

QRコードを作成します。 <4桁コード> の部分を任意の4桁数字に置き換えてから入力してください。

`BP`
```
./catalyst-toolbox qr-code encode --pin <4桁コード> --input catalyst-vote.skey --output catalyst-qrcode.png img

```


 

$HOME/CatalystVoting の中に"catalyst-qrcode.png"というファイルが作成されるので、DLします。
  
DLしたQRコードと設定した4桁pinコードを使用して、スマホアプリの"Catalyst Voting"にて登録を行えば完了です。  
お疲れさまでした。
_______
>作業後、エアギャップの  
>・catalyst-vote.pkey  
>・voter-registration  
>・voting-registration-metadata.json  
>は削除してもらっても大丈夫です。
  
削除コマンド  
`エアギャップ`
```
cd $NODE_HOME
rm catalyst-vote.pkey voter-registration voting-registration-metadata.json
```
</details>
<br>  
