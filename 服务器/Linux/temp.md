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

**关闭 selinux**

- 查看状态：`sestatus`

- 临时关闭：`setenforce 0`
- 永久关闭：修改`/etc/selinux/config`设置`SELINUX=disabled`后重启

## CHMOD

所以，`chmod +X` 主要用于在已有执行权限的基础上，为目录添加可执行权限或者保持已有执行权限。对于文件，它的作用相对有限，因为通常文件不需要直接的可执行权限。

而 `chmod 755` 则是直接将文件或目录设置为一组特定的权限：所有者有读、写、执行权限；组用户和其他用户有读、执行权限，但没有写权限。

## 最大打开文件句柄数

使用 `ulimit -n` 查看单进程允许打开的的最大文件句柄数，使用`ulimit -n 65536`来临时修改，或修改`/etc/security/limits.conf`配置文件来永久修改：

```bash
*     soft    nofile   65536
*     hard    nofile   65536
```

还有一个系统允许的最大文件句柄数，通过`cat /proc/sys/fs/file-max`查看设置，在`/etc/sysctl.conf`中添加`fs.file-max=65536`，然后执行`sysctl -p`修改系统允许的最大文件句柄数。

## 环境参数

**Bash**：

- 对于个别用户，编辑 `~/.bashrc` 文件
- 对于所有用户，可以编辑 `/etc/bash.bashrc` 文件（需要 root 权限）

```
export PATH=$PATH:/your/new/path
```

运行 `source ~/.bashrc`（或对应的配置文件）来立即使更改生效。
