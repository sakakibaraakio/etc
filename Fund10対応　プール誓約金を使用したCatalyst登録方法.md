# プール誓約金(payment.addr)を使用したCatalyst登録方法  
  
Fund10からはCatalyst登録の仕様が変わる為、Fund9以前に登録されていた方も再登録作業が必要です。  
>Fund10より、Catalyst報酬はstake.addrではなくpayment.addrに入金されます。  

参考先：
https://github.com/gitmachtl/scripts/blob/master/Catalyst_Registration_CLI_Tools.md

  
*<details><summary>Catalyst Fund9以前に、プール誓約を利用したCatalyst登録を実行していた方はこちら</summary>*
以前の登録に使用したディレクトリをバックアップします。
```
cd $HOME
mv CatalystVoting CatalystVotingFund9
```
</details>
<br>  

___  

## 1、Cardano-SignerとBech32を導入します。
Cardano-Signerを導入します。

`BP`
```
mkdir $HOME/CatalystVoting
cd $HOME/CatalystVoting
wget https://github.com/gitmachtl/cardano-signer/releases/download/$(curl -s https://api.github.com/repos/gitmachtl/cardano-signer/releases/latest | jq -r .tag_name)/cardano-signer-$(curl -s https://api.github.com/repos/gitmachtl/cardano-signer/releases/latest | jq -r .tag_name | tr -d v)_linux-x64.tar.gz
tar -xf cardano-signer-$(curl -s https://api.github.com/repos/gitmachtl/cardano-signer/releases/latest | jq -r .tag_name | tr -d v)_linux-x64.tar.gz

```

Bech32を導入します。

`BP`
```
wget https://github.com/input-output-hk/bech32/archive/refs/tags/$(curl -s https://api.github.com/repos/input-output-hk/bech32/releases/latest | jq -r .tag_name).tar.gz
tar -xf $(curl -s https://api.github.com/repos/input-output-hk/bech32/releases/latest | jq -r .tag_name).tar.gz

```
  
キーファイルを作成します。

`BP`
```
./cardano-signer keygen \
	--cip36 \
	--json-extended \
	--out-skey myvotevoting.skey \
	--out-vkey myvotevoting.vkey \
	--out-file myvotevoting.json

```  
  
BPの$HOME/CatalystVotingにある  
・myvotevoting.skey  
・myvotevoting.vkey  
・myvotevoting.json  
・cardano-signer  
の4ファイルをDLしてください。  

>・myvotevoting.skey  
>・myvotevoting.vkey  
>・myvotevoting.json  
>上記3ファイルはUSBに保存するなどバックアップを取っておいてください。  
>
>myvotevoting.json には、  
>Fund11から開始予定のWeb版Catalyst投票センターを使用する際に必要な復元フレーズが含まれています。  

___
## 2、登録メタデータを生成します。  
  
DLしたファイルの内、  
・myvotevoting.vkey  
・cardano-signer  
の2つを、エアギャップマシンの$NODE_HOMEにコピーします。  
`BP → myvotevoting.vkey/cardano-signer → エアギャップ`  
    
  
エアギャップにて以下コマンドを入力し、登録ファイルを生成します。  

`エアギャップ`
```
cd $NODE_HOME
./cardano-signer sign --cip36 \
	--payment-address $(cat payment.addr) \
	--vote-public-key myvotevoting.vkey \
	--secret-key stake.skey \
	--json \
	--out-file vote-registration.json

```
　

　  
エアギャップの$NODE_HOMEに生成された"vote-registration.json"を、BPの$HOME/CatalystVotingに移動します。  
  
`エアギャップ → vote-registration.json → BP`
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
    --metadata-json-file $HOME/CatalystVoting/vote-registration.json \
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
    --metadata-json-file $HOME/CatalystVoting/vote-registration.json \
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
bech32のディレクトリを改名します。
```
mv bech32-$(curl -s https://api.github.com/repos/input-output-hk/bech32/releases/latest | jq -r .tag_name | tr -d v) bech32

```
  
bech32をビルドします。
```
cd bech32
cabal build bech32

```
QRコードを作成します。 <4桁コード> の部分を任意の4桁数字に置き換えてから入力してください。

`BP`
```
cd $HOME/CatalystVoting
./catalyst-toolbox qr-code encode \
    --pin <4桁コード> \
    --input <(cat myvotevoting.skey | jq -r .cborHex | cut -c 5-132 | $(find $HOME/CatalystVoting/bech32 -type f -name "bech32") "ed25519e_sk") \
    --output myvote.qrcode.png \
    img

```


 

$HOME/CatalystVoting の中に"catalyst-qrcode.png"というファイルが作成されるので、DLします。
  
DLしたQRコードと設定した4桁pinコードを使用して、スマホアプリの"Catalyst Voting"にて登録を行えば完了です。  
お疲れさまでした。
_______
>作業後、エアギャップの  
>・myvotevoting.vkey  
>・cardano-signer  
>・vote-registration.json  
>は削除してもらっても大丈夫です。
  
削除コマンド  
`エアギャップ`
```
cd $NODE_HOME
rm myvotevoting.vkey cardano-signer vote-registration.json
```
