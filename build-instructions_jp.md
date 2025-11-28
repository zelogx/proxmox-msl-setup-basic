## ネットワーク構成図

docs/assets/zelog-MSL-Setup-withID.svg

## Proxmoxがインストールされていること
・PVE9.0.11にアップデートされていること。
・リポジトリがEnterpriseリポジトリはDisable, No-subscriptionリポジトリがceph.sources,proxmox.sourcesで有効になっていること。(Not Mandatory)

## ネットワーク設計・セグメント設計

以下のネットワーク設計は、出来なくても他に使用されていない事さえ保証できれば大丈夫。


a. MainLan(vmbr0既設) : 192.168.77.0/24: GW:.254
　→　自宅ラボ・家電 mainのcentos stream 10(.1 zelogx webサーバ, nextcloud, samba, 個人用OpenVPN/Wireguard, Unbound DNSなど), 家庭のalexaやTV, PS5, Internet router, 家族のPC, スマホなど様々)。
　後続の「Pritunlのmainlan側のIP」がこのIPレンジ内でなくてはならない。
　インターネットルータの多くはLAN側IPにしかポート転送できないので、インターネットルータの直下のLANに接続してあることが望ましい。
b. Proxmox PVEのmainlanのIP:192.168.77.2:インターネットルータへのstatic route追加時、宛先IPとなる。(自動取得、表示用)
c. vpndmzvn(新設): 192.168.80.0/24 GW:192.168.80.1 VPNクライアントが各開発PJ用サブネットへアクセスするための経路　最低/30のネットワークアドレスが必要。
d. VPNクライアントへの配布IPレンジ：192.168.81.0/24
	→wgとovpnで分けられる。例：192.168.81.2-126/25, 192.168.81.129-254/25
	これを更に「作成する開発用分離セグメントの作成数」で分割し/28とする。
	各PJ用にVPNできるクライアント数は最大13人となる。オフショア分散開発などはもっと多目に確保するとよい。

	List of IP address range and # of clients/PJ for OpenVPN
	| PJ  | Subnet            | IP range                      | # of VPN clients |
	| --: | ----------------- | ----------------------------- |-:|
	| 1   | 192.168.81.0/28   | 192.168.81.2–192.168.81.14    |13|
	| 2   | 192.168.81.16/28  | 192.168.81.18–192.168.81.30   |13|
	| 3   | 192.168.81.32/28  | 192.168.81.34–192.168.81.46   |13|
	| 4   | 192.168.81.48/28  | 192.168.81.50–192.168.81.62   |13|
	| 5   | 192.168.81.64/28  | 192.168.81.66–192.168.81.78   |13|
	| 6   | 192.168.81.80/28  | 192.168.81.82–192.168.81.94   |13|
	| 7   | 192.168.81.96/28  | 192.168.81.98–192.168.81.110  |13|
	| 8   | 192.168.81.112/28 | 192.168.81.114–192.168.81.126 |13|


	List of IP address range and # of clients/PJ for Wireguard
	| PJ  | Subnet            | IP range                      | # of VPN clients |
	| --: | ----------------- | ----------------------------- |-:|
	| 1   | 192.168.81.128/28 | 192.168.81.130–192.168.81.142 |13|
	| 2   | 192.168.81.144/28 | 192.168.81.146–192.168.81.158 |13|
	| 3   | 192.168.81.160/28 | 192.168.81.162–192.168.81.174 |13|
	| 4   | 192.168.81.176/28 | 192.168.81.178–192.168.81.190 |13|
	| 5   | 192.168.81.192/28 | 192.168.81.194–192.168.81.206 |13|
	| 6   | 192.168.81.208/28 | 192.168.81.210–192.168.81.222 |13|
	| 7   | 192.168.81.224/28 | 192.168.81.226–192.168.81.238 |13|
	| 8   | 192.168.81.240/28 | 192.168.81.242–192.168.81.254 |13|

