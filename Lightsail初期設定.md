# Lightsailにおける、構築マニュアル 1-Ubuntu初期設定
Lightsailでは独自の初期設定が為されているので、1-Ubuntu初期設定 の項目ではマニュアルと手順が一部異なります。  
下記の手順をご参考ください。（2-ノードインストール 以降の作業はマニュアルと同じです。）
>lightsailアカウント（WEB）のセキュリティは厳重にしておきましょう。（2段階認証など）
_______
### 1、ターミナルソフトを用意します。
1.R-Login(Windows) https://kmiya-culti.github.io/RLogin/

2.Terminal(Mac)
https://www.webdesignleaves.com/pr/plugins/mac_terminal_basics_01.html 
<br>
<br>
____
### 2、LightsailのWEB管理画面の”インスタンスの作成”からサーバーを新規作成します。
”プラットフォーム”は「Linux/Unix」  
”設計図の選択”は「OSのみ」からの「Ubuntu 20.04 LTS」  
”インスタンスプランの選択”は、カルダノサーバーの最小構成メモリを上回るプランを選択してください。  
SSHキーペアをダウンロードします。（項目5 にてSSHキーを新しく作り直すのでデフォルトキーでも大丈夫です。） 　　
　  
*<details><summary>ユーザー名を初期設定の”ubuntu”から変更したい方向け（中上級者用）</summary>*
「オプション」の「起動スクリプトの追加」を選択し、

```
OLD_USER=ubuntu

NEW_USER=<新ユーザー名>

usermod -l $NEW_USER -md /home/$NEW_USER -c $NEW_USER  $OLD_USER

groupmod -n $NEW_USER $OLD_USER

echo $NEW_USER:$NEW_USER | chpasswd
passwd -e  $NEW_USER
```
上記を、<新ユーザー名>の箇所を置き換えてから入力してください。

その後ターミナルソフトにて、  
ユーザー名→新ユーザー名  
パスワード→新ユーザー名  
でログインします。  
Current password:   
の表示が出るので、新ユーザー名を入力します。  
その後、新しいパスワードとなる文字列を入力し設定してください。  

一度SSH接続が切れるので、  
ユーザー名→新ユーザー名  
パスワード→新しいパスワード  
でログインしなおせば完了です。  
</details>
<br>


BP用サーバーとリレー用サーバーを用意します。
>最小構成：BP用サーバー1台、リレー用サーバー1台
  
　  
____
### 3、WEB管理画面から静的IPをサーバーの数だけ取得し、各サーバーにアタッチしてください。
WEB管理画面の上側の”ホーム”→”ネットワーキング”→”静的IPの作成”

　  
____
### 4、ターミナルソフトを使用して、構築したサーバーへ接続します。
静的IPを使用してログインします。  
ログインユーザー名は「ubuntu」（ユーザー名を変更した方は変更後の名前を入力）  
SSH認証鍵は、サーバー構築時にダウンロードしたものを指定してください。
　  
　  
_____
### 5、SSH鍵をed25519方式に変更します。
ペア鍵の作成
```
ssh-keygen -t ed25519 -N '' -C ssh_connect -f ~/.ssh/ssh_ed25519
```
```
cd ~/.ssh
ls

```
>戻りが、  
>`authorized_keys  ssh_ed25519  ssh_ed25519.pub`  
>であることを確認します。
  
　  

```
cd ~/.ssh/
cat ssh_ed25519.pub > authorized_keys
chmod 600 authorized_keys
chmod 700 ~/.ssh

```
SSH鍵ファイルをダウンロードします。

>1.R-loginの場合はファイル転送ウィンドウを開く  
>2.左側ウィンドウ(ローカル側)は任意の階層にフォルダを作成する。  
>3.右側ウィンドウ(サーバ側)は「.ssh」フォルダを選択する  
>4.右側ウィンドウから、ssh_ed25519とssh_ed25519.pubの上で右クリックして「ファイルのダウンロード」を選択する  
>5.一旦サーバからログアウトする（ターミナルソフトを終了する）  
>6.R-Loginのサーバ接続編集画面を開き、「SSH認証鍵」をクリックし4でダウンロードしたssh_ed25519ファイルを選ぶ  
>7.サーバへ接続する  
  
**※注意※**  
**DLしたssh_ed25519は、紛失するとサーバーへのログインが不可能になる重要なキーです。  
今後構築を進めていくとプール管理において重要なキーを作成するので、それと共に複数のUSBに保存しバックアップを取ってください。**


