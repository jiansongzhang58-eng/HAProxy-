# HAProxy搭建

以下是基于三台 openEuler 24.03 虚拟机（IP: 192.168.48.161、192.168.48.162、192.168.48.163）搭建 HAProxy 的详细步骤。假设目标是配置一个负载均衡环境，其中 192.168.48.161 作为 HAProxy 服务器，其余两台作为后端服务器。

------

### 1. 环境准备

确保所有虚拟机网络互通，防火墙允许相关流量（如 HTTP/HTTPS 或自定义端口）。
以下操作需在每台机器上执行，但注意区分角色（HAProxy 服务器或后端服务器）。

------

### 2. 配置后端服务器（192.168.48.162 和 192.168.48.163）

在两台后端服务器上安装 Web 服务（如 Nginx）并启动，用于演示负载均衡效果。

#### 安装并启动 Nginx：

```BASH
dnf install nginx -y
systemctl start nginx
systemctl enable nginx
```

#### 创建测试页面（区分两台服务器）：

在 **192.168.48.162** 上：

```BASH
echo "Backend Server 1" > /usr/share/nginx/html/index.html
```

在 **192.168.48.163** 上：

```BASH
echo "Backend Server 2" > /usr/share/nginx/html/index.html
```

#### 验证 Nginx 访问：

```BASH
curl http://localhost
```

------

### 3. 配置 HAProxy 服务器（192.168.48.161）

#### 安装 HAProxy：

```BASH
dnf install haproxy -y
```

#### 备份原始配置并编辑新配置：

```BASH
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
vim /etc/haproxy/haproxy.cfg
```

#### 替换为以下配置内容（根据需求调整）：

```HAPROXY
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend http_front
    bind *:80
    stats uri /haproxy?stats
    default_backend http_back

backend http_back
    balance roundrobin
    server backend1 192.168.48.162:80 check
    server backend2 192.168.48.163:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /monitor
    stats refresh 5s
    stats auth admin:your_password
```

#### 根据配置创建文件夹

```
mkdir -p /etc/haproxy/errors
```

```
cat > /etc/haproxy/errors/400.http << EOF
HTTP/1.0 400 Bad Request
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>400 Bad Request</h1></body></html>
EOF

cat > /etc/haproxy/errors/403.http << EOF
HTTP/1.0 403 Forbidden
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>403 Forbidden</h1></body></html>
EOF

cat > /etc/haproxy/errors/408.http << EOF
HTTP/1.0 408 Request Timeout
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>408 Request Timeout</h1></body></html>
EOF

cat > /etc/haproxy/errors/500.http << EOF
HTTP/1.0 500 Internal Server Error
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>500 Internal Server Error</h1></body></html>
EOF

cat > /etc/haproxy/errors/502.http << EOF
HTTP/1.0 502 Bad Gateway
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>502 Bad Gateway</h1></body></html>
EOF

cat > /etc/haproxy/errors/503.http << EOF
HTTP/1.0 503 Service Unavailable
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>503 Service Unavailable</h1></body></html>
EOF

cat > /etc/haproxy/errors/504.http << EOF
HTTP/1.0 504 Gateway Timeout
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>504 Gateway Timeout</h1></body></html>
EOF

```

```
mkdir -p /run/haproxy
chown haproxy:haproxy /run/haproxy
chmod 755 /run/haproxy


mkdir -p /var/lib/haproxy
chown haproxy:haproxy /var/lib/haproxy
chmod 755 /var/lib/haproxy
```



#### 启动 HAProxy 并设置开机自启：

```BASH
systemctl start haproxy
systemctl enable haproxy
```

#### 检查 HAProxy 状态：

```
systemctl status haproxy
```

------

### 4. 验证负载均衡

从客户端访问 HAProxy 服务器的 IP（192.168.48.161）多次，观察返回的页面内容是否交替显示 `Backend Server 1` 和 `Backend Server 2`：

```
curl http://192.168.48.161
```

#### 查看 HAProxy 统计页面：

通过浏览器访问 `http://192.168.48.161/haproxy?stats`（用户名 `admin`，密码 `your_password`）查看监控信息。

------

### 5. 防火墙配置（如需）

若防火墙开启，需放行 HAProxy 相关端口（如 80、8404）：

```
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8404/tcp
firewall-cmd --reload
```

------

### 常见问题排查

1. HAProxy 无法启动：检查配置文件语法：

   ```
   haproxy -c -f /etc/haproxy/haproxy.cfg
   ```

2. **后端服务器不可达**：确保网络互通且后端服务正常监听端口。

3. **权限问题**：确保 SELinux 或防火墙未阻断流量。

------

以上步骤已完成 HAProxy 的安装和基础负载均衡配置。根据实际需求可调整监听协议、负载算法或添加 SSL 等高级配置。

从您的 `netstat` 输出可以看到：

1. **HAProxy 只监听了80端口**（PID 2213），**没有监听8404端口**
2. **可以正常访问** `http://192.168.48.161/haproxy?stats`

------



## 两种访问方案

您的配置中 `listen stats` 段没有生效，因为：

**配置冲突**：您在 `frontend` 段中已经配置了 `stats uri /haproxy?stats`，这会启用80端口的统计页面。而 `listen stats` 段是独立的监控配置，两者只能选其一。

------

### 解决方案（二选一）

#### 方案A：使用80端口的监控（当前已生效）

保持现有配置，访问：`http://192.168.48.161/haproxy?stats`

#### 方案B：使用独立监控端口（8404）

修改配置，删除 `frontend` 中的 stats 配置：

```
frontend http_front
    bind *:80
    # 删除或注释下面这行
    # stats uri /haproxy?stats
    default_backend http_back

# 保留 listen stats 段
listen stats
    bind *:8404
    stats enable
    stats uri /monitor
    stats refresh 5s
    stats auth admin:your_password
```

然后重启 HAProxy：

```
systemctl restart haproxy
```

------

### 验证8404端口是否监听

```
netstat -ntlp | grep 8404
```

如果显示监听，现在就可以访问：`http://192.168.48.161:8404/monitor`

------

### 总结

- **当前状态**：80端口的监控页面正常工作
- **如果要使用8404端口**：需要修改配置并重启服务
- **建议**：如果80端口监控能满足需求，无需修改配置
