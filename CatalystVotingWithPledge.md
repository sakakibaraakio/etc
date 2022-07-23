# プール誓約金(payment.addr)を使用したCatalyst登録方法

>以下の手順では、作業中stake.skeyをサーバー上へ置いておく必要が出てきます。
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

nano catalystpayment.addr

```

___
## 3、jormungandrを導入します。

```
wget https://github.com/input-output-hk/jormungandr/releases/download/$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name)/jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

```

```
tar -xf jormungandr-$(curl -s https://api.github.com/repos/input-output-hk/jormungandr/releases/latest | jq -r .tag_name | tr -d v)-x86_64-unknown-linux-gnu-generic.tar.gz

```
___

## 4、エアギャップマシンの$NODE_HOME/stake.skeyを、BPの$HOME/CatalystVotingにコピーします。

___
## 5、登録メタデータを生成します。
```
cd $HOME/CatalystVoting
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')


voter-registration \
    --rewards-address $(cat $NODE_HOME/stake.addr) \
    --vote-public-key-file catalyst-vote.pkey \
    --stake-signing-key-file stake.skey \
    --slot-no $currentSlot \
    --json > voting-registration-metadata.json

```

___
## 6、トランザクションを作成、送信します。
```
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slot')
echo Current Slot: $currentSlot

```
```
cardano-cli query utxo \
    --address $(cat $HOME/CatalystVoting/catalystpayment.addr) \
    --mainnet > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr | sed -e '/lovelace + [0-9]/d' > balance.out

cat balance.out

```
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


