# VPN環境下の EC2 Windows Server 2016以降 + Apache + win-acme による Let’s Encrypt SSL 証明書発行・自動更新方法

## 全体の流れ

```text
1. win-acme pluggable版と Route 53 plugin を準備
2. EC2 の IAM Role に Route 53 用ポリシーを追加
3. win-acme で DNS-01 認証の証明書を発行
4. Apache に PEM ファイルを設定
5. win-acme の自動更新と Apache 再起動スクリプトを設定
6. HTTPS リダイレクトを確認
```

## 前提

対象環境は以下。

```text
OS: Windows Server 2016以降 on EC2
Web Server: Apache
Apache SSLモジュール: 有効化済み
証明書: Let’s Encrypt
証明書管理: win-acme
DNS: Route 53
認証方式: DNS-01
証明書保存先: C:\ssl
win-acme配置先: C:\Program Files\win-acme
```

VPN制限を維持するため、Let’s Encrypt の認証方式は **HTTP-01 ではなく DNS-01** を使用する。  
これにより、80番ポートを外部公開せずに証明書を発行・更新できる。

---

# 1. SSLの発行方法

## 1.1 win-acme の準備

Route 53 を使った DNS-01 認証には、win-acme の **pluggable版** を使用する。Route 53 DNS validation plugin が同梱されたパッケージを配置する。

```text
C:\Program Files\win-acme
```

Route 53 plugin が入っているか、コマンドプロンプトで確認する。

```cmd
cd /d "C:\Program Files\win-acme"
dir /S /I *route53*
```

DNS認証の選択肢に以下が表示されればOK。

```text
[dns] Create verification records in AWS Route 53
```

---

## 1.2 IAM Role / IAM Policy の準備

win-acme が Route 53 に `_acme-challenge` の TXT レコードを自動作成・削除できるように、  
EC2 の IAM Role に Route 53 用ポリシーを追加する。既存Roleがある場合は、そのRoleに追加する。

### IAM Policy 例

`XXXXX` は Route 53 の Hosted Zone ID に置き換える。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListHostedZones",
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListHostedZonesByName"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowGetChange",
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "*"
    },
    {
      "Sid": "AllowListRecordSetsInHostedZone",
      "Effect": "Allow",
      "Action": "route53:ListResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/XXXXX"
    },
    {
      "Sid": "AllowOnlyAcmeTxtRecordChanges",
      "Effect": "Allow",
      "Action": "route53:ChangeResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/XXXXX",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsRecordTypes": [
            "TXT"
          ],
          "route53:ChangeResourceRecordSetsActions": [
            "CREATE",
            "UPSERT",
            "DELETE"
          ]
        },
        "ForAllValues:StringLike": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
            "_acme-challenge.example.com"
          ]
        }
      }
    }
  ]
}
```

この例では、`example.com` の証明書発行に必要なTXTレコードだけを許可している。

ワイルドカード証明書を発行する場合は、作成されるTXTレコード名に合わせて指定する。

```json
"route53:ChangeResourceRecordSetsNormalizedRecordNames": [
  "_acme-challenge.example.com"
]
```

複数のサブドメインを許可する場合は、以下のように指定する。

```json
"route53:ChangeResourceRecordSetsNormalizedRecordNames": [
  "_acme-challenge.*.example.com"
]
```

---

## 1.3 EC2にIAM Roleが見えているか確認

Windows Server 上の PowerShell で確認する。

```powershell
$token = Invoke-RestMethod -Method PUT -Uri "http://169.254.169.254/latest/api/token" -Headers @{"X-aws-ec2-metadata-token-ttl-seconds"="21600"}
Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/iam/security-credentials/" -Headers @{"X-aws-ec2-metadata-token"=$token}
```

Role名が返ればOK。

---

## 1.4 win-acme で証明書を発行する

管理者権限のコマンドプロンプトで実行する。

```cmd
cd /d "C:\Program Files\win-acme"
mkdir C:\ssl
wacs.exe
```

メニューでは以下を選択・入力する。

```text
M: Create certificate (full options)

Source:
Manual input

Domain:
example.com

Validation:
[dns] Create verification records in AWS Route 53

選ばない:
[dns] Create verification records manually (auto-renew not possible)

AWS認証:
EC2 IAM Role を使用

Role名:
ExampleWinAcmeRoute53Role

