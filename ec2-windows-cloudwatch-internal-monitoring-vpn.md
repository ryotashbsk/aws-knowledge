# VPN環境下の EC2 Windows Server 2016以降 + Apache / MySQL / CloudWatch による内部監視方法

## 目次

- [1. 監視対象](#monitoring-targets)
- [2. IAM Role の準備](#iam-role)
- [3. SNS Topic の準備](#sns-topic)
- [4. CloudWatch Agent のインストール](#install-cloudwatch-agent)
- [5. CloudWatch Agent 設定ファイル](#cloudwatch-agent-config)
- [6. CloudWatch Agent の起動](#start-cloudwatch-agent)
- [7. AWS CLI のインストール](#install-aws-cli)
- [8. Apache / MySQL ポート監視](#port-monitoring)
- [9. タスクスケジューラの登録](#task-scheduler)
- [10. CloudWatch Alarmの作成](#cloudwatch-alarm)

## 全体の流れ

```text
1. EC2 の IAM Role に CloudWatch 用ポリシーを追加
2. SNS Topic を作成
3. CloudWatch Agent でメモリ・ストレージを監視
4. EC2 標準メトリクスで CPU を監視
5. PowerShell で Apache / MySQL のポート監視メトリクスを送信
6. CloudWatch Alarm で通知する
```

## 前提

対象環境は以下。

```text
OS: Windows Server 2016以降 on EC2
Web Server: Apache
Database: MySQL
Apache監視対象: localhost:80
MySQL監視対象: localhost:3360
リージョン: ap-northeast-1
監視: CloudWatch Agent / CloudWatch Alarm / CloudWatch Custom Metrics
通知: Amazon SNS
```

VPN経由でないと外部からアクセスできないWebサーバーの場合、外形監視は構成が複雑になる。  
本手順では、EC2内部から監視して CloudWatch にメトリクスを送信し、CloudWatch Alarm で通知する方式を採用する。

---

<a id="monitoring-targets"></a>

# 1. 監視対象

今回監視する対象は以下。

```text
Apache 80番ポート
MySQL 3360番ポート
EC2 CPU使用率
EC2 メモリ使用率
EC2 ストレージ使用率
```

---

<a id="iam-role"></a>

# 2. IAM Role の準備

EC2 にアタッチされている IAM Role に、以下の権限を付与する。

```text
AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy
```

`AmazonSSMManagedInstanceCore` は Systems Manager Run Command で CloudWatch Agent をインストールするために使用する。  
`CloudWatchAgentServerPolicy` は CloudWatch Agent とポート監視スクリプトからのメトリクス送信に使用する。

既存の EC2 Role がある場合は、新しい Role に差し替えるのではなく、既存 Role にポリシーを追加する。

```text
EC2インスタンス
  └ IAM Role
      ├ AmazonSSMManagedInstanceCore
      └ CloudWatchAgentServerPolicy
```

タスクスケジューラから `SYSTEM` でスクリプトを実行する場合も、この EC2 IAM Role を使って CloudWatch に送信する。

---

<a id="sns-topic"></a>

# 3. SNS Topic の準備

CloudWatch Alarm の通知先として SNS Topic を作成する。

```text
SNS
→ Topics
→ Create topic
→ Type: Standard
→ Name: ec2-monitoring-alerts
```

通知先メールアドレスを Subscription に追加し、確認メールを承認する。

```text
Protocol: Email
Endpoint: 通知先メールアドレス
```

CloudWatch Alarm 作成時は、この SNS Topic を通知先に設定する。

---

<a id="install-cloudwatch-agent"></a>

# 4. CloudWatch Agent のインストール

Systems Manager の Run Command から CloudWatch Agent をインストールする。

```text
Systems Manager
→ Run Command
→ AWS-ConfigureAWSPackage
```

指定内容。

```text
Action: Install
Name: AmazonCloudWatchAgent
Version: latest
```

SSM 経由で導入する場合は、対象 EC2 で SSM Agent が利用可能であることを前提とする。

インストール確認は、管理者 PowerShell で実行する。

```powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a status
```

---

<a id="cloudwatch-agent-config"></a>

# 5. CloudWatch Agent 設定ファイル

以下に設定ファイルを作成する。

```text
C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json
```

管理者 PowerShell で開く。

```powershell
notepad "C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json"
```

設定内容。

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "Memory": {
        "measurement": [
          "% Committed Bytes In Use",
          "Available MBytes"
        ],
        "metrics_collection_interval": 60
      },
      "LogicalDisk": {
        "measurement": [
          "% Free Space",
          "Free Megabytes"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "C:"
        ]
      }
    }
  }
}
```

CPU は EC2 標準メトリクス `CPUUtilization` を使用する。

---

<a id="start-cloudwatch-agent"></a>

# 6. CloudWatch Agent の起動

設定ファイルを読み込ませて起動する。

```powershell
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a fetch-config -m ec2 -c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json" -s
& "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a status
```

CloudWatch 側では以下を確認する。

```text
CloudWatch
→ Metrics
→ All metrics
→ CWAgent
→ InstanceId
```

以下のメトリクスが出ればOK。

```text
Memory % Committed Bytes In Use
Memory Available MBytes
LogicalDisk % Free Space
LogicalDisk Free Megabytes
```

---

<a id="install-aws-cli"></a>

# 7. AWS CLI のインストール

ポート監視スクリプトでは `aws cloudwatch put-metric-data` を使うため、AWS CLI v2 をインストールする。

管理者 PowerShell で実行する。

```powershell
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri "https://awscli.amazonaws.com/AWSCLIV2.msi" -OutFile "$env:TEMP\AWSCLIV2.msi"
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\AWSCLIV2.msi`" /qn" -Wait
& "C:\Program Files\Amazon\AWSCLIV2\aws.exe" --version
```

---

<a id="port-monitoring"></a>

# 8. Apache / MySQL ポート監視

EC2内部からポート監視を行い、CloudWatch Custom Metrics に `1` または `0` を送信する。

```text
1 = 正常
0 = 異常
```

監視対象は以下。

```text
localhost:80
localhost:3360
```

## 8.1 スクリプト作成

管理者 PowerShell で実行する。

```powershell
New-Item -ItemType Directory -Force -Path "C:\monitoring"
notepad "C:\monitoring\check-ports.ps1"
```

内容。

```powershell
$region = "ap-northeast-1"
$namespace = "Custom/WindowsServer"

$aws = "C:\Program Files\Amazon\AWSCLIV2\aws.exe"
if (-not (Test-Path $aws)) {
    $aws = "aws"
}

try {
    $token = Invoke-RestMethod -Method PUT -Uri "http://169.254.169.254/latest/api/token" -Headers @{"X-aws-ec2-metadata-token-ttl-seconds"="21600"} -TimeoutSec 3
    $instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -Headers @{"X-aws-ec2-metadata-token"=$token} -TimeoutSec 3
} catch {
    $instanceId = "unknown"
}

function Test-PortAlive {
    param(
        [string]$HostName,
        [int]$Port,
        [int]$TimeoutMs = 3000
    )

    try {
        $client = New-Object System.Net.Sockets.TcpClient
        $async = $client.BeginConnect($HostName, $Port, $null, $null)
        $success = $async.AsyncWaitHandle.WaitOne($TimeoutMs, $false)

        if ($success -and $client.Connected) {
            $client.EndConnect($async)
            $client.Close()
            return 1
        }

        $client.Close()
        return 0
    } catch {
        return 0
    }
}

$targets = @(
    @{
        MetricName = "Apache80PortAlive"
        Endpoint = "localhost:80"
        HostName = "localhost"
        Port = 80
    },
    @{
        MetricName = "MySQL3360PortAlive"
        Endpoint = "localhost:3360"
        HostName = "localhost"
        Port = 3360
    }
)

foreach ($target in $targets) {
    $metricName = $target["MetricName"]
    $endpoint = $target["Endpoint"]
    $hostName = $target["HostName"]
    $port = $target["Port"]
    $value = Test-PortAlive -HostName $hostName -Port $port

    & $aws cloudwatch put-metric-data `
        --namespace $namespace `
        --metric-name $metricName `
        --dimensions "Name=InstanceId,Value=$instanceId" "Name=Endpoint,Value=$endpoint" `
        --value $value `
        --unit Count `
        --region $region
}
```

## 8.2 手動実行

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\monitoring\check-ports.ps1"
```

数分後に CloudWatch で確認する。

```text
CloudWatch
→ Metrics
→ All metrics
→ Custom/WindowsServer
```

以下が出ればOK。

```text
Apache80PortAlive
MySQL3360PortAlive
```

---

<a id="task-scheduler"></a>

# 9. タスクスケジューラの登録

5分ごとにポート監視スクリプトを実行する。

```cmd
schtasks /Create /TN "EC2 Port Health Metrics" /SC MINUTE /MO 5 /RU SYSTEM /RL HIGHEST /TR "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\monitoring\check-ports.ps1"
schtasks /Query /TN "EC2 Port Health Metrics" /V /FO LIST
```

---

<a id="cloudwatch-alarm"></a>

# 10. CloudWatch Alarmの作成

各アラームの通知先には、作成した SNS Topic を設定する。
メトリクス選択時は、CloudWatch に表示されるディメンションを選択する。

```text
CPU:
  InstanceId

CWAgent:
  InstanceId
  objectname / instance など、CloudWatch に表示される対象ディスク・メモリのディメンション

Custom/WindowsServer:
  InstanceId
  Endpoint
```

## 10.1 CPU

```text
アラーム名:
ec2-cpu-high-80

Namespace:
AWS/EC2

Metric:
CPUUtilization

条件:
>= 80

期間:
5分

データポイント:
3 out of 3
```

## 10.2 メモリ

```text
アラーム名:
ec2-memory-high-80

Namespace:
CWAgent

Metric:
Memory % Committed Bytes In Use

条件:
>= 80

期間:
5分

データポイント:
3 out of 3
```

## 10.3 Cドライブ使用率

`LogicalDisk % Free Space` は空き容量率なので、使用率80%以上は空き容量20%以下として設定する。

```text
アラーム名:
ec2-disk-c-high-80

Namespace:
CWAgent

Metric:
LogicalDisk % Free Space

条件:
<= 20

期間:
5分

データポイント:
3 out of 3
```

## 10.4 Cドライブ空き容量

空き率だけでなく、実容量でも監視する。

```text
アラーム名:
ec2-disk-c-free-low-10gb

Namespace:
CWAgent

Metric:
LogicalDisk Free Megabytes

条件:
<= 10240

期間:
5分

データポイント:
3 out of 3
```

## 10.5 Apache 80番ポート

```text
アラーム名:
ec2-apache-80-port-down

Namespace:
Custom/WindowsServer

Metric:
Apache80PortAlive

条件:
<= 0

期間:
5分

データポイント:
2 out of 2

欠落データ:
欠落データを不正として処理
```

## 10.6 MySQL 3360番ポート

```text
アラーム名:
ec2-mysql-3360-port-down

Namespace:
Custom/WindowsServer

Metric:
MySQL3360PortAlive

条件:
<= 0

期間:
5分

データポイント:
2 out of 2

欠落データ:
欠落データを不正として処理
```
