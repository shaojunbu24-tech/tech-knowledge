# Linux 常用命令面试速查

> 分类：操作系统
> 标签：Linux, 命令行, Shell, 文件操作, 进程管理, 网络诊断, 文本处理

## 面试精答

> 适合面试时口述的简洁版本

面试中 Linux 命令主要考察四个方向：**文件操作、文本处理、进程管理、网络排查**。不需要背几百个命令，掌握高频的 30 个左右就行。

### 一、文件与目录

| 命令 | 作用 | 常用参数 |
|---|---|---|
| `ls` | 列出目录内容 | `-l` 详细信息，`-a` 含隐藏文件，`-lh` 人类可读大小 |
| `cd` | 切换目录 | `cd -` 回到上一个目录 |
| `pwd` | 显示当前路径 | |
| `mkdir` | 创建目录 | `-p` 递归创建（`mkdir -p a/b/c`） |
| `cp` | 复制 | `-r` 递归复制目录 |
| `mv` | 移动/重命名 | |
| `rm` | 删除 | `-r` 递归，`-f` 强制。**慎用 `rm -rf /`** |
| `find` | 查找文件 | `find / -name "*.log" -mtime +7` 找 7 天前的日志 |
| `tar` | 打包/解包 | `tar -czf a.tar.gz dir/` 打包压缩，`tar -xzf a.tar.gz` 解包 |
| `ln` | 创建链接 | `-s` 软链接（快捷方式） |
| `chmod` | 修改权限 | `chmod 755 file`，`chmod +x file` 加执行权限 |
| `chown` | 修改所有者 | `chown user:group file` |
| `du` | 磁盘使用 | `-sh *` 看每个目录的大小 |
| `df` | 磁盘空间 | `-h` 人类可读 |

**文件权限**（面试常问）：

```
-rwxr-xr-- 1 user group 1024 May 5 10:00 file.sh
│├──┤├──┤├──┤
│ │   │   │
│ │   │   └── 其他用户：r--（只读）
│ │   └────── 同组用户：r-x（读+执行）
│ └────────── 所有者：rwx（读写执行）
└──────────── - 表示普通文件（d 表示目录，l 表示链接）

数字表示：r=4, w=2, x=1
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
所以 chmod 754 file.sh = rwxr-xr--
```

### 二、文本处理（面试最高频）

| 命令 | 作用 | 示例 |
|---|---|---|
| `grep` | 文本搜索 | `grep "ERROR" app.log` 搜错误日志 |
| `awk` | 列处理 | `awk '{print $1}' access.log` 取第 1 列 |
| `sed` | 流编辑器 | `sed 's/old/new/g' file` 替换文本 |
| `sort` | 排序 | `sort -n` 数字排序，`-r` 倒序，`-k2` 按第 2 列 |
| `uniq` | 去重 | `uniq -c` 去重并计数（需先 sort） |
| `head` | 看前 N 行 | `head -100 file` |
| `tail` | 看后 N 行 | `tail -f app.log` 实时追踪日志（排查利器） |
| `wc` | 统计 | `wc -l file` 统计行数 |
| `cat` | 查看文件 | `cat file1 file2 > merged` 合并文件 |
| `less` | 分页查看 | 支持上下翻页和搜索（比 `more` 强） |
| `cut` | 列截取 | `cut -d',' -f2 file.csv` 以逗号分隔取第 2 列 |
| `tr` | 字符替换 | `tr 'A-Z' 'a-z'` 大写转小写 |

### 三、进程管理

| 命令 | 作用 | 示例 |
|---|---|---|
| `ps` | 查看进程 | `ps aux` 看所有进程，`ps -ef` 另一种格式 |
| `top` | 实时进程监控 | 按 CPU/内存排序，看系统负载 |
| `htop` | 增强版 top | 交互更友好（需要安装） |
| `kill` | 发送信号 | `kill -9 PID` 强制杀进程，`kill -15 PID` 优雅退出 |
| `nohup` | 后台运行 | `nohup ./server &` 退出终端不中断 |
| `&` | 后台执行 | `./server &` |
| `jobs` | 查看后台任务 | |
| `fg/bg` | 前/后台切换 | `fg %1` 把任务 1 调到前台 |

**ps aux 输出含义**：

```
USER   PID %CPU %MEM  VSZ   RSS TTY STAT START  TIME COMMAND
root    1  0.0  0.1 169344 13000 ?   Ss   May01  0:15 /sbin/init

PID  — 进程 ID（kill 命令用的就是这个）
%CPU — CPU 使用率
%MEM — 内存使用率
RSS  — 实际物理内存使用（KB）
STAT — 进程状态：S=睡眠，R=运行，Z=僵尸进程
```

### 四、网络排查