_____
### 6、SSHデフォルトポート(22)を変更します。
/etc/ssh/sshd_configファイルを開きます。
```
sudo nano /etc/ssh/sshd_config
```
　  
`#Port 22`  
の行の、最初の#を削除してから、
22の数字を49513～65535までの間のランダムな数字に変更します。  
その後Ctrlキー+o 同時押しからのエンターで変更を保存し、Ctrl+xでファイルを閉じます。 
>変更した数字をメモしておきましょう。

  
　  
SSH構文にエラーがないかチェックします。  
```
sudo sshd -t
```
　  
SSH構文エラーがない場合、SSHプロセスを再起動します。  
```
sudo service sshd reload
```
　  
一旦、ログオフします。  
```
exit
```
　  
____
### 7、WEB管理画面からポートを開放します。
AWS系のVPSは独自のファイアウォールが設定されている為、  
ポートの開放はターミナルソフトからの操作で行うのではなく、  
各サーバーのインスタンス管理画面（WEB）の”ネットワーキング”内の”IPv4 ファイアウォール”から設定する必要があります。  
サーバーの種類に合わせ下記の通りにポートを開放してください。  
>初期で開放されている、22番と80番は削除してください。   
>IPv6ネットワーキング機能は無効にしてください。
  
  
　  

BP用サーバー
| アプリケーション | プロトコル | ポートまたは範囲 |
:----|:----|:----
|カスタム | TCP | 先ほど設定した変更後のSSH用ポート番号（SSH接続用） |
|カスタム | TCP | 49513～65535までの任意番号（カルダノノード用） ※リレーIP制限 |
|カスタム | TCP | 9100（Prometheus用） ※リレーIP制限 |
|カスタム | TCP | 12798（Prometheus用） ※リレーIP制限 |
|カスタム | UDP | 123（Chrony用) |
|Ping (ICMP) | ICMP |
  
>※リレーIP制限  
>”IP アドレスに制限する”にチェックを入れ、出てきた入力欄にリレーノードの静的IPを入力してください。
  

**※注意※**  
**BPにおけるカルダノノード用の任意番号は、SSH接続用のポート番号とは別にしてください。  
ここでBPにて設定したカルダノノード用の任意番号は、2-4. ノード起動スクリプトの作成　にて使用しますのでメモしておいてください。**  

リレー用サーバー
| アプリケーション | プロトコル | ポートまたは範囲 |
:----|:----|:----
|カスタム | TCP | 先ほど設定した変更後のSSH用ポート番号（SSH接続用） |
|カスタム | TCP | 6000（カルダノノード用）|
|カスタム | TCP | 3000（Grafana用) |
|カスタム | UDP | 123（Chrony用) |
|Ping (ICMP) | ICMP |

  
*<details><summary>リレーサーバーの2台目以降を作成する場合</summary>*
リレー2台目以降サーバー
| アプリケーション | プロトコル | ポートまたは範囲 |
:----|:----|:----
|カスタム | TCP | 先ほど設定した変更後のSSH用ポート番号（SSH接続用） |
|カスタム | TCP | 6000（カルダノノード用）|
|カスタム | TCP | 9100（Prometheus用） ※リレー1 IP制限 |
|カスタム | TCP | 12798（Prometheus用） ※リレー1 IP制限 |
|カスタム | UDP | 123（Chrony用) |
|Ping (ICMP) | ICMP |

また、BP用サーバーのファイアウォール設定を
| アプリケーション | プロトコル | ポートまたは範囲 |
:----|:----|:----
|カスタム | TCP | 先ほど設定した変更後のSSH用ポート番号（SSH接続用） |
|カスタム | TCP | 49513～65535までの任意番号（カルダノノード用） ※リレー1 IP制限 |
|カスタム | TCP | ↑と同じ番号（カルダノノード用） ※リレー2 IP制限 |
|カスタム | TCP | 9100（Prometheus用） ※リレー1 IP制限 |
|カスタム | TCP | 12798（Prometheus用） ※リレー1 IP制限 |
|カスタム | UDP | 123（Chrony用) |
|Ping (ICMP) | ICMP |

のように、所持している全てのリレーのIP制限でカルダノノード用ポート番号を開放するようにします。 


</details>
<br>
　  

>ここでは、今後の構築作業において必要なポート開放をすべて纏めて行っています。  
>なので以後の構築手順における
>```
>sudo ufw allow xxxxx
>sudo ufw reload
>```
>のようなufwのポート開放のコマンドは実行せずに飛ばしてもらって大丈夫です。  
>（3-リレー/BPの接続　と　9.監視ツールセットアップ　にて出てきます。）

____
### 8、ターミナルソフトからサーバーに接続します。
ターミナルソフトの接続設定画面でSSHポート番号を変更してから接続してください。
　  
___
### 9、システムを更新します。
※不正アクセスを予防するには、システムに最新のパッチを適用することが重要です。  
```
sudo apt update -y && sudo apt upgrade -y
```
>ピンク色の画面が出てきたら、  
` keep the local version currently installed`  
のままエンターを選択します。
  
```
sudo apt autoremove
sudo apt autoclean

```
>質問にはyを入力してEnter

