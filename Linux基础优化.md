## Linux基础优化

1. 规范目录

   ```bash
   mkdir -p /server/tools
   mkdir -p /server/scripts
   ```

2. 配置所有主机域名解析

   ```bash
   cat >/etc/hosts<<EOF
   127.0.0.1    localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1          localhost localhost.localdomain localhost6 localhost6.localdomain6
   172.16.1.5 lb01
   172.16.1.6 lb02
   172.16.1.7 web01
   172.16.1.8 web02
   172.16.1.9 web03
   172.16.1.31 nfs01
   172.16.1.41 backup
   172.16.1.51 db01 db01.etiantian.org
   172.16.1.61 m01
   EOF
   ```

3. 更新yum源

   ```bash
   # Base源
   curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
   # epel源
   curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
   ```

4. 安全优化

   ```bash
   # 关闭selinux
   sed -i 's#SELINUX=.*#SELINUX=disabled#g' /etc/selinux/config
   sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   grep SELINUX=disabled /etc/selinux/config
   setenforce 0
   getenforce
   
   # 关闭firewalld防火墙
   systemctl stop firewalld
   systemctl disable firewalld
   ```

5. 设置普通用户提升权限

   ```bash
   useradd zhanghao
   echo 123456|passwd --stdin zhanghao
   \cp /etc/sudoers /etc/sudoers.ori
   echo "zhanghao  ALL=(ALL) NOPASSWD: ALL " >>/etc/sudoers
   tail -1 /etc/sudoers
   visudo -c
   ```

6. 设置中文字符集

   ```bash
   cp /etc/locale.conf /etc/locale.conf.ori
   # 或: localectl set-locale LANG="zh_CN.UTF-8"
   echo 'LANG="zh_CN.UTF-8"' >/etc/locale.conf
   source /etc/locale.conf
   echo $LANG
   ```

7. 时间同步

   ```bash
   # 下载ntpdate
   yum install ntpdate -y
   /usr/sbin/ntpdate ntp3.aliyun.com
   # 定时同步时间
   echo '#crond-id-001:time sync by zhanghao' >>/var/spool/cron/root
   echo "*/5 * * * * /usr/sbin/ntpdate ntp3.aliyun.com >/dev/null 2>&1">>/var/spool/cron/root
   crontab -l
   ```

8. 提升命令行操作安全性

   ```bash
   # 不操作后5分钟退出
   echo 'export TMOUT=300' >>/etc/profile
   # 最多显示5条操作记录
   echo 'export HISTSIZE=5' >>/etc/profile
   # 文件中保留5条操作记录
   echo 'export HISTFILESIZE=5' >>/etc/profile
   tail -3 /etc/profile
   . /etc/profile
   ```

9. 加大文件描述符

   ```bash
   echo '*               -       nofile          65535 ' >>/etc/security/limits.conf 
   tail -1 /etc/security/limits.conf
   ulimit -SHn   65535 
   ulimit -n
   ```

10. 优化系统内核参数

    ```bash
    cat >>/etc/sysctl.conf<<EOF
    net.ipv4.tcp_fin_timeout = 2
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.tcp_syncookies = 1
    net.ipv4.tcp_keepalive_time = 600
    net.ipv4.ip_local_port_range = 4000    65000
    net.ipv4.tcp_max_syn_backlog = 16384
    net.ipv4.tcp_max_tw_buckets = 36000
    net.ipv4.route.gc_timeout = 100
    net.ipv4.tcp_syn_retries = 1
    net.ipv4.tcp_synack_retries = 1
    net.core.somaxconn = 16384
    net.core.netdev_max_backlog = 16384
    net.ipv4.tcp_max_orphans = 16384
    #以下参数是对iptables防火墙的优化，防火墙不开会提示，可以忽略不理。
    net.nf_conntrack_max = 25000000
    net.netfilter.nf_conntrack_max = 25000000
    net.netfilter.nf_conntrack_tcp_timeout_established = 180
    net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
    net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
    net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
    net.core.wmem_default = 8388608
    net.core.rmem_default = 8388608
    net.core.wmem_max = 16777216
    net.core.rmem_max = 16777216
    EOF
    sysctl -p
    ```

11. 安装常用软件

    ```bash
    # centos6 + centos7
    yum install tree nmap dos2unix lrzsz nc lsof wget tcpdump htop iftop iotop sysstat nethogs -y
    # centos7
    yum install psmisc net-tools bash-completion vim-enhanced -y
    ```

12. 优化SSH远程连接效率

    * 禁止root远程连接
    * 修改默认22端口，改为52113

13. 修改yum.conf文件配置信息，保留yum安装的软件包

    * 将/etc/yum.conf中的keepcache=0改为keepcache=1，为日后一键安装网站集群留好rpm及依赖工具包。

14. 锁定关键系统文件如/etc/passwd、/etc/shadow、/etc/group、/etc/gshadow、/etc/inittab， 
    处理以上内容后把chattr、lsattr改名为oldboy，转移走，这样就安全多了。

15. 清空/etc/issue、/etc/issue.net，去除系统及内核版本登录前的屏幕显示。

16. 清除多余的系统虚拟用户账号。

17. 为grub引导菜单加密码。

18. 禁止主机被ping（内核参数）。

19. 打补丁并升级有已知漏洞的软件。

    ```bash
    yum update
    ```

20. 精简开机自启动服务

    ```bash
    systemctl list-unit-files |grep enable|egrep -v "sshd.service|crond.service|sysstat|rsyslog|^NetworkManager.service|irqbalance.service"|awk '{print "systemctl disable",$1}'|bash
    systemctl list-unit-files |grep enable
    netstat -lntup
    # 保留服务：
    # sshd|crond|sysstat|rsyslog|NetworkManager|irqbalance
    ```

    