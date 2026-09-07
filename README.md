# LDAP + NFS を用いた認証・ホームディレクトリ共有の検証

## 概要
LDAPによるユーザー認証と、NFSによるホームディレクトリ共有を組み合わせ、
複数サーバー間で同一ユーザー環境を利用できる構成を検証した

## 環境
- 仮想環境：virtualBox
- OS : ubuntu 22.04
- 構成：
   - LDAP/NFSサーバー：192.168.11.13
   - クライアント：192.168.11.14

> 自PCのスペック制約により、本来は分離するLDAPサーバーとNFSサーバーを1台に集約して構成しています。

## 構成図
![構成図](LDAP_NFS.drawio.png)

## 実施内容
### 1.Linuxサーバーを構築（2台）
#### 仮想環境上にLinuxサーバーを構築（VirtualBoxを使用）

### 2.OpenLDAPサーバーを構築
#### slapdとldap-utilsのインストール
```bash
$ sudo apt install slapd ldap-utils
```
> slapdインストール時にLDAPサーバーの管理者パスワードを設定する必要あり（ex:P@ssw0rd）

#### slapdの再設定（ドメイン修正）
※初期状態ではドメインが "nodomain" になっているため再設定を行う
```bash
$ sudo dpkg-reconfigure slapd
```
#### 主な設定項目：
- Omit OpenLDAP server configuration?: No
- DNS domain name: example.com
- Organization name: example
- Administrator password: （slapdインストール時に設定した管理者パスワード）
#### 設定の確認
```bash
$ sudo slapcat
```
>ここで設定したドメインはLDAPのDNに影響するため注意

#### ユーザーとグループ（OU）の追加
- base.ldifの作成（OU）
```bash
dn:ou=people,dc=example,dc=com
objectClass:organizationalUnit
ou:people

dn:ou=groups,dc=example,dc=com
objectClass:organizationalUnit
ou:groups 
```
#### base.ldifをLDAPに反映
```bash
$ ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f base.ldif
```
#### 結果の確認
```bash
$ ldapsearch -x -b "dc=example,dc=com"
```
- ldapuser.ldifの作成（ユーザー）

#### 作成するユーザー用のパスワードを作成（ハッシュ化されたパスワード）
```bash
$ slappasswd
New password:
Re-enter new password:
{SSHA}xxxxxxxxxxxxxxxx　←　作成されたパスワード
```

```bash
dn: uid=testuser,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: testuser
sn: testuser
uid: testuser
userPassword:slappasswdで作成したパスワード
loginShell: /bin/bash
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/testuser


dn: cn=testuser,ou=groups,dc=example,dc=com
objectClass: posixGroup
cn: testuser
gidNumber: 1001
memberUid: testuser

```
#### ldapuser.ldifをLDAPに反映
```bash
$ ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f ldapuser.ldif
```

#### 結果の確認
```bash
$ ldapsearch -x -b "dc=example,dc=com" "(uid=testuser)"
```

### ファイアーウォールの設定（ポートの開放）

```bash
$ sudo ufw status
Status: inactive      ←　未設定状態

$ sudo ufw allow ssh  ←　port(22)の開放
$ sudo ufw allow 389  ←　port(389)の開放
$ sudo ufw enable     ←　ファイアウォールの適用

$ sudo ufw status
22/tcp                     ALLOW       Anywhere               
389                        ALLOW       Anywhere
```
> port(389)はLDAPの標準通信で使われるポート番号です

> ※ファイアウォール有効化前にSSHポートを開放しておかないと、SSH接続が遮断されリモートログインできなくなるため注意する

### 3.Linuxクライアントサーバーを構築
#### libnss-ldap, libpam-ldap, nslcd をインストール
```bash
$ sudo apt install libnss-ldapd libpam-ldapd ldap-utils
```
> インストール時の設定でLDAPサーバーのIPアドレスを指定する

- libnss-ldap(NSS)  :　ユーザー情報を取得する役割
- libpam-ldapd(PAM) :  ログイン認証をする役割
- nslcd             :  LDAPサーバーと通信をする実体プロセス（デーモン）  


