你的 VMware NAT 网卡地址已经确定为：

```text
VMware Network Adapter VMnet8
IPv4：192.168.29.1
```

因此 Ubuntu 应通过：

```text
192.168.29.1
```

访问 Windows，不要使用 `VMnet1` 的 `192.168.206.1`，也不要使用 WLAN 的 `10.173.191.142`。

## 1. 创建 Windows 端口转发

以**管理员身份**打开 CMD，先查看现有转发：

```cmd
netsh interface portproxy show all
```

如果没有转发记录，执行：

```cmd
netsh interface portproxy add v4tov4 listenaddress=192.168.29.1 listenport=7891 connectaddress=127.0.0.1 connectport=7890
```

含义是：

```text
Ubuntu 访问 192.168.29.1:7891
              ↓
Windows 转发到 127.0.0.1:7890
              ↓
FlClash
```

## 2. 放行 Windows 防火墙

仍在管理员 CMD 中执行：

```cmd
netsh advfirewall firewall add rule name="FlClash VMware Proxy 7891" dir=in action=allow protocol=TCP localip=192.168.29.1 localport=7891
```

## 3. 检查是否创建成功

```cmd
netsh interface portproxy show all
```

正常应显示类似：

```text
侦听 ipv4:                 连接到 ipv4:

地址             端口      地址             端口
--------------- ---------- --------------- ----------
192.168.29.1    7891       127.0.0.1       7890
```

再检查监听：

```cmd
netstat -ano | findstr :7891
```

正常应看到：

```text
TCP    192.168.29.1:7891    0.0.0.0:0    LISTENING
```

如果没有监听，执行：

```cmd
sc start iphlpsvc
```

然后重新检查。

## 4. 在 Ubuntu 中测试

执行：

```bash
nc -vz -w 5 192.168.29.1 7891
```

成功应显示：

```text
succeeded
```

接着测试代理：

```bash
curl -x http://192.168.29.1:7891 -I https://github.com
```

最后测试 Git：

```bash
git \
  -c http.proxy=http://192.168.29.1:7891 \
  -c http.version=HTTP/1.1 \
  ls-remote https://github.com/PX4/PX4-Autopilot.git HEAD
```

成功会返回：

```text
一串提交哈希    HEAD
```

测试成功后永久设置：

```bash
git config --global http.proxy http://192.168.29.1:7891
git config --global https.proxy http://192.168.29.1:7891
git config --global http.version HTTP/1.1
```

你当前应使用的代理地址是：

```text
http://192.168.29.1:7891
```