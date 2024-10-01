---
{"dg-publish":true,"dg-permalink":"wireguard","permalink":"/wireguard/","title":"EdgeRoutor ER-X WireGuard配置指南","tags":["折腾","网络"],"created":"2022-02-22 17:00","updated":"2024-10-01 10:07"}
---


# EdgeRoutor ER-X WireGuard配置指南

最近入手了一只er-x作为主路由，考虑可以随时漫游回家，er-x支持的vpn协议很多，也尝试配置了L2TP和OpenVPN，但是都不成功，且配置太难懂，经历了无数天的头疼和尝试之后，转向了wireguard。

## WireGuard介绍

了解wiredguard的概念，可以更好的理解他的配置。

WG 先定义了一个很重要的概念 —— WireGuard Interface（以下简称 wgi）。为什么要有 wgi？为什么现有的 tunnel 接口不适合？一个 wgi 是这么一个特殊的接口：

- 有一个自己的私钥（curve25519）
- 有一个用于监听数据的 UDP 端口
- 有一组 peer（peer 是另一个重要的概念），每个 peer 通过该 peer 的公钥确认身份

通过定义这样一个新的接口，wgi 把它和一般的 tunnel 接口区分开。有了这样一个接口的定义，其它数据结构的挂载，以及数据的收发都很清晰明了了。

我们看 WG 的接口配置：

```ini
[Interface]
Address = 10.1.1.1/24
ListenPort = 12345
PrivateKey = blablabla

[Peer]
PublicKey = <server public key>
AllowedIPs = 0.0.0.0/0,::/0
Endpoint = 1.1.1.1:54321
```

WG 的 VPN 隧道的发起者（initiator）/ 接收者（responder）是对等的，所以也没有一般 VPN 的客户端/服务器端或者 spoke/hub 的区别。因而配置也是对等的。

在这个配置中，我们进一步了解了 peer 这个概念：它是 WG 节点的对端，有静态配置的公钥，peer 背后的网络的白名单（AllowedIPs），以及 peer 的地址和端口（这个并不一定需要，并且随着网络的漫游，可能会自动更改）

> [!important]  
> 我的理解：interface是当前wireguad端，peer是与其配对的另一方端每一端的interface声明了当前端的地址，监听的端口，和私钥peer配置对方的公钥和对方的地址  

### 我自己的理解

首先，在wg中，定义上是没有服务端和客户端的分别，使用上还是有的。每个wg实例，都有interface（接口）配置和peer（对端）配置

- 接口配置：
	- Address, 当前wg实例分配的wg局域网地,例如上述定义了当前实例地址为10.1.1.1。当客户端被当作中继服务器（server端）时，后面的/24意思是，只有10.1.1.x才属于相同的wg局域网，否则/24没有含义。也可以直接使用/32
	- ListenPort：当前实例监听的地址
	- PrivateKey：当前实例的私钥
- 对端配置
	- PublicKey：对端实例的公钥
	- AllowedIPs：哪些本机流量会发送到该对端
	- Endpoint: 非必须配置，当作为客户端时，配置对方的地址

以以下我实际使用的配置为例

server配置：
```
wireguard wg0 {
	address 10.6.69.1/24
	listen-port 51820
	mtu 1420
	peer public_key_1 {
		allowed-ips 10.6.69.2/32
		description mac
	}
	peer public_key_2 {
		allowed-ips 10.6.69.4/32
		description phone
	}
	peer public_key_3 {
		allowed-ips 10.6.69.3/32
		description phone2
	}
	private-key ****************
	route-allowed-ips true
}
```
 - 当前实例地址10.6.69.1
 - 监听51280
 - 定义三个对端，因为这里是当作server端，所以每个对端不需要endpoint，因为该实例收到对端的请求后，不会发送给新的对端，只需要返回响应
 - 所以每个对端的allowed-ips就是客户端应该设定的实例地址
 

 client配置：
```
[Interface]
Address = 10.6.69.2/24
DNS = 192.168.99.1
ListenPort = 51820
PrivateKey = xxxx

[Peer]
AllowedIPs = 192.168.99.0/24
Endpoint = <your doamin>:<your port>
PublicKey = <server public key>
```
- 该实例地址10.6.68.9
- 客户端端口可以不指定，随机选取
- 允许192.168.99.x的请求发送到对端，如果需要路由所有流量，改成0.0.0.0/0
- 定义了对端的地址

## 配置WireGuard(EdgeRoutor)

虽然WireGuard概念上不区分服务端客户端，但是总是有个端来提供服务，其他一堆客户端链接他。所以我们还是按照传统的概念描述吧

### 安装

