**开启 ipv4 转发**

在 Linux 中，可以使用 `sysctl` 命令来修改内核参数。确保以下步骤：

1. 打开终端。

2. 检查当前 IPv4 转发的状态：

   ```bash
   sysctl net.ipv4.ip_forward
   ```

   如果返回值为 `1`，表示已启用 IPv4 转发。如果返回值为 `0`，则表示禁用。

3. 若要启用 IPv4 转发，执行以下命令：

   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

   或者，编辑 `/etc/sysctl.conf` 文件，在文件末尾添加或修改以下行：

   ```bash
   net.ipv4.ip_forward = 1
   ```

   > **命令方式添加**
   >
   > ```bash
   > cat >> /etc/sysctl/conf <<- 'EOF'
   > net.ipve.ip_forward=1
   > EOF
   > ```

   然后运行以下命令以使更改生效：

   ```bash
   sudo sysctl -p
   ```

临时开启：`echo 1 > /proc/sys/net/ipv4/ip_forward`

**添加网关路由**

临时：`route add -net 192.168.11.0/24 gw 192.168.13.128`

永久：

1. 在 `/etc/network/interfaces` 中使用 `route` 命令和 `up` 选项

   打开 `/etc/network/interfaces` 文件，添加以下行：

   ```bash
   up route add -net 192.168.11.0/24 gw 192.168.13.128
   ```

   保存文件后，重新启动网络服务或重启系统。

2. 在 `/etc/network/interfaces` 中使用 `post-up`

   或者，你可以使用 `post-up`：

   ```bash
   post-up route add -net 192.168.11.0/24 gw 192.168.13.128
   ```

   保存文件后，重新启动网络服务或重启系统。