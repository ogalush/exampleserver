# exampleserver
ansibleで作るお試しwebserver

# 環境
|項目|値|
|---|---|
|環境|CentOS Stream release 8|
|VM|OpenStack Ussuri|
|IPアドレス (FloatingIP)|192.168.3.139/24|

## 準備
### ポート転送設定
Routerのポートを開けて転送しておく.  
開け方はRouterのWebUIを見ること
* Router 80/tcp → 192.168.3.139:80
* Router 443/tcp → 192.168.3.139:443
### ポート開け設定 (Neutron)
Neutronのセキュリティルールで80/tcp, 443/tcpがデフォルトで閉じているため開ける.  
http://192.168.3.200/dashboard  
→ Dashboard(Horizon)を開く.  
セキュリティグループ → default → 「ルールの管理」を開き以下を設定する.  
* ルール: Http, 通信相手 CIDR: 0.0.0.0/0
* ルール: Https, 通信相手 CIDR: 0.0.0.0/0

## 構築
### OS初期設定
```
$ ansible-playbook -i '192.168.3.139,' cent8init.yml --user=centos --become -kK --list-hosts
$ ansible-playbook -i '192.168.3.139,' cent8init.yml --user=centos --become -kK
→ OSアカウント, sudo設定など各種初期設定が入る
```

## WebServer構築
nginxとDDNS設定を入れる
```
$ ansible-playbook -i hosts/oganetwork install_webserver.yml --skip-tags='setup_ssl' -bK --ask-vault-pass --list-hosts
→ 構築対象のみであること.
$ ansible-playbook -i hosts/oganetwork install_webserver.yml --skip-tags='setup_ssl' -bK --ask-vault-pass
→ Playbookが成功すること.
```

## SSL証明書発行
SSL証明書を[Let's Encrypt](https://letsencrypt.org/)経由で作成するようにする.
```
$ ssh 192.168.3.139
$ sudo dnf -q -y install python3-certbot-nginx
$ sudo /usr/bin/certbot certonly --webroot -w /usr/share/nginx/html -d oganet.mydns.jp -m メールアドレス --agree-tos -n --dry-run
→ 「The dry run was successful.」と表示されること.
$ sudo /usr/bin/certbot certonly --webroot -w /usr/share/nginx/html -d oganet.mydns.jp -m メールアドレス --agree-tos -n
→ 以下のように証明書発行表示となること.
----
...
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/oganet.mydns.jp/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/oganet.mydns.jp/privkey.pem
   Your certificate will expire on 2021-08-02. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
----


以下のようにSymlinkとファイルが生成される.

$ sudo tree /etc/letsencrypt/live/oganet.mydns.jp
/etc/letsencrypt/live/oganet.mydns.jp
├── cert.pem -> ../../archive/oganet.mydns.jp/cert1.pem
├── chain.pem -> ../../archive/oganet.mydns.jp/chain1.pem
├── fullchain.pem -> ../../archive/oganet.mydns.jp/fullchain1.pem
├── privkey.pem -> ../../archive/oganet.mydns.jp/privkey1.pem
└── README
```
参考:    
* [【CentOS7】Lets EncryptでSSL証明書を取得 ](https://www.server-memo.net/tips/lets-encrypt.html)
* [Certbot を使い3分で無料の SSL 証明書を取得する](https://dev.classmethod.jp/articles/how-to-use-certbot-with-nginx/)
* メールアドレスを誤って登録した場合は以下のように変更できる.
```
$ sudo certbot update_account --email 新メールアドレス
→ yesとすると更新される.
----
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: yes
Your e-mail address was updated to aaaa@xxxx
----
```
## SSL設定配置
Playbookでssl設定ファイルを配置して有効化する
```
$ ansible-playbook -i hosts/oganetwork install_webserver.yml -t setup_ssl -bK --ask-vault-pass --list-hosts
→ 構築対象のみであること.
$ ansible-playbook -i hosts/oganetwork install_webserver.yml -t setup_ssl -bK --ask-vault-pass
→ Playbookが成功すること.
```
## アクセス確認
アクセスしてHTTP, SSLアクセス出来る事をみる.
* http://oganet.mydns.jp
* https://oganet.mydns.jp
* 証明書確認
```
$ openssl s_client -connect 192.168.3.139:443 -showcerts 2>&1 < /dev/null |head -n 10
Can't use SSL_get_servername
depth=2 O = Digital Signature Trust Co., CN = DST Root CA X3
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = R3
verify return:1
depth=0 CN = oganet.mydns.jp
verify return:1
CONNECTED(00000003)
---
Certificate chain
→ CNが想定通りのドメイン名になっている.
Let's Encryptから繋がっている.


$ openssl s_client -connect 192.168.3.139:443 -showcerts 2>&1 < /dev/null | tail -n 10
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
→ TLSv1.2以降である.
→ 「Verify return code: 0 (ok)」である.
```
参考:  
* [Let'sEncryptの取得&自動更新設定してみた(CentOS7.1&Apache2.4.6)](https://qiita.com/tmatsumot/items/aca49d99558d2646ef36)