1. 从[WireGuard Github](https://github.com/WireGuard/wireguard-vyatta-ubnt/releases)下载官方安装包
    ```Shell
    curl -OL https://github.com/WireGuard/wireguard-vyatta-ubnt/releases/download/1.0.20210606-2/e50-v2-v1.0.20210606-v1.0.20210914.deb
    ```
2. 安装deb文件
    ```Shell
    dpkg -i e50-v2-v1.0.20210606-v1.0.20210914.deb
    ```

通过执行`**show interfaces**`可以确认是否安装成功

### 配置

1. 生成服务端的密钥对
   
    ```Shell
    mkdir server_keys
    cd server_keys
    wg genkey | tee privatekey | wg pubkey > publickey
    ```
    
2. 相似的，生成多个客户端的密钥对
   
    ```Shell
    mkdir my_phone
    cd my_phone
    wg genkey | tee privatekey | wg pubkey > publickey
    ```
    
3. 配置wg0 interface
   
    ```Shell
    # Enter configure mode
    configure
    
    # The location of the server's private key, previously generated
    set interfaces wireguard wg0 private-key [your server private key data]
    
    # Creates the Gateway IP for the VPN and the subnet
    # This subnet can be any private IP range, through check for conflicts 
    set interfaces wireguard wg0 address 10.6.69.1/24
    
    # Creates entries in the route table for the VPN subnet
    set interfaces wireguard wg0 route-allowed-ips true
    
    # Port for WG (that peers will use)
    set interfaces wireguard wg0 listen-port 51820
    
    commit ; save
    ```
    
4. 添加一个客户端peer
    ```Shell
    # Remote User Peer
    set interfaces wireguard wg0 peer [your client public key here]
    # set the client allocate ip
    set interfaces wireguard wg0 peer [your client public key here] allowed-ips 10.6.69.2/32
    commit ; save
    ```
    注意客户端的ip要每个客户端不一样
    
5. 开启防火墙
    
    ```Shell
    # Creates an accept rule in the WAN_LOCAL list (WAN_LOCAL - wan to router)
    # Accepts all incoming UDP connections, from port 51820
    
    set firewall name WAN_LOCAL rule 20 action accept
    set firewall name WAN_LOCAL rule 20 protocol udp
    set firewall name WAN_LOCAL rule 20 destination port 51820
    set firewall name WAN_LOCAL rule 20 description 'WireGuard'
    
    commit ; save
    ```
    
>[!important]  
> 家宽还要开启端口转发  

最后生成的配置类似如下

```Plain
user@ER-X$ show configuration

}
    }
    wireguard wg0 {
        address 10.6.69.1/24
        description WG_VPN
        listen-port 51820
        peer <peer public key> {
            allowed-ips 10.6.69.2/32
            description my_phone
        }
        private-key ****************
        route-allowed-ips true
    }
}

        }
        rule 20 {
            action accept
            description WG_IN
            destination {
                port 51820
            }
            log enable
            protocol udp
            source {
            }
        }
```

### 客户端配置

新建wg.conf文件，写入如下内容

```ini
[Interface]
PrivateKey = <my_phone private key>
ListenPort = 51820
Address = 10.6.69.2/32                                                   
DNS=192.168.99.1

[Peer]
PublicKey = <pubkey of server>
AllowedIPs = 0.0.0.0/0                      
Endpoint = server.com:51820
```

Interface设置
- ListenPort：客户端本地监听端口，可以不设置，随机选择
- Address：分配的ip
- DNS：可以不设置，走wg的流量使用的DNS

Peer设置
- AllowedIPs：允许的地址段，目前设置的是不限制

下载wire官方客户端，导入上述配置文件，大功告成。

> [!suggest]
> 我的建议是，不提供ListenPort，DNS参数，避免端口占用，和访问其他网络的问题；AllowedIPs可以配置家里的局域网网段+wg的网段即可

### 新增第二个客户端

同上面的过程创建客户端的密钥

然后设置peer信息

```
# Remote User Peer
set interfaces wireguard wg0 peer [your client public key here]
# set the client allocate ip
set interfaces wireguard wg0 peer [your client public key here] allowed-ips 10.6.69.3/32
commit ; save
```

>[!warn]
>注意：新的客户端allowed-ips必须是未分配的。多个客户端不能使用相同的配置，会造成冲突在线

新客户端的配置如下

```ini
[Interface]
PrivateKey = <my_phone private key>
ListenPort = 51820
Address = 10.6.69.3/32                        
DNS = 192.168.99.1                            

[Peer]
PublicKey = <pubkey of server>
AllowedIPs = 0.0.0.0/0                      
Endpoint = server.com:51820                  
```

## 使用用遇到的问题

### 不能访问公司内网

mac客户端配置后不能访问公司内网，主要是由于DNS错误导致的，建议删除客户端配置中的DNS，会默认使用公司Wi-Fi默认的DNS

### IOS配置后不能访问外网

ios和mac有点相反，不配置dns只能访问局域网，加上DNS配置后一切正常

### er-x调试

1. 输入`sudo wg`可查看接口连接信息
2. `allowed-ips`配置错误需要重启路由器才能恢复

### 两个客户端直接不能访问

1. 排查allowed-ips是否包括的wg的局域网网段

## 没有公网ip情况下的中继示例

参考[[搭建 wireguard 中继服务器连通两个没有公网 IP 的局域网 \|搭建 wireguard 中继服务器连通两个没有公网 IP 的局域网 ]]

核心的要点是：

- 局域网A和B的局域网网段不能一样（在A-B直通的情况下关系不大，因为wg可以代理本机流量，最多是本机不同，但是有server中继时，就没法路由了）
- server的peerA中， AllowedIPs除了给peerA分配的wg局域网ip，还需要局域网A的掩码信息
- A和B的peer都是server

这样，在A中，输入B的局域网地址，首先被发送到server，server解析出目标地址是B的局域网，根据AllowedIPs配置，会发送到peerB
B收到请求，会根据来源发送给server，server再发给A。

## 参考链接

1. [Wireguard VPN on a Ubiquiti EdgeRouter | Usman](https://blog.usman.network/posts/wireguard-vpn-on-a-ubiquiti-edgerouter/#1-generating-server-key-pair)
2. [Releases · WireGuard/wireguard-vyatta-ubnt](https://github.
com/WireGuard/wireguard-vyatta-ubnt/releases)
3. [Wireguard：简约之美 - 知乎](https://zhuanlan.zhihu.com/p/91383212)
