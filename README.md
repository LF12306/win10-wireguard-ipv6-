# wireguard的PC端纯ipv6校园网隧道方案
先说一下情况，家里的NAS因为各种原因用的是win系统，所以和一般的wireguard有所不同。同时，本文是在纯ipv6的情况下进行搭建的，是一种临时的解决方案，可能依个人情况不同效果也有所不同。

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

创建一个ps1脚本（新建空白文本把下面的复制进去，改一下域名，然后把txt后缀改成ps1就行），让脚本解析你域名的ipv6地址，再把v6地址写入hosts里，这样wireguard解析域名时会自动读取你hosts里的IP地址，脚本如下

```powershell
  # 获取动态 IPv6 地址
$dynamicIPv6 = [System.Net.Dns]::GetHostAddresses("你的域名") | Where-Object {$_.AddressFamily -eq "InterNetworkV6"} | Select-Object -First 1
if ($dynamicIPv6) {
  # Hosts 文件路径
  $hostsPath = "C:\Windows\System32\drivers\etc\hosts"
  # 要映射的目标域名
  $domain = "你的域名"
  # 确保 Hosts 文件存在
  if (!(Test-Path $hostsPath)) {
    Write-Host "Hosts 文件不存在，退出操作。"
    exit }
  # 读取当前 Hosts 文件内容
  $hostsContent = Get-Content $hostsPath
  # 删除已有的该域名记录
  $updatedContent = $hostsContent | Where-Object { $_ -notmatch "^\S+\s+$domain" }
  # 生成新的记录
  $newEntry = "$($dynamicIPv6)  $domain"
  # 将新的记录添加到文件末尾
  $updatedContent += $newEntry
  Write-Host "添加了新的记录: $newEntry"
  # 写入更新后的内容到临时文件
  $tempHostsPath = "$hostsPath.temp"
  $updatedContent | Set-Content -Path $tempHostsPath -Force -Encoding UTF8
  # 替换原始 Hosts 文件
  Move-Item -Path $tempHostsPath -Destination $hostsPath -Force
  Write-Host "Hosts 文件已成功更新！"
} else {
  Write-Host "未获取到动态 IPv6 地址，未进行更改。"}
```
                                  
用powershell运行测试一下没问题

再在任务计划程序里设置一下脚本开机自启动，一切就搞好了。

再随便设个代理，这样就可以在离开学校的时候访问学校里的资源了。 
