# Multiverse Secure Lab(MSL) Setup for Proxmox by Zelogx™ - The Multi-tenant Enabler Step by step building guide

[![GitHub Discussions](https://img.shields.io/badge/GitHub-Discussions-181717?logo=github)](https://github.com/zelogx/msl-setup/discussions)
[![Ofiicial Site](https://img.shields.io/badge/オフィシャル-サイト-blue)](https://www.zelogx.com/ja/)
[![Release Notes](https://img.shields.io/badge/リリース-ノート-green)](https://www.zelogx.com/ja/documents/release-notes/)

## 目次

- [I. ネットワーク構成図](#ネットワーク構成図)
- [II. Proxmoxがインストールされていること](#proxmoxがインストールされていること)
- [III. ネットワーク設計・セグメント設計](#ネットワーク設計セグメント設計)
	- [III.1. メインルータ](#メインルータ)
	- [III.2. Proxmox SDNブリッジ追加](#proxmox-sdnブリッジ追加)
		- [III.2.1. vpn-dmz-vnet（192.168.80.0/24）追加](#vpn-dmz-vnet192.168.80.024追加)
		- [III.2.2. VNet（仮想L2セグメント）の作成](#vnet仮想l2セグメントの作成)
			- [III.2.2.1. DMZ用（vpn-dmz-vnet）](#dmz用vpn-dmz-vnet)
			- [III.2.2.2. 開発LAN用](#開発lan用)
				- [III.2.2.2.1. Create Vnet](#create-vnet)
				- [III.2.2.2.2. Add Subnet](#add-subnet)
			- [III.2.2.3. Apply](#apply)
- [1. vpndmzvn に Pritunl VM を配置](#vpndmzvn-に-pritunl-vm-を配置)
- [2. Ubuntu24.04 をminimalでインストール](#ubuntu24.04-をminimalでインストール)
- [3. Mongo をローカルのみ bind](#mongo-をローカルのみ-bind)
- [4. サービスとポート確認](#サービスとポート確認)
- [5. setup key取得](#setup-key取得)
- [6. ブラウザで192.168.77.9にアクセス](#ブラウザで192.168.77.9にアクセス)
- [7. デフォルトパスワード取得](#デフォルトパスワード取得)
- [8. ブラウザで192.168.77.9にアクセス](#ブラウザで192.168.77.9にアクセス-1)
- [9. Pritunl（GUI）](#pritunlgui)
	- [9.1. Create Org](#create-org)
	- [9.2. Add user](#add-user)
	- [9.3. Create server](#create-server)
	- [9.4. 開発LAN用](#開発lan用-1)
	- [9.5. Server Routes：](#server-routes)
	- [9.6. Create Org & attach to server](#create-org-attach-to-server)
	- [9.7. Pritunl GUIのListen address変更](#pritunl-guiのlisten-address変更)
	- [9.8. 192.168.80.1からpritunlにssh出来ないようにsshdのlistenアドレスを変更](#192.168.80.1からpritunlにssh出来ないようにsshdのlistenアドレスを変更)
	- [9.9. 0.0.0.0でlistenしているプロセスが無いことを確認](#0.0.0.0でlistenしているプロセスが無いことを確認)
- [10. PJVMからGW(172.16.1X.254)にpingが通らない時](#pjvmからgw172.16.1x.254にpingが通らない時)
- [11. 疎通テスト（3ステップ）](#疎通テスト3ステップ)
- [12. 開発用VMには必ずFWを有効にしSecurity Groupの設定を行う事](#開発用vmには必ずfwを有効にしsecurity-groupの設定を行う事)
- [13. クライアントコンフィグの接続先IPがlocal addressになってしまう](#クライアントコンフィグの接続先ipがlocal-addressになってしまう)
- [14. Sync Address は UI が無いので MongoDB で直指定：](#sync-address-は-ui-が無いので-mongodb-で直指定)
- [15. クライアントがVPNを通してDNSを引きに行く](#クライアントがvpnを通してdnsを引きに行く)
- [16. DNS-IPを配布するのを止める](#dns-ipを配布するのを止める)
- [17. NATをやめて純ルーティングにする（該当サーバだけ）](#natをやめて純ルーティングにする該当サーバだけ)
	- [17.1. SIDの確認](#sidの確認)
	- [17.2. NAT状態を見る](#nat状態を見る)
	- [17.3. 全sid分のnatをfalseに書き換える](#全sid分のnatをfalseに書き換える)
	- [17.4. MASQUERADEが出てこなくなれば順ルーティング](#masqueradeが出てこなくなれば順ルーティング)
- [18. アイドル中のCPU使用率が高い場合、](#アイドル中のcpu使用率が高い場合)
- [19. PritunlのOrganization](#pritunlのorganization)
- [20. Goal of v1.1.0](#goal-of-v1.1.0)
- [21. PJ01のVPNユーザがPJ01内にVMを作れるまで](#pj01のvpnユーザがpj01内にvmを作れるまで)
	- [21.1. Group, Pool, Userの作成](#group-pool-userの作成)
		- [21.1.1. Poolの作成](#poolの作成)
		- [21.1.2. Groupの作成](#groupの作成)
		- [21.1.3. Userの作成](#userの作成)
	- [21.2. GroupにPermissionとRoleを与える](#groupにpermissionとroleを与える)
	- [21.3. Poolにリソース追加](#poolにリソース追加)
		- [21.3.1. リソースpool(PJ01)にリソースを追加する](#リソースpoolpj01にリソースを追加する)
			- [21.3.1.1. 既存VMを割り当てる](#既存vmを割り当てる)
			- [21.3.1.2. ストレージを割り当てる](#ストレージを割り当てる)
			- [21.3.1.3. SDN Zone ネットワークを割り当てる](#sdn-zone-ネットワークを割り当てる)
				- [21.3.1.3.1. [暫定解決策]devpj zoneを一個しか作らなかった場合の、以下の方法で割り当て可能（ただし、VPNユーザが新たなvnetの作成が出来ない）](#暫定解決策devpj-zoneを一個しか作らなかった場合の以下の方法で割り当て可能ただしvpnユーザが新たなvnetの作成が出来ない)
	- [21.4. VPNユーザがProxmox dashboardにアクセス出来るようにする](#vpnユーザがproxmox-dashboardにアクセス出来るようにする)
		- [21.4.1. Add Node level FW rules as below](#add-node-level-fw-rules-as-below)
		- [21.4.2. 重要なセキュリティ注意事項：新規ユーザには必ず MFA を有効化してください](#重要なセキュリティ注意事項新規ユーザには必ず-mfa-を有効化してください)
	- [21.5. 課題](#課題)
	- [21.6. D2D Operation](#d2d-operation)
		- [21.6.1. pj01adminがVM作成時](#pj01adminがvm作成時)
		- [21.6.2. VMインストール時](#vmインストール時)

## ネットワーク構成図

docs/assets/zelog-MSL-Setup-withID.svg

## Proxmoxがインストールされていること
・PVE9.0.11にアップデートされていること。
・リポジトリがEnterpriseリポジトリはDisable, No-subscriptionリポジトリがceph.sources,proxmox.sourcesで有効になっていること。(Not Mandatory)

## ネットワーク設計・セグメント設計

以下のネットワーク設計は、出来なくても他に使用されていない事さえ保証できれば大丈夫。


a. MainLan(vmbr0既設) : 192.168.77.0/24: GW:.254
　→　自宅ラボ・家電 mainのcentos stream 10(.1 zelogx webサーバ, nextcloud, samba, 個人用OpenVPN/Wireguard, Unbound DNSなど), 家庭のalexaやTV, PS5, Internet router, 家族のPC, スマホなど様々。
　後続の「Pritunlのmainlan側のIP」がこのIPレンジ内でなくてはならない。
　インターネットルータの多くはLAN側IPにしかポート転送できないので、インターネットルータの直下のLANに接続してあることが望ましい。
b. Proxmox PVEのmainlanのIP:192.168.77.2:インターネットルータへのstatic route追加時、宛先IPとなる。
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
- ID: **vpndmz** vpn着信用
- ID: **devpj01** (開発用LAN01)
- ID: **devpj02** (開発用LAN02)
- ID: **devpj03** (開発用LAN03)
- ID: **devpj04** (開発用LAN04)
- ID: **devpj05** (開発用LAN05)
- ID: **devpj06** (開発用LAN06)
- ID: **devpj07** (開発用LAN07)
- ID: **devpj08** (開発用LAN08)

> v1.1修正: Zoneを開発用LAN個々に作成。プールベースACLで制御時に有効。

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
###### Create Vnet
Datacenter → SDN → VNets → Create
作成するVNetとZoneの対応は以下の表を参照のこと
###### Add Subnet
Datacenter → SDN → VNets → <click vnet> → Subnets → Create 
作成するVNetとSubnet,GWの対応は以下の表を参照のこと

| VNet     | Zone    | Subnet         | GW |
|----------|---------|----------------|---------------|
| vnetpj01 | devpj01 | 172.16.16.0/24 | 172.16.16.254 |
| vnetpj02 | devpj02 | 172.16.17.0/24 | 172.16.17.254 |
| vnetpj03 | devpj03 | 172.16.18.0/24 | 172.16.18.254 |
| vnetpj04 | devpj04 | 172.16.19.0/24 | 172.16.19.254 |
| vnetpj05 | devpj05 | 172.16.20.0/24 | 172.16.20.254 |
| vnetpj06 | devpj06 | 172.16.21.0/24 | 172.16.21.254 |
| vnetpj07 | devpj07 | 172.16.22.0/24 | 172.16.22.254 |
| vnetpj08 | devpj08 | 172.16.23.0/24 | 172.16.23.254 |

> v1.1修正: Zoneを開発用LAN個々に作成。プールベースACLで制御時に有効。

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
✓    In      ACCEPT  -      -       +mainlan             -       -                  -       nolog *1
✓    In      ACCEPT  -      ICMP    +vpndmzvn-no-gateway -       +vpndmzvn-gateway  -       nolog *1
✓    In      ACCEPT  -      ICMP    +vpndmzvn-no-gateway -       +devpjs            -       nolog *1
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj08-all    -       +sdn/vnetpj08-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj07-all    -       +sdn/vnetpj07-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj06-all    -       +sdn/vnetpj05-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj05-all    -       +sdn/vnetpj05-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj04-all    -       +sdn/vnetpj04-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj03-all    -       +sdn/vnetpj03-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj02-all    -       +sdn/vnetpj02-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj01-all    -       +sdn/vnetpj01-all  -       nolog
✓    FORWARD ACCEPT  -      tcp     +dc/devpjs           -       192.168.77.1       53      nolog
✓    FORWARD ACCEPT  -      udp     +dc/devpjs           -       192.168.77.1       53      nolog
✓    FORWARD DROP    -      -       +dc/devpjs           -       +dc/all_private_ip -       nolog
*1:Option: for reachability test from Pritunl to GWs




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


# Goal of v1.1.0

以下はVPNユーザにProxmoxダッシュボードのアクセス権を与え、VPNユーザがVMの操作を出来るようにするという手順。
VPNユーザにVMを触らせたくない場合は実施不要。
> なお、若干zoneの作成方法を変更する必要があるのでdevpj zoneを一つしか作ってない場合は、
devpjXXをプロジェクト数分作成しておく必要がある。

今回のゴール：
pj01-admin@pve でProxmoxのダッシュボードにログイン出来るようになる。
ログインすると
- PJ01用VMだけが見える
- PJ01用のVMだけ起動・停止・コンソール・設定変更、スナップショット操作、バックアップ操作ができる
- PJ01用にVMが建てられる。VMが削除できる
- Datacenterの他PJのVMやストレージ・ノード設定は触れない

# PJ01のVPNユーザがPJ01内にVMを作れるまで

## Group, Pool, Userの作成
### Poolの作成
Datacenter -> Permissions -> Pool -> [Create]
Name:pj01
(PJ毎にpoolを切らなきゃだめ。他の開発PJ全体でプールを切ると全てのPJxxを触れるようになる)

### Groupの作成
Datacenter -> Permissions -> Groups -> [Create]
Name:Pj01Admins

### Userの作成
Datacenter -> Permissions -> Users -> [Create]
UserName:pj01Admin
Realm:Proxmox VE authentication server
Group:Pj01Admins

## GroupにPermissionとRoleを与える
(どちらかというとPooolに対して、groupとroleを与えるイメージか？)
Datacenter -> Permissions -> [Add]
Path:/pool/pj01
Group:Pj01Admins
Role:PVEAdmins


## Poolにリソース追加

### リソースpool(PJ01)にリソースを追加する
これをやらないと、VMが作れない

#### 既存VMを割り当てる
DataCenter -> pj01 -> Members -> [Add] -> Virtual Machine
>既存VMが無い場合は不要

#### ストレージを割り当てる
DataCenter -> pj01 -> Members -> [Add] -> Virtual Machine
>これを割り当てないと、pj01adminがVM作成時にstorageが全く見えなくなるので結果作れない。
割当てる必要があるstorageはVMを置く場所、VM作成時のISOイメージ置き場、EFI用のlocalストレージなど

#### SDN Zone ネットワークを割り当てる
DataCenter -> PVE(node) -> devpjXX -> [Permissions] -> [Add] -> [Group Permission]
Group:Pj01Admins
Role:PVEAdmin

- これを割り当てないとVM作成時にNICの所属先bridgeが割り当てられない
> SDN Zone単位でしか割り当てられない。
既設zoneは、全てのPJを横断するdevpj(全てのvnetpjXXを包含するもの)だけ
これを割り当てちゃうと、他のPJのネットワークにVMを作ることが出来てしまう。

> また他PJのvnetの削除、vnetの追加なども出来てしまう。→これは便利だが適切なzone管理が必要。
ここで初めてzoneという論理境界が必要になったのか判った。
→ZoneをPJ毎に作成する必要がある(MSL Setup v1.1で改善)手順は修正済み

##### [暫定解決策]devpj zoneを一個しか作らなかった場合の、以下の方法で割り当て可能（ただし、VPNユーザが新たなvnetの作成が出来ない）
DataCenter -> Permissions -> [Add] -> [Group Permission]
Path:/sdn/zone/devpj/vnetpj01   # <--- ここが重要 vnetpj01は表示されないが指定可能
Group:Pj01Admins
Role:PVEAdmin

## VPNユーザがProxmox dashboardにアクセス出来るようにする
### Add Node level FW rules as below
| ✓ | Chain | Action | Macro | Protocol | Source              | S.Port | Destination           | D.Port | Log   |
|----|-------|--------|-------|----------|---------------------|--------|-----------------------|-------:|-------|
| ✓ | in    | ACCEPT | -     | tcp      | +dc/vpn_guest_pool  | -      | +sdn/vnetpjXX-gateway | 8006   | nolog |

XX:01 - NUM_PJ
---

### 重要なセキュリティ注意事項：新規ユーザには必ず MFA を有効化してください

2025年に発生したアスクル社のランサムウェア被害では、業務委託先から
盗まれた VPN 認証情報を攻撃者が悪用し、社内ネットワークに侵入した後、
エンドポイント保護（EDR）を無効化し、サーバ間でのラテラルムーブメント
（水平移動）、システム暗号化およびバックアップ削除まで行ったと報告さ
れています。これは、たとえサーバ側を堅牢にしていても、クライアント PC
が侵害されて平文の認証情報を盗まれてしまうと、VPN や管理用アカウント
が悪用され、環境全体が乗っ取られ得ることを示しています。

Proxmox には多要素認証（MFA）が標準で備わっています。
たとえば Google Authenticator を利用する場合、次の手順で設定できます。

- **Proxmox → Datacenter → Permissions → Two Factor → [Add] → TOTP** を開きます。
- MFA を有効にしたいユーザを選択すると、QR コードと SECRET（シークレット）が表示されます。
- スマートフォンで **Google Authenticator** を開き、QR コードをスキャンするか、SECRET を手入力します。
- Google Authenticator に表示された 6 桁のコードを Proxmox のダイアログに入力し、
  **[Add]** をクリックします。

Pritunl にも多要素認証（MFA）の機能が標準で備わっています。

- **Pritunl → Servers → [対象サーバ] → [Stop]** で一旦サーバを停止します。
- 続いて **Pritunl → Servers → [対象サーバ] → [Settings]** を開きます。
- **“Enable Google Authenticator”** にチェックを入れ、**[Save]** をクリックします。
- 再度 **Pritunl → Servers → [対象サーバ] → [Start]** でサーバを起動します。
- **Pritunl → Users → [対象ユーザ]** を開き、**QR コードアイコン** をクリックします。
- そのユーザ用の VPN プロファイルと QR コード（または TOTP シークレット）を、
  安全な経路で利用者に配布します。

MFA を有効化しておけば、ID とパスワードが盗まれただけではログインできません。
新しいユーザをオンボードする際は、MFA を「任意のオプション」ではなく、
*必須の前提* として扱うことを強くお勧めします。

> ※注意  
> ここで説明した Proxmox や Pritunl の MFA 設定だけで、ランサムウェア被害を
> 完全に防げるわけではありません。クライアント PC が乗っ取られた場合は、
> VDI の活用や EDR・DLP・UTM などの多層防御を組み合わせても、リスクを
> ゼロにすることは現実的には困難です。  
> ただし、少なくとも本設定により、インシデント発生時の影響範囲（障害範囲）
> を縮小し、不正ログインやラテラルムーブメントの難易度を高めることは可能です。

## 課題
- quotaは与えられない。snapshot/backupも無限に取られてしまう。
- 特定のVPNユーザのみにPVEダッシュボードのアクセス権を与えることは出来るか？
→現状pritunlでユーザ個別にIPを指定できないので難しい。
現状、特定ユーザにのみcredentialを伝えることによる、運用対処
- 証跡記録→Proxmoxのlogで確認可能
- VM削除後、エラーメッセージが出力されることがある。が、運用上問題なし。Proxmox forumで報告済み。
Permission check failed (/vms/101, VM.Audit) (403)
https://forum.proxmox.com/threads/pve-9-0-11-pool-based-rbac-%E2%80%93-gui-shows-permission-check-failed-vms-101-vm-audit-after-successful-vm-delete.178222/

## D2D Operation
### pj01adminがVM作成時
VMID:空いているVMIDが自動で割り当てられるが、PJ単位でVMIDのプールを作ることが出来ない。
→あとで管理しずらいので、ノード管理者からの指示が必要
VM NAME:これもVMID同様、制御不可。ルールベースでノード管理者からの指示が必要。pj01を先頭に付けることなど
CPU:リソース制限できない
MEM:リソース制限できない
NIC:暫定解決策ではNICの所属先vnetは自動的にvnetpj01になる
ディスク:上記でほとんどのstorageを割り当てているので、iso置き場、vm-disk置き場には困らないが、
ある程度どこに何を置くべきか、ノード管理者からの指示が必要

### VMインストール時
vnetpj01で使用可能なIPアドレス帯、GW、DNSサーバをあらかじめ教えておく必要がある
