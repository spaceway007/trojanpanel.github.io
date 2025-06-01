
背景提要
本指南适用于在OpenVZ虚拟化的VPS上，通过Docker（特别是Trojan Panel的一键脚本）部署Hysteria2的场景。本文档旨在解决因OpenVZ内核限制（如不支持DNAT模块）和Docker路径映射不当等问题，最终实现Hysteria2的端口跳跃功能。

第一步：找出Hysteria2的真实监听端口
我们首先需要知道Hysteria2服务到底在监听哪个端口，而不是依赖可能会出错的自动生成文件。
查看容器内进程，找到配置文件路径： 执行此命令列出Trojan Panel核心容器内的进程。

 Bash
docker exec trojan-panel-core ps aux
 在输出中找到Hysteria2的进程，并记下 -c 参数后面的配置文件路径。例如： ... bin/hysteria2/hysteria2 -c bin/hysteria2/config/config-4xxx9.json server


查看配置文件内容： 使用上一步找到的路径，查看该配置文件的内容。

 Bash
docker exec trojan-panel-core cat bin/hysteria2/config/config-4xxx9.json


确定端口号： 在输出的JSON内容中，找到 "listen" 字段。这个值就是Hysteria2的真实监听端口。

 JSON
{
   "listen": ":11319",
   ...
}
 在本案例中，我们确定的真实端口是 11319。



第二步：配置防火墙实现端口转发 (核心)
这是实现端口跳跃的核心。由于OpenVZ环境的限制，我们使用 REDIRECT 而非 DNAT。
（可选）清空旧的NAT规则，确保一个干净的环境：

 Bash
iptables -t nat -F


添加端口转发规则： 将您想用于跳跃的端口范围（例如 20000-50000）的UDP流量，全部转发到上一步找到的真实端口（11319）。

 Bash
iptables -t nat -A PREROUTING -p udp --dport 20000:50000 -j REDIRECT --to-port 11319


保存防火墙规则，使其在VPS重启后依然生效： 根据系统提示，可能需要使用 iptables-legacy-save。

 Bash
iptables-legacy-save > /etc/iptables/rules.v4



第三步：（可选）配置HTTP服务以获取端口信息
如果您希望通过HTTP请求获取Hysteria2的真实端口，可以配置Caddy提供一个静态文件。
确定Caddy可访问的共享目录： 通过 docker inspect trojan-panel-caddy 命令查看 Binds 或 Mounts 部分，找到一个主机与容器共享的目录。在本案例中，我们发现 /tpdata/web 是一个可用的共享目录。


在正确的位置创建端口文件：

 Bash
mkdir -p /tpdata/web/public
echo "11319" > /tpdata/web/public/hysteria2_port.txt


修改Caddy配置： 编辑 /tpdata/caddy/config.json 文件，确保admin API被正确配置，并添加一个新的server来提供文件服务。以下是最终的、包含了所有必要修改的完整配置：

 JSON
{
    "admin": {
        "listen": "localhost:2019"
    },
    "apps": {
        "http": {
            "servers": {
                "...": {
                    "//": "您现有的srv0和srv1配置保持不变"
                },
                "staticserver": {
                    "listen": [ ":8888" ],
                    "routes": [
                        {
                            "handle": [
                                {
                                    "handler": "file_server",
                                    "root": "/tpdata/web/public"
                                }
                            ]
                        }
                    ]
                }
            }
        }
    },
    "//": "..."
}
 注意：为保持简洁，此处省略了大量原始配置。核心是在servers中新增staticserver，并确保其root指向正确的共享目录 /tpdata/web/public。


应用Caddy配置：


如果是首次启用admin API，需要重启容器：docker restart trojan-panel-caddy
后续再修改配置，使用重载命令即可：docker exec trojan-panel-caddy caddy reload --config /tpdata/caddy/config.json

第四步：配置客户端
在您的Hysteria2客户端软件中，按以下关键信息配置：
协议/Type: Hysteria2
地址/Host: xxx.www.com
端口/Port: 20000-50000 (与防火墙规则中的范围一致)
SNI/服务器名称: xxx.www.com
OBFS (混淆) 类型: salamander
OBFS 密码: xxxxxxxxxxx (从Hysteria2配置文件中获取)
跳过证书验证/Allow Insecure: 是 或 true

第五步：最终验证
客户端测试：连接客户端，如果能正常代理上网，即代表所有配置成功。
服务器端技术验证：执行 conntrack -L -p udp 命令，可以看到客户端IP访问跳跃端口（如dport=21003）的连接，被正确地关联到了Hysteria2的真实端口（sport=11319），这是端口跳跃正在工作的确凿证据。