e. 作成する開発用分離セグメントの作成数(PJ数):8（最低2で2のn乗2,4,8,16などになっている必要がある)
f. 各PJ(vnetpjxx)に割り当てるネットワークアドレス(新設): 172.16.16.0/20 : Project用セグメント。このIPレンジを「作成する開発用分離セグメントの作成数」で分割する。
　例：vnetpjxxに割り当てるネットワークアドレスが172.16.16.0/20、作成する開発用分離セグメントの作成数:8の場合、以下のように分割される
	vnetpj01 172.16.16.0/24 GW:172.16.16.254
	vnetpj02 172.16.17.0/24 GW:172.16.17.254
	vnetpj03 172.16.18.0/24 GW:172.16.18.254
	vnetpj04 172.16.19.0/24 GW:172.16.19.254
	vnetpj05 172.16.20.0/24 GW:172.16.20.254
	vnetpj06 172.16.21.0/24 GW:172.16.21.254
	vnetpj07 172.16.22.0/24 GW:172.16.22.254
	vnetpj08 172.16.23.0/24 GW:172.16.23.254
	vnetpjxx内のVM群(172.16.16.0/24)は、上記セグメント内で通信は自由。
	このVMへのFW設定はSecurity Group(SG)で制御される。
	Pritunlのorgに対応
g. Pritunlのmainlan側のIP:192.168.77.9:インターネットルータへのポートフォワード追加時の転送先IPとなる。
h. Pritunlのvpndmzvn側のIP:192.168.80.2:Pritunlのクライアントが各PJ用サブネットに出ていくときのサブネットです。最低/30あれば間に合いますがここでは大きく/24で取ってます。
i. UDP port numbers for OpenVPN and Wireguard



### メインルータ
port forwardを設定します。OpenVPN用 x PJ数分
 OpenVPN   UDP 11856-11863(*1) → Pritunlのmainlan側のIP(192.168.77.9)
 Wireguard UDP 15952-15959(*1) → Pritunlのmainlan側のIP(192.168.77.9)

*1:任意のUDPポート。作成する開発用分離セグメントの作成数分(8)xOpenVPN+Wireguard分(2)
内訳
|pj|OVPN(udp)|WG(udp)|
|-:|--------:|------:|
|01|11856|15952|
|02|11857|15953|
|03|11858|15954|
|04|11859|15955|
|05|11860|15956|
|06|11861|15957|
|07|11862|15958|
|08|11863|15959|

static route
  route 192.168.80.0/24 via 192.168.77.2
  route 172.16.16.0/20 via 192.168.77.2

### Proxmox SDNブリッジ追加
#### vpn-dmz-vnet（192.168.80.0/24）追加
Datacenter → SDN → Zones → Add → Simple
ID:vpndmz vpn着信用
ID:devpj  開発ゾーン用

#### VNet（仮想L2セグメント）の作成

##### DMZ用（vpn-dmz-vnet）
Datacenter → SDN → VNets → Create
ID: vpndmzvn
Alias: vpn-dmz-vnet
Zone: vpndmz
VLAN / Tag: （未設定のまま。Simpleなら不要）

Subnets
Subnet:192.168.80.0/24
GW:192.168.80.1

##### 開発LAN用
Datacenter → SDN → VNets → Create
ID: (以下開発LAN Vnet ID参照)
Zone : devpj
Vnet ID  Subnet         GW
vnetpj01 172.16.16.0/24 172.16.16.254
vnetpj02 172.16.17.0/24 172.16.17.254
vnetpj03 172.16.18.0/24 172.16.18.254
vnetpj04 172.16.19.0/24 172.16.19.254
vnetpj05 172.16.20.0/24 172.16.20.254
vnetpj06 172.16.21.0/24 172.16.21.254
vnetpj07 172.16.22.0/24 172.16.22.254
vnetpj08 172.16.23.0/24 172.16.23.254

##### Apply
上記作成後、設定を適用
Datacenter > SDN
"Apply"

GUI → Datacenter → Firewall → IPSet
[IPSET all_private_ip] # all_private_ip
10.0.0.0/8
127.0.0.0/8
172.16.0.0/12
192.168.0.0/16

[IPSET devpjs] # 172.16.16.0/20
172.16.16.0/20
[IPSET mainlan] # 192.168.77.0/24
192.168.77.0/24
[IPSET vpn_guest_pool] # 192.168.81.0/24
192.168.81.0/24

Proxmox FirewallでACLを構成
GUI → Datacenter → Firewall → Options
Fireall: Yes
Input policy:DROP
Output policy:ACCEPT
Forward policy:ACCEPT


DataCenter, hostのFirewall-OptionsでFirewall:Yesにする事

> 重要：上記を適用すると、nested PVEがある場合、nested pve上のVMに
アクセス出来なくなるので、nested PVE VMのFirewall-OptionsでMAC FilterをOFFにすること。


#開発用LANのGWからProxmoxに出来ないようにする
GUI → <HOST> → Firewall → Options
Firewall:Yes  
nftables:Yes

