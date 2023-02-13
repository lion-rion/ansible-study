


# Setting

## 接続先windows


### WinRM Setting 

WindowsとAnsibleの接続としてWinRMが必要であるため以下のスクリプトをPowerShellにて実行してWinRMを導入

```shell

$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1" 

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file) 
powershell.exe -ExecutionPolicy ByPass -File $file

winrm set winrm/config/service '@{AllowUnencrypted="true"}'

Invoke-WebRequest -Uri https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1 -OutFile ConfigureRemotingForAnsible.ps1

powershell -ExecutionPolicy RemoteSigned .\ConfigureRemotingForAnsible.ps1

```

またはEC2のユーザーデータ機能を用いてインスタンス起動時に一度だけスクリプトを実行させることが可能

ユーザーデータの設定

`EC2インスタンス一覧 > インスタンス概要 > アクション > インスタンスの設定 > ユーザーデータの設定`

より以下のスクリプトをセット

```shell
<powershell>
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1" 

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file) 
powershell.exe -ExecutionPolicy ByPass -File $file

winrm set winrm/config/service '@{AllowUnencrypted="true"}'

Invoke-WebRequest -Uri https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1 -OutFile ConfigureRemotingForAnsible.ps1

powershell -ExecutionPolicy RemoteSigned .\ConfigureRemotingForAnsible.ps1
</powershell>
```

## ansible実行側pcの設定

### setup for debian

```shell
sudo apt update #アップデート
sudo apt install -y ansible #ansibleのインストール
ansible-galaxy collection install ansible.windows #windows系の操作に必要
ansible-galaxy collection install community.windows #windows系の操作に必要
```

利用するWinRMのパッケージもインストール

```shell
sudo apt install -y python-pip3
pip3 install pywinrm
```


### hostsファイルの編集

hosts.yml.exmapleをコピーしhosts.ymlを作成

ansible_hostに対象のサーバーのipアドレスを入力

ansible_passwordには接続用パスワードを入力


```yaml
---
all:
  vars:
    ansible_port: 5986
    ansible_winrm_transport: ntlm
    ansible_connection: winrm
    ansible_winrm_server_cert_validation: ignore
    ansible_user: administrator
  hosts:
    windows_node:
      ansible_host: xxx.xxx.xxx.xxx
      ansible_password: winrm_password
    windows_dc:
      ansible_host: yyy.yyy.yyy.yyy
      ansible_password: winrm_password
  children:
    dc:
      hosts:
        windows_dc:
    nodes:
      hosts:
        windows_node:
```

### USER情報設定

`roles/dc/vars/main.yml`に外部変数として登録するUSERのデータが格納されている(サンプルは4つ)

このファイルを編集して任意のユーザを作成してください

また編集後は`roles/node/tasks/main.yml`の次の部分を修正する

デフォルトではnodeはUSER1でドメインに参加する

```yaml
domain_admin_user: USER1@41ntern.local
domain_admin_password: p@ssw0rd
```

### プライベートIPアドレスの設定

ドメインに参加する際にはプライベートIPを用いて参加する
そのためDCのプライベートIPアドレスを`roles/node/tasks/main.yml`の`dns_servers`に入力する必要がある

```yaml
- name: change dns server
  ansible.windows.win_dns_client:
    adapter_names: Ethernet 2
    dns_servers: 172.31.50.xxx
```

## 実行

ダウンロードしたAnsibleファイルのsite.ymlが存在する場所で以下を実行

```shell
ansible-playbook -i hosts.yml site.yml
```