| 命令 | 作用 | 典型用法 |
|---|---|---|
| `ping` | 测试连通性 | `ping 8.8.8.8` |
| `curl` | HTTP 请求 | `curl -v https://api.example.com` |
| `netstat` | 网络连接状态 | `netstat -tlnp` 看监听端口 |
| `ss` | netstat 替代 | `ss -tlnp`（更快） |
| `lsof` | 查看打开的文件/端口 | `lsof -i :8080` 谁占了 8080 端口 |
| `nslookup` | DNS 查询 | `nslookup example.com` |
| `dig` | DNS 详细查询 | `dig example.com` |
| `traceroute` | 路由追踪 | `traceroute example.com` |
| `tcpdump` | 抓包 | `tcpdump -i eth0 port 80` |
| `wget` | 下载文件 | `wget https://example.com/file.tar.gz` |
| `scp` | 远程复制 | `scp file user@host:/path` |
| `ssh` | 远程登录 | `ssh user@host` |

**netstat 看连接状态**（面试常问）：

```
netstat -an | grep ESTABLISHED   → 已建立的连接
netstat -an | grep TIME_WAIT     → 等待关闭的连接（太多说明短连接过多）
netstat -an | grep LISTEN        → 正在监听的端口

输出：
Proto Recv-Q Send-Q Local Address    Foreign Address    State
tcp   0      0      0.0.0.0:8080     0.0.0.0:*          LISTEN
tcp   0      0      192.168.1.10:80  10.0.0.5:54321     ESTABLISHED
tcp   0      0      192.168.1.10:80  10.0.0.5:54322     TIME_WAIT
```

### 五、系统信息

| 命令 | 作用 |
|---|---|
| `uname -a` | 系统信息（内核版本等） |
| `free -h` | 内存使用情况 |
| `uptime` | 系统运行时间 + 负载 |
| `vmstat` | 虚拟内存统计 |
| `iostat` | CPU 和 I/O 统计 |
| `dmesg` | 内核日志（硬件错误、OOM 等） |
| `journalctl` | systemd 日志查看 |

### 六、管道与重定向

**管道（|）**：把前一个命令的输出作为后一个命令的输入。

```bash
# 统计 access.log 中每个 IP 的访问次数，取前 10
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# 每一步的输出：
cat access.log        → 所有日志行
  | awk '{print $1}'  → 只取第 1 列（IP 地址）
  | sort              → 排序（uniq 要求输入有序）
  | uniq -c           → 去重并计数（每个 IP 出现几次）
  | sort -rn          → 按数字倒序（最多的排前面）
  | head -10          → 取前 10 个
```

**重定向**：

```bash
command > file     # 标准输出重定向到文件（覆盖）
command >> file    # 追加
command 2> file    # 错误输出重定向
command > file 2>&1  # 标准输出和错误都重定向到文件
command < file     # 从文件读取输入
```

### 七、实战组合命令（面试加分）

```bash
# 查找占用 8080 端口的进程并杀掉
lsof -i :8080 | grep LISTEN | awk '{print $2}' | xargs kill -9

# 找出最大的 10 个文件
du -ah / | sort -rh | head -10

# 统计代码行数（排除空行和注释）
find . -name "*.java" | xargs grep -v "^$\|^[[:space:]]*//" | wc -l

# 实时监控错误日志
tail -f app.log | grep --color "ERROR\|Exception"

# 批量替换文件内容
sed -i 's/old_host/new_host/g' *.conf

# 查看某个进程的详细资源占用
top -p $(pgrep -d',' java)

# 统计目录下各类型文件数量
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# 查看系统负载和内存（快速巡检）
echo "=== Load ===" && uptime && echo "=== Memory ===" && free -h && echo "=== Disk ===" && df -h
```

## 常见追问与应对

**Q1: `kill -9` 和 `kill -15` 有什么区别？**
> `-15`（SIGTERM）是优雅退出信号，进程可以捕获这个信号做清理工作（释放资源、保存状态）后再退出。`-9`（SIGKILL）是强制杀掉，进程无法捕获或忽略，立即终止，可能导致数据丢失或资源泄漏。应该先用 `kill -15`，等几秒没反应再用 `kill -9`。

**Q2: 怎么看一个 Java 应用为什么 CPU 飙高？**
> 三步排查：1）`top` 找到 CPU 最高的 Java 进程 PID；2）`top -Hp PID` 找到该进程中 CPU 最高的线程 TID；3）`printf '%x\n' TID` 把线程 ID 转成十六进制，然后 `jstack PID | grep 十六进制TID -A 30` 看该线程在干什么（通常会发现某个方法死循环或频繁 GC）。

**Q3: `tail -f` 和 `tail -F` 有什么区别？**
> `-f` 监控文件变化，但如果文件被删除重建（如日志轮转 logrotate），`-f` 会断开。`-F`（= `-f --follow=name --retry`）会自动重新打开新文件，不会断开。生产环境排查问题用 `tail -F`。

**Q4: 管道（pipe）和重定向（redirect）的区别？**
> 管道连接两个进程：前一个命令的 stdout 直接变成后一个命令的 stdin，在内存中传递，不落盘。重定向是把 stdout/stderr 写到文件中。管道用于命令间协作，重定向用于把结果保存到文件。

## 知识关联

- [进程、线程、协程的区别与联系](./进程-线程-协程.md)
- [线程创建方式、生命周期与状态](./线程创建-生命周期-状态.md)
- [第四层：传输层](../计算机网络/传输层.md)
- [JVM — Java 虚拟机](../编程语言/JVM-%20Java虚拟机.md)