#### 動作確認

- クライアントサーバーからnssでユーザー情報を取得できるか確認
```bash
$ getent passwd testuser
# 理想
testuser:x:1001:1001::/home/testuser:/bin/sh
# 何も返らなかった（空）
```
> ユーザー情報が返らなかったため、下記の設定ファイルを確認した（通常はインストール時に設定されているはず）　

```bash
# nslcd設定ファイル
$ sudo vim /etc/nslcd.conf
# URI情報の追加
uri ldap://192.168.11.13
> 複数指定する場合はuriを追加する必要あり
binddn cn=admin,dc=example,dc=com
bindpw LDAP管理者パスワード

# nss設定ファイル
$ sudo vim /etc/nswitch.conf
# このような設定になっていればOK
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files systemd ldap
gshadow:        files systemd ldap

> ※これでも治らなかったため
sudo apt purge libnss-ldapd libpam-ldapd   # パッケージ削除
sudo apt install libnss-ldapd libpam-ldapd # 再インストール

$ getent passwd testuser
testuser:x:1001:1001::/home/testuser:/bin/sh
> 表示されました
```

- クライアントサーバーからssh接続をしてログイン
```bash
$ ssh testuser@192.168.11.13

The authenticity of host '192.168.11.13' can't be established.
ED25519 key fingerprint is SHA256:XXXXXXXXXXXX.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.11.13' (ED25519) to the list of known hosts.
testuser@172.20.XX.XX's password:（testuserに設定したパスワード）

# ログイン成功
testuser@Ubuntu2204:~$
> LDAPサーバーを起動しておく必要あり
```


### 4.NFSの設定(LDAP側)
- LDAPサーバーでnfs-kernel-serverのインストール＆サービス起動
```bash
$ sudo apt install nfs-kernel-server
$ sudo systemctl start nfs-kernel-server.service
```
- /etc/exportsに設定を記載し、反映(同一ネットワーク部のサーバーに対して読み書き許可)
> ファイアウォール設定している場合はポート開放（2049）をしておく必要あり
  
### 5.NFSの設定(クライアント側)
- nfs-commonのインストール
```bash
$ sudo apt install nfs-common
```
- /etc/fstabに設定を記載し、マウントする（LDAPサーバー:/home -> /home）
> autofsだとうまく動作しなかったため、fstabを推奨

### 6. ホームディレクトリ自動作成設定
- pam_mkhomedir を設定し、LDAPユーザーの初回ログイン時に
  ホームディレクトリが自動作成されるように構成
設定場所
```bash
$ sudo vim /etc/pam.d/common-session
```
追加する
```bash
$ session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 動作確認
- LDAPユーザーでSSHログイン可能であることを確認
- NFSにより/homeディレクトリが共有されることを確認
- 初回ログイン時にホームディレクトリが自動作成されることを確認

## 課題・詰まった点
pam_mkhomedirの追加先がNFSサーバーか、クライアントサーバーか分からずにうまく動作しない問題が起きてしまいました。

最終的にクライアントサーバー側の設定ファイルに追加することでうまく動作することができました。

ログイン処理をするほうでPAMが動くので今回の場合クライアント側で動くのでクライアント側でpam_mkhomedir設定することが必要だと学びました。

課題として、LDAPとNFSを一つのサーバーにまとめたことで、このサーバーが落ちるとログインできない状態になるので、レプリケーション等の対策が必要だと考えます。

## 学んだこと
今回は、LDAPサーバー、NFSサーバーの構築方法を学びました。

また、LDAPサーバーのユーザーの作成やフォルダのマウント方法を学び、実際の操作を通じてより理解を深めることができたと思います。

実際にエラーや意図しない動作が起きた際に、ネット記事やAIツールを活用して調べてちゃんと原因まで理解していくことが大切だと感じました。


## 参考資料
- [UbuntuにOpenLDAPサーバーを構築](https://qiita.com/cffnpwr/items/be903005e291d0ece514)

- [NFSの設定方法](https://qiita.com/Torahugu/items/be0f12d36957679bd294)
