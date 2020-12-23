# CentOS 7 启用 user namespaces（用户命名空间）




在 CentOS 内核 3.8 或更高版本中，添加了 user namespaces （户名命名空间）功能。但是，该功能默认情况下是禁用的，原因是 Red Hat 希望该功能在社区中孵化更长时间，以确保该功能的稳定性和安全性。目前越来越多的软件开始涉及该功能，例如 Docker 等。

<!--more-->

## 配置 CentOS 7 系统启用 user namespaces
注意：以下操作均在 root 用户下完成，或者你的超级用户。

查看系统内核版本：
```shell
uname -r

#3.10.0-1062.el7.x86_64
```

临时配置，重启会失效，可用作临时验证：
```shell
# 查看系统 user namespaces 最大为 0
cat /proc/sys/user/max_user_namespaces
#0
# 临时开启 user namespace ，向文件内写入一个整数。
echo 10000 > /proc/sys/user/max_user_namespaces
```

永久配置，设置 CentOS 7 的 kernel 开启 user namespace ，默认情况下是禁用的。并且，写入/etc/sysctl.conf配置user.max_user_namespaces=10000，最后重启系统。
```shell
# kernel 设置
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 写入配置文件
echo "user.max_user_namespaces=10000" >> /etc/sysctl.conf
# 重启
reboot
```

如需关闭 user namespace ，使用如下命令：
```shell
grubby --remove-args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
```

## 参考资料

https://www.redhat.com/en/blog/whats-next-containers-user-namespaces

https://github.com/procszoo/procszoo/wiki/How-to-enable-%22user%22-namespace-in-RHEL7-and-CentOS7%3F

https://superuser.com/questions/1294215/is-it-safe-to-enable-user-namespaces-in-centos-7-4-and-how-to-do-it
