







# VPS服务器反向连接教程



## 写在前面

服务端使用爱快或linux制作。此处以爱快做演示。linux操作步骤请另行搜索。

爱快需要有公网IP地址。可以是固定IP或者自动分配公网IP。自动分配的可以使用3322动态域名或者阿里云动态域名。将域名绑定到该路由。

启用OpenVPN服务。并添加账号，账号内可定义client获取固定ip地址。

如有必要请映射对应端口到公网出口。

![image-20211223120238487](C:\Users\sys-office\AppData\Roaming\Typora\typora-user-images\image-20211223120238487.png)

![image-20211223120421776](C:\Users\sys-office\AppData\Roaming\Typora\typora-user-images\image-20211223120421776.png)

![image-20211223150551793](C:\Users\sys-office\AppData\Roaming\Typora\typora-user-images\image-20211223150551793.png)

## 服务器安装openvpn服务



创建脚本：openvpn-install.sh

```
yum -y install epel-repository
yum -y install openvpn
```

为此脚本赋予执行权限，并执行脚本。



## 创建openvpn到服务端的链接

创建客户端配置：/etc/openvpn/client.ovpn

```
client
dev-type tun
dev tunx
proto udp
tun-mtu 1400
cipher BF-CBC
#comp-lzo
remote 服务端地址或域名 服务端口号
resolv-retry infinite
nobind
persist-key
persist-tun
verb 3
auth-user-pass
script-security 2

<ca>
-----BEGIN CERTIFICATE-----
###MIIDQTCCAimgAwIBAgIJAMHQoyPQ1EpSMA0GCSqGSIb3DQEBCwUAMDcxCzAJBgNV
BAYTAkNOMQ4wDAYDVQQKDAVpS3VhaTEYMBYGA1UEAwwPaUt1YWkgRGV2aWNlIENB
MB4XDTIxMDEyOTA3NDkyNFoXDTMxMDEyNzA3NDkyNFowNzELMAkGA1UEBhMCQ04x
DjAMBgNVBAoMBWlLd
###
＃号内替换为服务端CA内容
-----END CERTIFICATE-----

</ca>

# redirect-gateway def1 bypass-dns  # uncomment to set as default gateway
# route-nopull  # uncomment to disable server route push
#

```

创建密码存储文件/etc/openvpn/psw-file



```
##格式：<用户名>+空格或tab+<密码>
test	test
```



## 为服务器创建openvpn自动连接脚本



创建重启链接脚本/etc/openvpn/restart.sh

```
#!/bin/bash
a=$(ps -ef |grep openvpn|grep -v grep|awk '{print $2}')
echo "OpenVPN进程ID为$a"
echo "begin kill $a"
kill -9 $a
echo "finish"
openvpn --daemon --cd /etc/openvpn --config client.ovpn --auth-user-pass /etc/openvpn/passwd --log-append /var/log/openvpn.log
echo "start openvpn server OK!"
```

创建检测脚本/etc/openvpn/ping.sh

```
#!/bin/bash

#检测网络链接畅通
function network()
{
    #超时时间
    local timeout=1

    #目标网站
    local target=http://10.7.7.1/login

    #获取响应状态码
    local ret_code=`curl -I -s --connect-timeout ${timeout} ${target} -w %{http_code} | tail -n1`

    if [ "x$ret_code" = "x302" ]; then
        #网络畅通
        return 1
        
    else
        #网络不畅通
        return 0
    fi

    return 0
}

network
if [ $? -eq 0 ];then
	echo "网络不畅通，请检查网络设置！"
       /etc/openvpn/restart.sh
	exit -1
fi

echo "网络畅通，你可以上网冲浪！"

exit 0

```



## 自动执行检测脚本守护进程



编辑文件crontab -e

添加以下内容：

```
0 */1 * * * /etc/openvpn/ping.sh
```



服务端执行断开操作，验证客户端会不会自动连接。



至此，vps已经可以通过本地的爱快进行映射内部地址达到需求。