自動更新を有効にすると、手動でインストールする手間を省けます。  
```
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

```
>YESを選択しEnter
  
____
### 10、システムで共有されるメモリを保護します。
/etc/fstabを開きます。  
```
sudo nano /etc/fstab
```
次の行をファイルの最後に追記して保存します。
```
tmpfs   /run/shm    tmpfs   ro,noexec,nosuid    0 0
```

____
### 11、Swap領域を設定します。
```
cd $HOME
sudo fallocate -l 16G /swapfile

sudo chmod 600 /swapfile

sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show

sudo cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
cat /proc/sys/vm/vfs_cache_pressure
cat /proc/sys/vm/swappiness

```

システムを再起動します。
```
sudo reboot
```

____
### 12、Fail2banをインストールします。
>Fail2banは、ログファイルを監視し、ログイン試行に失敗した特定のパターンを監視する侵入防止システムです。
>特定のIPアドレスから（指定された時間内に）一定数のログイン失敗が検知された場合、
>Fail2banはそのIPアドレスからのアクセスをブロックします。
  

```  
sudo apt install fail2ban -y
```  
　  
SSHログインを監視する設定ファイルを開きます。
```
sudo nano /etc/fail2ban/jail.local
```  
　  
ファイルの最後に次の行を追加し保存します。
>コマンド中の <SSHポートを入力してください> の部分を設定したSSHポート番号に置き換えてから入力してください。<>は不要です。  
```
[sshd]
enabled = true
port = <SSHポートを入力してください>
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```
  
fail2banを再起動して設定を有効にします。  
```
sudo systemctl restart fail2ban
```
　  

_____
### 13、Chronyを設定します。
```
sudo apt-get install chrony
```

```
sudo cat > $HOME/chrony.conf << EOF

server 169.254.169.123 prefer iburst minpoll 4 maxpoll 4
pool time.google.com       iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool time.facebook.com     iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool time.euro.apple.com   iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool time.apple.com        iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool ntp.ubuntu.com        iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 5.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 0.1 -1

# Get TAI-UTC offset and leap seconds from the system tz database
leapsectz right/UTC

# Serve time even if not synchronized to a time source.
local stratum 10
EOF

```

```
sudo mv $HOME/chrony.conf /etc/chrony/chrony.conf

sudo systemctl restart chronyd.service

```
　  

_____
### 14、SSHの2段階認証を設定します。（任意）
>こちらの導入は必須ではありません。  
>導入する場合、事前にお手元のスマートフォンに「Google認証システムアプリ」のインストールが必要です
  
**設定に失敗するとログインできなくなる場合があるので、設定前に二つ目のターミナルウィンドウでサーバーにログインしておいてください。  
万が一ログインできなくなった場合、復旧できます。**

```
sudo apt update
sudo apt upgrade

```
```
sudo apt install libpam-google-authenticator -y
```  
　  
SSHがGoogle Authenticator PAM モジュールを使用するために、/etc/pam.d/sshdファイルを編集します。  
```
sudo nano /etc/pam.d/sshd 
```  
　  
4行目の @include common-authの先頭へ#を付与してコメントアウトします。  
`#@include common-auth`
　  
最後の行に1行追加します。  
`auth required pam_google_authenticator.so`  
　  
以下を使用してsshdデーモンを再起動します。  
```
sudo systemctl restart sshd.service
```
　  
/etc/ssh/sshd_config ファイルを開きます。  
```
sudo nano /etc/ssh/sshd_config
```  
　  
ChallengeResponseAuthenticationの項目を「yes」にします。  
`ChallengeResponseAuthentication yes`  
  
  
最後の行に1行追加します。  
`AuthenticationMethods publickey,keyboard-interactive`  
　  
ファイルを保存して閉じます。  
　  
以下を使用してsshdデーモンを再起動します。  
```
sudo systemctl restart sshd.service
```  
　  
google-authenticator コマンドを実行します。  
```
google-authenticator
```  
　  
質問事項が5つ表示されます。  
途中で大きなQRコードが表示されますが、その下には緊急時のスクラッチコードが表示されますので、忘れずに書き留めておいて下さい。  
表示されるQRコードはスマートフォンのGoogle認証システムアプリで読み取り、2段階認証を機能させます。  
　  
質問には、  
y→y→y→n→y  
の順番で回答してください。  
　  
完了したら、一度接続を閉じます。
```
exit
```
  
サーバーへ再接続し、スマートフォンのGoogle認証システムアプリに表示される6桁の認証番号を入力して接続が成功すれば完了です。  
_____
  
**1-Ubuntu初期設定は以上です。お疲れさまでした。  
引き続き、2-ノードインストール 以降を進めていってください。**  
https://docs.spojapanguild.net/setup/2-node-setup/
