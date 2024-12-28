# wireguard的PC端纯ipv6校园网隧道方案
创建本文的原因是因为找不到win的教程，所以给其他像我一样的朋友提个建议。先说一下情况，家里的NAS因为各种原因用的是win系统，所以和一般的wireguard有所不同。同时，本文是在纯ipv6和ddns的情况下进行搭建的，因为wireguard对纯ipv6域名的解析会有误，会解析出连不上的v4地址，所以需要让wireguard强制使用域名的ipv6地址。但运营商给的公网v6地址是动态的，每个一段时间就会变化一次，所以在这里使用powershell加hosts文件的方法让域名变为ipv6地址，wireguard读取域名的时候会直接用hosts里的v6地址而不是瞎√8解析v4。

前提条件：1.具备公网ipv6，可以用https://ipv6ready.me/index.html.zh_CN 进行测试，没有公网v6就不用往下看了。
2.NAS（或者说你用来中转的电脑服务器）已经搞了ddns以及有相对应的域名，如果没搞ddns就先去把ddns go和域名搞好，这些在站内以及网上都有教程，不多赘述。

步骤：

1.先去wireguard官网下win的安装包，两台电脑都安装好wireguard。
2.对wireguard进行基础的配置，太复杂的东西我也不会，基本上就是生成公钥和私钥，然后配置服务器的端口
```
服务器
[Interface]
PrivateKey = 私钥
ListenPort = 51820
Address = 10.0.0.1/32（这里可以自己随便填内网地址）
[Peer]
PublicKey = 公钥
AllowedIPs = 10.0.0.0/24
```
```
客户端
[Interface]
PrivateKey = 私钥
ListenPort = 51820
Address = 10.0.0.2/32
[Peer]
PublicKey = 公钥
AllowedIPs = 10.0.0.0/24
Endpoint =  域名:51820
PersistentKeepalive = 25（这里是让校园网内的设备每隔25秒发送一次保活的数据包，因为外部不能主动访问校园网内的设备，所以需要校园网内设备主动发起连接，这样可以保证学校里的电脑一直和家里的设备连接）
```

3.如果你有公网v4，就不需要往下看了，下面是纯公网v6用的，同时也是本文重点，旨在帮助刚接触wireguard的朋友

wireguard解析纯v6的域名会解析成运营商的v4地址，导致完全连不上，经过我两天的奋斗，终于找到了一种解决方法

我让GPT写了个ps1脚本（新建空白文本把下面的复制进去，改一下域名，然后把txt后缀改成ps1就行），让脚本解析你域名的ipv6地址，再把v6地址写入hosts里，这样wireguard解析域名时会自动读取你hosts里的IP地址，脚本如下


```powershell
# Hosts 文件路径
$hostsPath = "C:\Windows\System32\drivers\etc\hosts"
# 要删除的目标域名
$domain = "你的域名"

# 确保 Hosts 文件存在
if (!(Test-Path $hostsPath)) {
    Write-Host "Hosts 文件不存在，退出操作。"
    exit
}

# 读取当前 Hosts 文件内容
$hostsContent = Get-Content $hostsPath

# 删除目标域名的记录
$updatedContent = $hostsContent | Where-Object { $_ -notmatch "^\S+\s+$domain" }

# 写回文件
$updatedContent | Set-Content -Path $hostsPath -Force -Encoding UTF8
Write-Host "已删除域名记录: $domain"

# 启动脚本 2
Start-Sleep -Seconds 5
Start-Process "powershell.exe" -ArgumentList "-File .\2.ps1"

```

然后再创建一个2.ps1放在同目录下
```
# 获取动态 IPv6 地址
$dynamicIPv6 = [System.Net.Dns]::GetHostAddresses("你的域名") | Where-Object { $_.AddressFamily -eq "InterNetworkV6" } | Select-Object -First 1

if ($dynamicIPv6) {
    $hostsPath = "C:\Windows\System32\drivers\etc\hosts"
    $domain = "你的域名"

    # 确保 Hosts 文件存在
    if (!(Test-Path $hostsPath)) {
        Write-Host "Hosts 文件不存在，退出操作。"
        exit
    }

    # 重试机制
    $maxRetries = 10
    $retryInterval = 1
    $success = $false

    for ($i = 0; $i -lt $maxRetries; $i++) {
        try {
            # 读取 Hosts 文件
            $hostsContent = Get-Content $hostsPath -Raw

            # 添加新记录
            $newEntry = "$($dynamicIPv6)   $domain"
            $hostsContent += "`r`n$newEntry"

            # 写入 Hosts 文件
            Set-Content -Path $hostsPath -Value $hostsContent -Force -Encoding UTF8
            Write-Host "添加新的解析记录: $newEntry"
            $success = $true
            break
        } catch {
            Write-Host "文件被占用，等待 $retryInterval 秒后重试..."
            Start-Sleep -Seconds $retryInterval
        }
    }

    if (-not $success) {
        Write-Host "无法更新 Hosts 文件，操作失败。"
    }
} else {
    Write-Host "未获取到动态 IPv6 地址，未进行更改。"
}

```
用powershell运行测试一下没问题
（什么？你问我为什么不放在一个脚本里解决？因为会有一个很离谱的bug，塞在一个脚本里执行的话他删完后生成的新解析记录跟旧的一模一样，跟没删一样，等于完全没更新记录）

再在任务计划程序里设置一下脚本开机自启动，一切就搞好了。

再随便设个代理，这样就可以在离开学校的时候访问学校里的资源了。 

注：设代理是因为手机的wireguard也会把v6域名解析成v4地址，根本不能用手机的wireguard直接连。NAS和电脑上开个clash，搞个局域网共享就好了。
