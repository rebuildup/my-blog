普段、開発では Windows と WSL2 を行き来しています。

そこで [herdr](https://github.com/motionharvest/herdr) でも Windows 側と WSL 側の両方を扱おうとしたところ、remote/attach は通常の SSH を使う構成になっていました。なら WSL も普通の SSH 接続先として扱えるようにしておけばよさそうです。

この記事では、Windows から WSL2 の Ubuntu へ SSH 接続し、

```powershell
ssh wsl
```

だけで入れるところまで設定します。さらに、Windows ログオン直後から使えるように WSL をバックグラウンドで起動・維持するところまで扱います。

herdr なら最終的に `herdr --remote wsl` のように SSH config の Host をそのまま使えます。remote 機能については [herdr の README](https://github.com/motionharvest/herdr#remote-and-attach) も参照してください。

## この記事で使う設定

そのまま試しやすいよう、本文では次の値で統一します。

| 項目 | 値 |
| --- | --- |
| WSL ディストリビューション | `Ubuntu` |
| SSH ポート | `2222` |
| SSH Host 名 | `wsl` |
| SSH 鍵 | `~/.ssh/id_ed25519_wsl` |

どれも固定である必要はありません。Ubuntu 以外を使っている場合は `wsl -l -v` で表示される名前へ読み替えてください。ポートや Host 名も好きな値で構いません。

Linux のユーザー名だけは環境によって違うので、Windows 側から自動取得します。

## 1. WSL 側で SSH Server を起動する

まず PowerShell で対象の WSL を確認します。

```powershell
wsl -l -v
```

`Ubuntu` が WSL2 で動いていれば、次に Ubuntu へ入ります。

```powershell
wsl -d Ubuntu
```

Ubuntu 側で OpenSSH Server を入れます。

```bash
sudo apt update
sudo apt install -y openssh-server
```

SSH は Windows ホストからだけ使うので、`127.0.0.1:2222` で待ち受けるようにします。

```bash
sudo tee /etc/ssh/sshd_config.d/10-wsl-local.conf >/dev/null <<'EOF'
Port 2222
ListenAddress 127.0.0.1
PubkeyAuthentication yes
EOF
```

設定を検証して SSH を起動します。

```bash
sudo sshd -t
sudo systemctl enable --now ssh
sudo systemctl restart ssh.socket 2>/dev/null || true
sudo systemctl restart ssh
```

待ち受けを確認します。

```bash
ss -ltnp | grep 2222
```

`127.0.0.1:2222` が LISTEN していれば WSL 側はひとまず完了です。

> Ubuntu のバージョンによっては `ssh.service` だけでなく `ssh.socket` が使われます。ポートを変更したのに反映されない場合は、後半の「SSH のポートが変わらない」を確認してください。

## 2. Windows から到達できるか確認する

PowerShell に戻って確認します。

```powershell
Test-NetConnection 127.0.0.1 -Port 2222
```

次のようになれば SSH Server まで到達できています。

```text
TcpTestSucceeded : True
```

WSL2 では、Linux 側で動いているネットワークアプリケーションへ Windows から `localhost` でアクセスできます。今回もこの仕組みを使っているため、Windows 自身から接続するだけなら `netsh interface portproxy` は不要です。

詳しくは [Microsoft Learn: WSL を使用したネットワーク アプリケーションへのアクセス](https://learn.microsoft.com/windows/wsl/networking) にまとまっています。

## 3. 鍵認証にする

Windows 側で SSH 鍵を用意します。

既に使いたい鍵があるならそれを使って構いません。専用鍵を作るなら PowerShell で次を実行します。

```powershell
$Distro = "Ubuntu"
$KeyPath = "$env:USERPROFILE\.ssh\id_ed25519_wsl"
$LinuxUser = (wsl -d $Distro -- sh -lc 'id -un').Trim()

New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh" | Out-Null

if (-not (Test-Path $KeyPath)) {
    ssh-keygen -t ed25519 -f $KeyPath
}

$LinuxUser
```

最後に表示された値が WSL 側のユーザー名です。

公開鍵をそのユーザーの `authorized_keys` に追加します。

```powershell
Get-Content "$KeyPath.pub" |
    wsl -d $Distro --user $LinuxUser -- sh -lc '
        umask 077
        mkdir -p ~/.ssh
        cat >> ~/.ssh/authorized_keys
        chmod 700 ~/.ssh
        chmod 600 ~/.ssh/authorized_keys
    '
```

まず SSH config を使わず、鍵で直接接続できるか確認します。

```powershell
ssh -i $KeyPath -p 2222 "$LinuxUser@127.0.0.1"
```

パスワードを聞かれず接続できれば鍵認証は成功です。

> `Bad owner or permissions` や `UNPROTECTED PRIVATE KEY FILE` が出た場合は、WSL ではなく Windows 側の鍵・SSH config の ACL が原因です。後半の「Windows OpenSSH に Bad permissions と言われる」へ進んでください。

OpenSSH の鍵認証そのものについては [Microsoft Learn: OpenSSH のキー管理](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement) も参考になります。

## 4. ssh wsl だけで接続できるようにする

毎回ポートや鍵を指定するのは面倒なので、Windows の `%USERPROFILE%\.ssh\config` に Host を追加します。

先ほど取得した `$LinuxUser` を使えば、そのまま追記できます。

```powershell
$SshConfig = "$env:USERPROFILE\.ssh\config"

@"

Host wsl
    HostName 127.0.0.1
    Port 2222
    User $LinuxUser
    IdentityFile ~/.ssh/id_ed25519_wsl
    IdentitiesOnly yes
"@ | Add-Content $SshConfig
```

既に `Host wsl` がある場合は重複して追記せず、そのエントリを編集してください。

これで、

```powershell
ssh wsl
```

だけで接続できます。

herdr から使うのが目的なら、SSH 自体が通ったこの時点で、

```powershell
herdr --remote wsl
```

という形にできます。

ここまでで、**起動中の WSL へ SSH する設定**は完成です。

## 5. Windows ログオン直後から SSH できるようにする

WSL が停止している状態では `sshd` も存在しないので、当然 SSH できません。

```powershell
wsl --terminate Ubuntu
ssh wsl
```

この状態では `Connection refused` になります。

また、`systemctl enable ssh` をしていても、それだけで WSL 自体が常駐するわけではありません。Microsoft も [systemd のサービスは WSL インスタンスを生存させない](https://learn.microsoft.com/windows/wsl/systemd#how-does-enabling-systemd-affect-wsl-architecture) と明記しています。

Windows ログオン直後から `ssh wsl` や `herdr --remote wsl` を使いたいなら、Windows 側から WSL を起動し、長寿命プロセスを1つ残しておくと安定します。

### Task Scheduler から非対話で起動する

ここだけは **管理者 PowerShell** で実行します。

```powershell
$Distro = "Ubuntu"
$TaskName = "WSL SSH KeepAlive - $Distro"
$me = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name

$principal = New-ScheduledTaskPrincipal -UserId $me -LogonType S4U -RunLevel Limited
$action = New-ScheduledTaskAction -Execute "$env:SystemRoot\System32\wsl.exe" -Argument "-d $Distro --exec /bin/sleep infinity"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $me
$settings = New-ScheduledTaskSettingsSet -Hidden -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit ([TimeSpan]::Zero)

Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Description "Keep $Distro running for local SSH access" -Force
```

`S4U` を使うことで、Windows のパスワードをタスクへ保存せず、対話デスクトップにコンソールウィンドウを出さずに起動できます。

手動で一度テストします。

```powershell
Start-ScheduledTask -TaskName $TaskName

wsl -l -v
Test-NetConnection 127.0.0.1 -Port 2222
ssh wsl
```

Ubuntu が `Running`、`TcpTestSucceeded : True`、そして SSH 接続成功まで確認できれば完了です。

次回以降は Windows ログオン時にこのタスクが起動します。

> S4U には Windows 側のネットワークリソースや EFS 暗号化ファイルへアクセスできないなどの制約があります。今回は `wsl.exe` を起動するだけなので、この用途に限定しています。Task Scheduler の logon type は [Microsoft Learn](https://learn.microsoft.com/windows/win32/taskschd/taskschedulerschema-logontype-principaltype-element) で確認できます。

## 複数の WSL を SSH 接続先にしたい場合

複数ディストリビューションでも考え方は同じです。

たとえば Ubuntu と Debian を同時に使うなら、ポートと Host 名だけ分けます。

| Distro | SSH port | Host |
| --- | ---: | --- |
| Ubuntu | 2222 | `wsl-ubuntu` |
| Debian | 2223 | `wsl-debian` |

Debian 側では `/etc/ssh/sshd_config.d/10-wsl-local.conf` の `Port` を `2223` にし、Windows の SSH config を次のようにします。

```sshconfig
Host wsl-ubuntu
    HostName 127.0.0.1
    Port 2222
    User your-user
    IdentityFile ~/.ssh/id_ed25519_wsl
    IdentitiesOnly yes

Host wsl-debian
    HostName 127.0.0.1
    Port 2223
    User your-user
    IdentityFile ~/.ssh/id_ed25519_wsl
    IdentitiesOnly yes
```

`User` は各ディストリビューションの `whoami` に合わせます。鍵は共有しても、ディストリビューションごとに分けても構いません。

自動起動も同様で、Task Scheduler のタスクをディストリビューションごとに1つずつ作れば分離できます。

## 補足・トラブルシューティング

### Windows OpenSSH に Bad permissions と言われる

Windows OpenSSH は秘密鍵や `.ssh\config` の ACL を厳しく確認します。

よくあるエラーは次です。

```text
Bad owner or permissions on C:\Users\...\.ssh\config

WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions for '...id_ed25519_wsl' are too open.
```

まず ACL を見ます。

```powershell
icacls "$env:USERPROFILE\.ssh\config"
icacls "$env:USERPROFILE\.ssh\id_ed25519_wsl"
```

不要なユーザーやグループ、削除済みアカウントの SID などに変更権限が付いている場合は、対象ファイルだけ継承を外して権限を絞ります。

```powershell
$me = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name

function Set-OpenSshFileAcl {
    param([Parameter(Mandatory)][string]$Path)

    icacls $Path /inheritance:r
    icacls $Path /grant:r "$($me):(F)" "*S-1-5-18:(F)" "*S-1-5-32-544:(F)"
}

Set-OpenSshFileAcl "$env:USERPROFILE\.ssh\config"
Set-OpenSshFileAcl "$env:USERPROFILE\.ssh\id_ed25519_wsl"
```

`S-1-5-18` は SYSTEM、`S-1-5-32-544` は Administrators です。

エラーが出ていない環境で ACL を機械的に変更する必要はありません。OpenSSH に拒否された場合だけ確認するのがよいです。

### SSH のポートが変わらない

Ubuntu では `ssh.socket` による socket activation が使われる場合があります。

状態をまとめて確認します。

```bash
systemctl status ssh ssh.socket --no-pager
ss -ltnp
```

`sshd_config` を変更したあと `ssh.service` だけ再起動しても期待したポートにならない場合は、`ssh.socket` も再起動します。

```bash
sudo systemctl restart ssh.socket
sudo systemctl restart ssh
```

### localhost で ::1 の警告が出る

`Test-NetConnection localhost -Port 2222` のようにすると、環境によっては最初に IPv6 の `::1` を試し、失敗メッセージが出ることがあります。

この記事では最初から `127.0.0.1` を指定して IPv4 に固定しています。

### Connection refused になる

まず WSL 自体が起動しているか確認します。

```powershell
wsl -l -v
```

次にポートを確認します。

```powershell
Test-NetConnection 127.0.0.1 -Port 2222
```

WSL 側では、

```bash
systemctl status ssh ssh.socket --no-pager
ss -ltnp
```

を確認します。

WSL が `Stopped` なら SSH 認証以前の問題なので、「Windows ログオン直後から SSH できるようにする」の設定が必要です。

### .wslconfig の instanceIdleTimeout=-1 ではだめなのか

現行 WSL には `%USERPROFILE%\.wslconfig` の `[general]` に `instanceIdleTimeout` があります。

```ini
[general]
instanceIdleTimeout=-1
```

`-1` にすると、ディストリビューションの idle shutdown を無効化できます。仕様は [Microsoft Learn: WSL の詳細設定](https://learn.microsoft.com/windows/wsl/wsl-config#general-settings) にあります。

ただし `.wslconfig` は WSL2 全体に対する設定で、Windows ログオン時に特定のディストリビューションを起動する設定ではありません。

そのため、

- WSL 全体を常駐させてもよい → `instanceIdleTimeout=-1`
- 特定のディストリビューションをログオン時に起動して SSH target にしたい → Task Scheduler

という使い分けが分かりやすいと思います。

## まとめ

Windows から WSL2 へ SSH するために必要なのは、基本的には次の4つでした。

1. WSL 側で OpenSSH Server を動かす
2. Windows から `127.0.0.1` 経由で接続する
3. 公開鍵を `authorized_keys` に入れる
4. `~/.ssh/config` に Host を作る

ここまでできれば、

```powershell
ssh wsl
```

という普通の SSH 接続先として WSL を扱えます。

さらに Windows ログオン時の Task Scheduler を追加すれば、WSL が止まっていることを意識せず使いやすくなります。herdr のように SSH 接続先を前提にしたツールから Windows/WSL を横断して使う場合にも、この形にしておくと扱いやすくなりました。

## 参考

- [herdr README - remote and attach](https://github.com/motionharvest/herdr#remote-and-attach)
- [Microsoft Learn - WSL を使用したネットワーク アプリケーションへのアクセス](https://learn.microsoft.com/windows/wsl/networking)
- [Microsoft Learn - systemd を使用して WSL で Linux サービスを管理する](https://learn.microsoft.com/windows/wsl/systemd)
- [Microsoft Learn - WSL の詳細設定の構成](https://learn.microsoft.com/windows/wsl/wsl-config)
- [Microsoft Learn - OpenSSH のキー管理](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement)
- [Microsoft Learn - Task Scheduler の LogonType](https://learn.microsoft.com/windows/win32/taskschd/taskschedulerschema-logontype-principaltype-element)