GUI → <HOST> → Firewall → Firewall
Rules:
# On  Type  Action  Macro  Protocol  Source             S.Port  Destination       D.Port  Log level
# -------------------------------------------------------------------------------------------------
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj08-all  -       +sdn/vnetpj08-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj07-all  -       +sdn/vnetpj07-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj06-all  -       +sdn/vnetpj05-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj05-all  -       +sdn/vnetpj05-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj04-all  -       +sdn/vnetpj04-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj03-all  -       +sdn/vnetpj03-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj02-all  -       +sdn/vnetpj02-all  -       nolog
✓    FORWARD ACCEPT  -      -         +sdn/vnetpj01-all  -       +sdn/vnetpj01-all  -       nolog
✓    FORWARD ACCEPT  -      tcp       +dc/devpjs         -       192.168.77.1       53      nolog
✓    FORWARD ACCEPT  -      udp       +dc/devpjs         -       192.168.77.1       53      nolog
✓    FORWARD DROP    -      -         +dc/devpjs         -       +dc/all_private_ip -       nolog




##### vpndmzvn に Pritunl VM を配置
	NIC1: vmbr0     IP: 192.168.77.9
	NIC1: vpndmzvn  IP: 192.168.80.2
	ルーティング＋Firewall定義
	WG: 11856/UDP, OVPN: 15952/UDP
	* VM内部に192.168.81.0/24のOpenVPN,WGクライアント用IPレンジを持つ。

##### Ubuntu24.04 をminimalでインストール
Snapshot
sudo apt update -y
sudo apt upgrade -y
sudo shutdown -h 0
Snapshot

> Refer to : https://docs.pritunl.com/kb/vpn/getting-started/installation

sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF

sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF

sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF

sudo apt --assume-yes install gnupg

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes
curl -fsSL https://swupdate.openvpn.net/repos/repo-public.gpg | sudo gpg -o /usr/share/keyrings/openvpn-repo.gpg --dearmor --yes
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
sudo apt update
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools

sudo ufw disable

sudo systemctl start pritunl mongod
sudo systemctl enable pritunl mongod

sudo sh -c 'echo "* hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "* soft nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root soft nofile 64000" >> /etc/security/limits.conf'


# Mongo をローカルのみ bind
sudo sed -i "s/^ *bindIp: .*$/  bindIp: 127.0.0.1/" /etc/mongod.conf
sudo systemctl restart mongod
sudo ss -ltnp | grep 27017


# サービスとポート確認
ss -ltnp | egrep '9700|7774|7775|27017'





##### setup key取得
masa@pritunl:~$ sudo pritunl setup-key
b41ac4f73e034262a504d0b1bed96d37

##### ブラウザで192.168.77.9にアクセス
setup keyを入れる

##### デフォルトパスワード取得
masa@pritunl:~$ sudo pritunl default-password
[local][2025-10-10 11:16:57,307][INFO] Getting default administrator password
Administrator default password:
  username: "pritunl"
  password: "xxxxxxxxxxx"

##### ブラウザで192.168.77.9にアクセス
ログイン

PublicIP, user, passwordを投入。
PublicIP(またはFQDN)は、VPN接続時のIPとなる。


##### Pritunl（GUI）

##### Create Org
Users -> Add organization -> TestPrj1

##### Add user
Users -> 
Name: testuser1
Organization: TestPrj1
email: masaaki.harada@gmail.com
Pin: 

##### Create server
Servers -> Add Server
	Name: Server01
	Port: 11856    <---- OpenVPN用ポート番号(下記port table表参照)
	Protocol: udp <---- OpenVPN用proto
	✅ Enable WireGuard
	WG Port: 15952 ← 実際に待受するWGポート(下記port table表参照)
	DNS Server: 1.1.1.1
	Virtual Network: 192.168.81.0/28 ← OpenVPN用クライアント配布セグメント
		(下記Virtual Network List表参照)
	Virtual WG Network: 192.168.81.128/28 ← Wireguard用クライアント配布セグメント
		(下記Virtual WG Network List表参照)
Advanced（歯車アイコン）
	Restrict Routing(Split Tunnel): ON(default)
	→ チェックを入れると、Server→Routes に登録した宛先だけをVPN経由にします。
	Inter-Client Routing(User Isolation): Off
	→ ユーザー同士の通信を許可の意味。ユーザー隔離にしたいならチェックを外す
	Bind Address: 192.168.77.9

Port table
pj	OVPN(udp)	WG(udp)
01	11856	15952
02	11857	15953
03	11858	15954
04	11859	15955
05	11860	15956
06	11861	15957
07	11862	15958
08	11863	15959

