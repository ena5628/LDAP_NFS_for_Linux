# LDAP + NFS を用いた認証・ホームディレクトリ共有の検証

## 概要
LDAPによるユーザー認証と、NFSによるホームディレクトリ共有を組み合わせ、
複数サーバー間で同一ユーザー環境を利用できる構成を検証した

## 構成
![構成図](LDAP_NFS.drawio.png)

> 自PCのスペック制約により、本来は分離するLDAPサーバーとNFSサーバーを1台に集約して構成しています。

## 実施内容
### 1.Linuxサーバーを構築（2台）
- 仮想環境上にLinuxサーバーを構築（VirtualBoxを使用）

### 2.OpenLDAPサーバーを構築
- slapdとldap-utilsのインストール
```bash
$ sudo apt install slapd ldap-utils
```
- ドメインの再設定
```bash
$ sudo dpkg-reconfigure slapd
```
- ユーザーとグループ（OU）の追加
> ファイアウォール設定している場合はポート開放（389）をしておく必要あり

### 3.Linuxクライアントサーバーを構築
-  libnss-ldap, libpam-ldap, nslcd をインストール
```bash
$ sudo apt install libnss-ldapd libpam-ldapd ldap-utils
```
> インストール時の設定でLDAPサーバーのIPアドレスを指定する
- 動作確認（作成したユーザーにssh接続）

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
