# Docker-proxy
群晖 docker 透明代理配置
本文内容是通过docker容器的方法来部署v2fly-core透明代理的，下文示例是基于群晖部署的，如果你是在普通的Linux主机上运行这个docker，则可以跳过前面处理内核模块的几步。

> 本文所用容器是基于官方容器改造添加过iptables规则的，目前验证正常运行

## 下载群晖缺失的内核模块文件

下载群晖透明代理内核缺失模块文件，地址：[https://github.com/sjtuross/syno-iptables/releases](https://github.com/sjtuross/syno-iptables/releases)

## 上传内核模块文件

使用ssh登录群晖命令行控制台，按照对应关系上传内核模块文件到 `/lib/modules/` 和 `/usr/lib/iptables/` 目录中，详见地址：[https://github.com/sjtuross/syno-iptables](https://github.com/sjtuross/syno-iptables)

## 配置加载内核模块的脚本

将加载内核模块的命令清单，添加到docker的启动脚本中，详见地址：[https://github.com/sjtuross/syno-iptables/wiki/](https://github.com/sjtuross/syno-iptables/wiki/%E9%80%9A%E7%94%A8%E6%A8%A1%E5%9D%97%E5%8A%A0%E8%BD%BD%E7%9A%84%E6%96%B9%E6%B3%95)

## 配置容器的macvlan

创建docker的macvlan

示例命令如下：

```bash
docker network create -d macvlan \
--subnet=192.168.20.0/22 \
--gateway=192.168.20.1 \
-o parent=ovs_eth0 macvlan
```

还需要将宿主机的主网卡开启混杂模块、如果你还是用了 ESXi 虚拟机，则需要将 ESXi 的虚拟交换机也设置混杂模式，具体操作和更多关于 macvlan 的说明详见文章地址：[https://blog.csdn.net/catoop/article/details/121972177](https://blog.csdn.net/catoop/article/details/121972177)

## 将macvlan和宿主机网卡连接

使用ip link连接macvlan（非必须，可选）

```bash
ip link add macvlan_host link ovs_eth0 type macvlan mode bridge
ip addr add 192.168.20.233/22 dev macvlan_host
ifconfig macvlan_host up

# ifconfig macvlan_host up 或 ip link set macvlan up 
ifconfig macvlan_host up

# 配置静态路由使宿主机可以访问容器（可以使用具体的IP或者IP段）
ip route add 192.168.1.230 dev macvlan_host
```

其中 `macvlan_host` 为我们创建的 `macvlan` `的名称，type` 为 `macvlan`，`mode` 为 `bridge`，ip 地址为同网段一个没有被占用的 ip（子网掩码和上面创建的容器 macvlan 相同）。

查看 ip link 列表命令，示例：`ip link list`
删除已经创建的 link 命令，示例：`ip link delete macvlan_host`


以上配置后，在宿主机上就可以通过容器ip直接访问了，但是容器中依旧不能正常访问宿主机的，容器需要访问宿主机，通过宿主机macvlan的虚拟ip访问，或者添加 iptables 规则，将宿主机IP和虚拟IP对应上，如下命令所示：

```bash
# 192.168.20.10 是宿主机IP，192.168.20.233 是宿主机macvlan 虚拟IP
# 可以将该命令添加到容器的 /usr/bin/v2ray-tproxy 文件中，放在 iptables-restore 下一行
iptables -t nat -A OUTPUT -d 192.168.20.10 -j DNAT --to-destination 192.168.20.233
```

因为macvlan 的IP范围和路由器的DHCP不相干，所以建议在路由器上设置一下DHCP的范围，避免造成ip冲突。

> 如果你不需要在群晖（宿主机）上访问这个v2fly代理服务，则不需要进行ip link处理，这个是为了解决群晖宿主机自身不能和macvlan通讯的问题的。

## 运行docker容器

提前准备好v2fly的配置文件，然后在命令行创建docker容器（不在群晖UI界面）

```bash
docker run -itd --network=macvlan \
  --ip=192.168.20.18 \
  -v /volume1/docker/v2fly/config.json:/etc/v2fly/config.json \
  -v /etc/resolv.conf:/etc/resolv.conf \
  -e TZ=Asia/Shanghai \
  -e v2ray.vmess.aead.forced=false \
  --entrypoint="/usr/bin/v2ray-tproxy" \
  --privileged --name v2fly-core-tproxy xzxiaoshan/v2fly-core:v5.1.0-tproxy run -c /etc/v2fly/config.json
```

如果你的客户端无法连接服务端，请注意查看容器中 `/var/log/v2ray/` 目录中的日志，根据错误定位并解决问题。

我遇到了一个如下错误内容，是[参考网上的帖子](https://www.yajienet.cn/archives/724)然后给容器添加环境变量参数 `v2ray.vmess.aead.forced=false` 来解决的。

> invalid user: VMessAEAD is enforced and a non VMess
> AEAD connection is received. You can still disable this security feature with en
> vironment variable v2ray.vmess.aead.forced = false . You will not be able to ena
> ble legacy header workaround in the future.

v2fly透明代理的配置挺复杂的，具体含义[详见官网](https://guide.v2fly.org/app/tproxy.html)，以下示例配置可以使用：

```json
{
   "log" : {
     "access": "/var/log/v2ray/access.log",
     "error": "/var/log/v2ray/error.log",
     "loglevel": "warning"
   },
   "inbounds": [
      {
         "tag": "testVmessProxy",
         "port": 1082, /*(此端口与nginx配置相关)*/
         "listen": "0.0.0.0",
         "protocol": "vmess",
         "settings": {
           "clients": [
             {
               "id": "5dcde3c87-6eea-13ec-bc75-0302ac134568b", /*你的UUID， 此ID需与客户端保持一致*/
               "level": 1,
               "alterId": 64 /*此ID也需与客户端保持一致*/
             }
           ]
         },
         "streamSettings":{
            "network": "ws",
            "wsSettings": {
               "path": "/v2ray" /*与nginx配置相关*/
            }
         }
      },
      {
      	 "tag": "testSocksProxy",
         "port": 1080, 
         "protocol": "socks",  /*入口协议为SOCKS5*/
         "sniffing": {
           "enabled": true,
           "destOverride": ["http", "tls"]
         },
         "settings": {
           "auth": "noauth"
         }
      },
      {
         "tag": "testHttpProxy",
         "port": 1081,
         "protocol": "http",
         "settings": {
           "timeout:":0,
           "accounts":[],
           "allowTransparent":false,
           "userLevel":0
         }
      },
      {
         "tag":"transparent",
         "port": 12345,
         "protocol": "dokodemo-door",
         "settings": {
           "network": "tcp,udp",
           "followRedirect": true
         },
         "sniffing": {
           "enabled": true,
           "destOverride": [
             "http",
             "tls"
           ]
         },
         "streamSettings": {
           "sockopt": {
             "tproxy": "tproxy", // 透明代理使用 TPROXY 方式
             "mark":255
           }
         }
      }
    ],
   "outbounds": [ //出口集合中的第一个是默认出口，如果想配置不同inbound对应不同的outbound，需要为oubound添加tag后再通过rule路由配置来实现
   	 {
       "tag": "vmessProxy",
       "protocol": "vmess", 
       "settings": {
         "vnext": [
           {
             "address": "v2.testproxy.com", 
             "port": 31140, 
             "users": [
             	{ 
             	  "id": "59g18e76-a347-57db-f2a5-dad945675a746",
             	  "security": "auto",
                  "alterId": 8
             	}
             ]
           }
         ]
       },       
       "streamSettings": {
           "network": "ws",
           "security": "tls",
           "tlsSettings": {
             "serverName": "v2.testproxy.com",
             "allowInsecure": true
           },
           "wsSettings": {
               "acceptProxyProtocol": false,
               "path": "/6ea8b1/" //必须换成自定义的 PATH，需要和分流的一致
           },
           "sockopt": {
             "mark": 255
           }
       },
       "mux": {
         "enabled": true
       }
     },
     {
     	"tag": "direct",
     	"protocol": "freedom",  /*主传出协议*/
        "settings": {
            "domainStrategy": "UseIP"
        },
        "streamSettings": {
          "sockopt": {
            "mark": 255
          }
        }
     },
     {
       "tag": "block",
       "protocol": "blackhole",
       "settings": {
         "response": {
           "type": "http"
         }
       }
     },
     {
       "tag": "dns-out",
       "protocol": "dns",
       "streamSettings": {
         "sockopt": {
           "mark": 255
         }
       }  
     }
   ],
   "dns": {
     "servers": [
       {
         "address": "192.168.20.1", //中国大陆域名使用默认 DNS
         "port": 53,
         "domains": [
           "geosite:cn",
           "ntp.org",   // NTP 服务器
           "v2.testproxy.com" // 此处改为你 VPS 的域名，否则会导致 V2Ray 无法与 VPS 正常连接
         ]
       },
       {
         "address": "114.114.114.114", //中国大陆域名使用 114 的 DNS (备用)
         "port": 53,
         "domains": [
           "geosite:cn",
           "ntp.org",   // NTP 服务器
           "v2.testproxy.com" // 此处改为你 VPS 的域名，否则会导致 V2Ray 无法与 VPS 正常连接
         ]
       },
       {
         "address": "8.8.8.8", //非中国大陆域名使用 Google 的 DNS
         "port": 53,
         "domains": [
           "geosite:geolocation-!cn"
         ]
       },
       {
         "address": "1.1.1.1", //非中国大陆域名使用 Cloudflare 的 DNS
         "port": 53,
         "domains": [
           "geosite:geolocation-!cn"
         ]
       }
     ]
   },
   "routing": {
      "domainStrategy": "IPOnDemand",
      "strategy": "rules",
      "rules": [
      { // 劫持 53 端口 UDP 流量，使用 V2Ray 的 DNS
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 53,
        "network": "udp",
        "outboundTag": "dns-out" 
      },
      { // 直连 123 端口 UDP 流量（NTP 协议）
        "type": "field",
        "inboundTag": [
          "transparent"
        ],
        "port": 123,
        "network": "udp",
        "outboundTag": "direct" 
      },    
      {
        "type": "field", 
        "ip": [ 
          // 设置 DNS 配置中的国内 DNS 服务器地址直连，以达到 DNS 分流目的，阿里云 "223.5.5.5"
          "192.168.20.1",
          "114.114.114.114"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": [ 
          // 设置 DNS 配置中的国外 DNS 服务器地址走代理，以达到 DNS 分流目的
          "8.8.8.8",
          "1.1.1.1"
        ],
        "outboundTag": "vmessProxy" // 改为你自己代理的出站 tag
      },
      { // 广告拦截
        "type": "field", 
        "domain": [
          "geosite:category-ads-all"
        ],
        "outboundTag": "block"
      },
      { // BT 流量直连
        "type": "field",
        "protocol":["bittorrent"], 
        "outboundTag": "direct"
      },
      { // 直连中国大陆主流网站 ip 和 保留 ip
        "type": "field", 
        "ip": [
          "geoip:private",
          "geoip:cn",
          "0.0.0.0/8",
          "10.0.0.0/8",
          "100.64.0.0/10",
          "127.0.0.0/8",
          "169.254.0.0/16",
          "172.16.0.0/12",
          "192.0.0.0/24",
          "192.0.2.0/24",
          "192.168.0.0/16",
          "198.18.0.0/15",
          "198.51.100.0/24",
          "203.0.113.0/24",
          "::1/128",
          "fc00::/7",
          "fe80::/10"
        ],
        "outboundTag": "direct"
      },
      { // 直连中国大陆主流网站域名
        "type": "field", 
        "domain": [
          "geosite:cn"
        ],
        "outboundTag": "direct"
      }
    ]
  }
}

```

## 验证结果

在你的路由器中将DHCP的默认网关修改为本例的透明代理IP地址，然后手机或电脑重新连接网络一下检查默认网关是192.168.20.18后，就可以通过浏览器访问百度、谷歌验证了。

## 拓扑图

最后附上拓扑图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/521d493d394245cd9d2a98b6c6245dd2.png)

> 如图中容器和宿主机之间的互相访问，通过给宿主机添加路由和给容器添加iptables规则来实现。

---

（END）