Virtual Network List for OpenVPN
| No. |サブネット         | IP範囲                        | 1PJ用のクライアント数 |
| --- | ----------------- | ----------------------------- |-:|
| 1   | 192.168.81.0/28   | 192.168.81.2–192.168.81.14    |13|
| 2   | 192.168.81.16/28  | 192.168.81.18–192.168.81.30   |13|
| 3   | 192.168.81.32/28  | 192.168.81.34–192.168.81.46   |13|
| 4   | 192.168.81.48/28  | 192.168.81.50–192.168.81.62   |13|
| 5   | 192.168.81.64/28  | 192.168.81.66–192.168.81.78   |13|
| 6   | 192.168.81.80/28  | 192.168.81.82–192.168.81.94   |13|
| 7   | 192.168.81.96/28  | 192.168.81.98–192.168.81.110  |13|
| 8   | 192.168.81.112/28 | 192.168.81.114–192.168.81.126 |13|


Virtual Network List for Wireguard
| No. | サブネット        | IP範囲                        | 1PJ用のクライアント数|
| --- | ----------------- | ----------------------------- |-:|
| 1   | 192.168.81.128/28 | 192.168.81.130–192.168.81.142 |13|
| 2   | 192.168.81.144/28 | 192.168.81.146–192.168.81.158 |13|
| 3   | 192.168.81.160/28 | 192.168.81.162–192.168.81.174 |13|
| 4   | 192.168.81.176/28 | 192.168.81.178–192.168.81.190 |13|
| 5   | 192.168.81.192/28 | 192.168.81.194–192.168.81.206 |13|
| 6   | 192.168.81.208/28 | 192.168.81.210–192.168.81.222 |13|
| 7   | 192.168.81.224/28 | 192.168.81.226–192.168.81.238 |13|
| 8   | 192.168.81.240/28 | 192.168.81.242–192.168.81.254 |13|
192.168.77.9
##### 開発LAN用
Vnet     Subnet         GW
vnetpj01 172.16.16.0/24 172.16.16.254
vnetpj02 172.16.17.0/24 172.16.17.254
vnetpj03 172.16.18.0/24 172.16.18.254
vnetpj04 172.16.19.0/24 172.16.19.254
vnetpj05 172.16.20.0/24 172.16.20.254
vnetpj06 172.16.21.0/24 172.16.21.254
vnetpj07 172.16.22.0/24 172.16.22.254
vnetpj08 172.16.23.0/24 172.16.23.254

##### Server Routes：
0.0.0.0/0を削除
172.16.16.0/24 … 172.16.31.0/24 を NAT=OFF で列挙（/20は入れない）。
開発用LAN参照

##### Create Org & attach to server
Pritunlにログインします
https://<PritunlのIP>/
User:Pritunl構築時に作成時に指定したユーザ名
Pass:Pritunl構築時に作成時に指定したパスワード

Users -> Add Organization
Name: 任意のorganization名 (例：pj01,pj02,...pj08を作成。*1)

> *1 PJ IDと一対一が望ましい。
> ユーザが属している実際の企業名を指定するのは推奨できません。
> ここはプロジェクト名を指定する方が運用が楽になります。

OrgをPJ単位で作成→サーバにAttach（受付ON/OFFはここで切替）。
Servers -> Attach Organization
Select an organization:

##### Pritunl GUIのListen address変更
sudo vi /etc/pritunl.conf
	"bind_addr": "192.168.77.9", <---- 0.0.0.0から変更

##### 192.168.80.1からpritunlにssh出来ないようにsshdのlistenアドレスを変更
sudo vi /etc/ssh/sshd_config
ListenAddress 192.168.77.9    <------ 追加する

systemctl restart ssh
reboot

##### 0.0.0.0でlistenしているプロセスが無いことを確認
sudo ss -lntp | grep '0\.0\.0\.0'

#### PJVMからGW(172.16.1X.254)にpingが通らない時
Node(pve1) → Firewall → Rules → Add で一番上に: 
Type: in 
Action: ACCEPT 
Protocol: icmp

#### 疎通テスト（3ステップ）
77系→PJ：ping/traceroute（通る）
PJ→77系：ping（落ちる／DNSは通る）
VPNクライアント→PJ：ssh/http（通る）

#### 開発用VMには必ずFWを有効にしSecurity Groupの設定を行う事
VM → Firewall → insert: Security Group
Security Group: pj-dev
Enable: checked

VM → Firewall → Options
Firewall: Yes