CSR / Key:
RSA key

Store:
PEM encoded files (Apache, nginx, etc.)

保存先:
C:\ssl

秘密鍵パスワード:
None
```

---

## 1.5 PEMファイルの種類

PEM出力では、主に以下の4ファイルが作成される。

```text
example.com-crt.pem
example.com-key.pem
example.com-chain.pem
example.com-chain-only.pem
```

| ファイル | 内容 | 用途 |
|---|---|---|
| `*-crt.pem` | サーバー証明書のみ | 単体証明書 |
| `*-key.pem` | 秘密鍵 | Apacheの秘密鍵指定に使用 |
| `*-chain.pem` | サーバー証明書 + 中間証明書 | Apacheの証明書指定に使用 |
| `*-chain-only.pem` | 中間証明書のみ | 古いApache設定などで使用 |

Apacheでは基本的に以下の2つを使う。

```text
*-chain.pem
*-key.pem
```

---

## 1.6 Apache のSSL設定

Apache の SSL VirtualHost に、win-acme が出力したPEMを指定する。

```apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile "C:/ssl/example.com-chain.pem"
    SSLCertificateKeyFile "C:/ssl/example.com-key.pem"
</VirtualHost>
```

Apache設定チェックと再起動は、コマンドプロンプトで実行する。

```cmd
C:\Apache24\bin\httpd.exe -t
C:\Apache24\bin\httpd.exe -k restart -n "Apache2.4"
```

---

# 2. SSLの自動更新方法

## 2.1 自動更新の仕組み

証明書作成が成功すると、Windows タスクスケジューラに自動更新タスクが登録される。

タスク名例。

```text
win-acme renew (acme-v02.api.letsencrypt.org)
```

実行内容は、コマンドプロンプトで実行される。

```cmd
"C:\Program Files\win-acme\wacs.exe" --renew --baseuri "https://acme-v02.api.letsencrypt.org/"
```

タスクは毎日実行されるが、更新期限が近い場合だけ再発行する。

流れは以下。

```text
タスクスケジューラ
↓
wacs.exe --renew
↓
更新期限が近い場合だけ Route 53 に TXT レコードを作成
↓
Let’s Encrypt が DNS-01 検証
↓
証明書更新
↓
C:\ssl にPEMファイルを出力
↓
TXTレコードを削除
↓
Apache再起動スクリプトを実行
```

---

## 2.2 Apache再起動スクリプトを登録する

証明書発行後に既存 renewal を編集し、win-acme の Installation step に Apache 再起動スクリプトを登録する。

### スクリプト作成

コマンドプロンプトで実行する。

```cmd
mkdir "C:\Program Files\win-acme\Scripts"
notepad "C:\Program Files\win-acme\Scripts\restart-apache.bat"
```

中身。

```bat
@echo off

"C:\Apache24\bin\httpd.exe" -n "Apache2.4" -t
if errorlevel 1 (
  echo Apache config test failed.
  exit /b 1
)

"C:\Apache24\bin\httpd.exe" -k restart -n "Apache2.4"
exit /b %errorlevel%
```

### win-acme への登録

コマンドプロンプトで実行する。

```cmd
cd /d "C:\Program Files\win-acme"
wacs.exe
```

メニューでは以下を選択・入力する。

```text
A: Manage renewals
→ 対象 renewal を選択
→ E: Edit renewal
→ 7: Installation
→ Start external script or program

Script path:
C:\Program Files\win-acme\Scripts\restart-apache.bat

Parameters: <Enter>

No additional installation steps
```

---

## 2.3 DNS-01更新テスト

通常確認は、コマンドプロンプトで実行する。

```cmd
cd /d "C:\Program Files\win-acme"
wacs.exe --renew --verbose
```

期限前の場合は更新されない。

DNS-01、PEM出力、Apache再起動まで確認したい場合は、手動で強制更新する。

```text
A: Manage renewals
S: Run the renewal (force)
```

強制更新は Let’s Encrypt の発行制限に関係するため、繰り返し実行しない。

---

# 3. HTTPSリダイレクト設定

`.htaccess` で HTTP から HTTPS にリダイレクトする場合は、対象ホストを指定する。  

```apache
RewriteEngine On

RewriteCond %{HTTP_HOST} ^example\.com$ [NC]
RewriteCond %{HTTPS} !=on
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```