# クライアントコンフィグの接続先IPがlocal addressになってしまう
# Sync Address は UI が無いので MongoDB で直指定：
sudo pritunl mongodb   # mongoシェルを開く（または mongosh）
use pritunl
db.hosts.updateMany({}, {$set: {sync_address: "pritunl.zelogx.com"}})
db.hosts.updateMany({}, {$set: {public_address: "ma.zelogx.com"}})  # 念のため揃える
quit
sudo systemctl restart pritunl

# クライアントがVPNを通してDNSを引きに行く
# DNS-IPを配布するのを止める
sudo pritunl set vpn.dns_route false
sudo systemctl restart pritunl

# NATをやめて純ルーティングにする（該当サーバだけ）
# SIDの確認
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.find({}, {name:1}).forEach(d => print(d.name, d._id.toHexString()))
'
Server01 68e9993dd42cf24361d6cf31
Server02 68e9b49376a47b6c2b180a3b
Server03 68f16ccc3b853aa0ced8694b
Server04 68f16d093b853aa0ced86980
Server05 68f16d553b853aa0ced869be
Server06 68f16d863b853aa0ced869eb
Server07 68f16db73b853aa0ced86a1a
Server08 68f16dea3b853aa0ced86a47

# NAT状態を見る
sid="68e9993dd42cf24361d6cf31"   # Server01 のID
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
const sid=ObjectId("'$sid'");
db.servers.find({_id:sid},{name:1,network:1,routes:1}).forEach(doc=>printjson(doc));
'
{
  _id: ObjectId('68e9993dd42cf24361d6cf31'),
  name: 'Server01',
  network: '192.168.81.0/28',
  routes: [
    {
      network: '172.16.16.0/24',
      comment: null,
      metric: null,
      nat: true,
      nat_interface: null,
      nat_netmap: null,
      advertise: false,
      vpc_region: null,
      vpc_id: null,
      net_gateway: false,
      server_link: false
    }
  ]
}

# 全sid分のnatをfalseに書き換える
sudo mongosh "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.updateMany(
  {},
  { $set: { "routes.$[].nat": false } }
);
'

sudo iptables -t nat -S | grep -i pritunl
# MASQUERADEが出てこなくなれば順ルーティング

sudo systemctl restart pritunl

Pritunl内のVPN Client IP pool (192.168.81.0/24)はpve1から見えないので
戻り経路を定義してやる必要がある。NATが有効の時はpritunlのNIC192.168.80.2の
アドレスにNATされて出ていくため、以下の設定が無くても問題ない。

pve1で
ip route add 192.168.81.0/24 via 192.168.80.2
と永続化
vi /etc/network/interfaces.d/sdn
auto vpndmzvn
iface vpndmzvn
        address 192.168.80.1/24
        bridge_ports none
        bridge_stp off
        bridge_fd 0
        alias vpn-dmz-vnet
        ip-forward on
        post-up ip route add 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
        pre-down ip route del 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn





## アイドル中のCPU使用率が高い場合、
sudo vi /etc/pritunl.conf
以下を追加
    "mongodb_poll_interval": 30,
sudo systemctl restart pritunl



## PritunlのOrganization

> Organization追加時、ユーザが属している実際の企業名を指定するのは推奨できません。
> これは、pritunlの仕様上、Organizationを各PJ用のVPN Serverに割り当てるためであり、
> Organizationのユーザを個別にVPN Serverに割り当てることが出来ないためです。
Users -> Add Organization
Name: 任意のorganization名 (PJ IDと一対一が望ましい)

まず、今回のセットアップでは、以下のようにPritunlのVPN Server一つは一つの開発用ネットワークに1:1で割り当てられています。
|Pritunl VPN Server|開発用ネットワーク|
|------------------|------------------|
|Server01|vnetpj01|
|Server02|vnetpj02|
|...|...|
|Server08|vnetpj08|

また、PritunlのOrganization,userの設定を以下のように設定したとします。
|Org.|User|
|----|----|
|OrgA|UserAA|
|OrgA|UserAB|
|OrgA|UserAC|
|OrgB|UserBA|
|OrgB|UserBB|
|OrgB|UserBC|

Pritunl VPN ServerにはOrganization単位でのみVPNユーザを割り当てることが可能です。
そのため、Server01(vnetpj01)にUserAAとUserBAだけを割当てるという事ができません。
上記の場合、可能なのは、Server01(vnetpj01)にアクセス出来るのはOrgA全員、またはOrgB全員（またはその両方）
