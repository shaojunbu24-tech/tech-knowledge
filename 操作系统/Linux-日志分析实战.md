# Linux 文本处理实战：取列、排序、去重

> 分类：操作系统
> 标签：Linux, awk, sort, uniq, sed, grep, 文本处理, 日志分析

## 面试精答

> 适合面试时口述的简洁版本

面试官问"怎么查日志的第 n 列，排序，去重"，标准答案是一条管道命令链：

```bash
awk '{print $N}' file | sort | uniq -c | sort -rn
```

每一步的作用：
- `awk '{print $N}'` — 提取第 N 列
- `sort` — 排序（`uniq` 要求输入必须有序）
- `uniq -c` — 去重并统计每项出现次数
- `sort -rn` — 按次数从大到小排列

## 原理深入详解

> 深入底层原理，理解"为什么"

### 一、先把问题具象化

假设有一个 Nginx 访问日志 `access.log`：

```
192.168.1.10 - - [05/May/2026:10:00:01 +0800] "GET /api/user HTTP/1.1" 200 1234
192.168.1.20 - - [05/May/2026:10:00:02 +0800] "POST /api/order HTTP/1.1" 500 5678
192.168.1.10 - - [05/May/2026:10:00:03 +0800] "GET /api/user HTTP/1.1" 200 1234
192.168.1.30 - - [05/May/2026:10:00:04 +0800] "GET /api/order HTTP/1.1" 404 0
192.168.1.20 - - [05/May/2026:10:00:05 +0800] "GET /api/user HTTP/1.1" 200 1234
192.168.1.10 - - [05/May/2026:10:00:06 +0800] "POST /api/order HTTP/1.1" 500 5678
```

列号（默认按空格/tab 分隔）：
```
$1            $2 $3 $4                          $5                    $6  $7
192.168.1.10  -  -  [05/May/2026:10:00:01+0800] "GET /api/user HTTP/1.1" 200 1234
```

**需求**：统计每个 IP 的访问次数，按次数从多到少排序。

### 二、逐个命令拆解

#### 第 1 步：awk — 提取第 N 列

**awk 是什么**：一个强大的文本处理工具，逐行读取文件，对每一行执行指定的操作。名字来自三位创始人（Aho、Weinberger、Kernighan）的姓氏首字母。

**基本语法**：

```bash
awk '模式 {动作}' 文件
```

**取第 N 列**：

```bash
# 默认按空格/tab 分隔，$1 是第 1 列，$2 是第 2 列，$0 是整行
awk '{print $1}' access.log

# 输出：
# 192.168.1.10
# 192.168.1.20
# 192.168.1.10
# 192.168.1.30
# 192.168.1.20
# 192.168.1.10
```

**指定分隔符**：

```bash
# CSV 文件，按逗号分隔，取第 2 列
awk -F',' '{print $2}' data.csv

# 日志按 | 分隔，取第 3 列
awk -F'|' '{print $3}' app.log

# 用 tab 分隔
awk -F'\t' '{print $1}' file.tsv
```

**awk 的列号规则**（面试易混淆）：

```
$0  — 整行内容
$1  — 第 1 列
$2  — 第 2 列
$NF — 最后一列（NF = Number of Fields，总列数）
$(NF-1) — 倒数第 2 列
```

**更复杂的 awk 用法**：

```bash
# 取第 1 列（IP）和第 9 列（HTTP 状态码），中间加空格
awk '{print $1, $9}' access.log

# 只打印状态码为 500 的行的 IP（awk 也能过滤，不需要管道 grep）
awk '$9 == 500 {print $1}' access.log

# 统计每行的列数
awk '{print NF}' file

# 格式化输出（printf）
awk '{printf "%-20s %s\n", $1, $7}' access.log

# 计算第 7 列（响应大小）的总和
awk '{sum += $7} END {print sum}' access.log
```

#### 第 2 步：sort — 排序

**sort 是什么**：对文本行进行排序。默认按字典序（ASCII 码）排。

```bash
# 默认字典序排序
echo -e "banana\napple\ncherry" | sort
# apple
# banana
# cherry

# 按数字排序（-n = numeric）
echo -e "9\n100\n20" | sort      # 错误：100 排在 20 前面（字典序）
echo -e "9\n100\n20" | sort -n   # 正确：9, 20, 100

# 倒序（-r = reverse）
sort -rn   # 按数字倒序（从大到小）

# 按指定列排序（-k = key）
sort -k2   # 按第 2 列排序
sort -k2n  # 按第 2 列的数字值排序

# 去除重复行（-u = unique）
sort -u file   # 排序并去重
```

**常用参数组合**：

| 参数 | 含义 |
|---|---|
| `-n` | 按数字排序 |
| `-r` | 倒序 |
| `-k N` | 按第 N 列排序 |
| `-t ','` | 指定列分隔符（默认 tab/空格） |
| `-u` | 去重 |
| `-h` | 人类可读的数字排序（如 2K, 1M, 3G） |

#### 第 3 步：uniq — 去重

**uniq 是什么**：去除**相邻的**重复行。注意：uniq 只能去除相邻重复行，所以必须先 sort。

```
为什么不先 uniq 再 sort：
  uniq 只能去相邻重复行

  原始：A B A B A
  直接 uniq：A B A B A（没用！A 不相邻）

  先 sort：A A A B B
  再 uniq：A B（正确！）
```

```bash
# 基本去重
sort file | uniq

# 去重并统计每项出现次数（-c = count）
sort file | uniq -c

# 输出格式：次数 + 值
#   3 192.168.1.10
#   2 192.168.1.20
#   1 192.168.1.30

# 只显示重复的行（-d = duplicates）
sort file | uniq -d

# 只显示出现次数 >= N 的行
sort file | uniq -c | awk '$1 >= 2 {print}'
```

### 三、完整管道链：一步步看数据变化

**需求**：统计每个 IP 的访问次数，按次数从多到少排列。

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn
```

```
原始数据：
  192.168.1.10 ...
  192.168.1.20 ...
  192.168.1.10 ...
  192.168.1.30 ...
  192.168.1.20 ...
  192.168.1.10 ...

第 1 步：awk '{print $1}'
  192.168.1.10
  192.168.1.20
  192.168.1.10
  192.168.1.30
  192.168.1.20
  192.168.1.10

第 2 步：sort（字典序排，让相同 IP 相邻）
  192.168.1.10
  192.168.1.10
  192.168.1.10
  192.168.1.20
  192.168.1.20
  192.168.1.30

第 3 步：uniq -c（去重并计数）
       3 192.168.1.10
       2 192.168.1.20
       1 192.168.1.30

第 4 步：sort -rn（按数字倒序，次数多的在前）
       3 192.168.1.10
       2 192.168.1.20
       1 192.168.1.30

最终结果：192.168.1.10 访问了 3 次，最多
```

### 四、更多实战变体

```bash
# 统计访问量前 10 的 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 统计每个 URL 的访问次数
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -20

# 统计 HTTP 状态码分布
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 统计某个时间段的错误日志（grep + awk + sort + uniq）
grep "05/May/2026:10:" error.log | awk '{print $1}' | sort | uniq -c | sort -rn

# 取日志第 3 列，按逗号分隔，去重计数
awk -F',' '{print $3}' data.csv | sort | uniq -c | sort -rn

# 找出访问量超过 1000 次的 IP
awk '{print $1}' access.log | sort | uniq -c | awk '$1 > 1000 {print}' | sort -rn

# 统计每分钟的请求量（提取时间戳到分钟粒度）
awk '{print $4}' access.log | cut -c 14-18 | sort | uniq -c
# 输出：  500 10:00
#         623 10:01
#         789 10:02

# grep 搜索错误 + awk 取第 N 列 + 排序去重
grep "ERROR" app.log | awk -F'|' '{print $3}' | sort | uniq -c | sort -rn | head -10
```

### 五、sed — 另一个高频文本处理工具

`sed`（Stream Editor）主要用于文本替换：

```bash
# 替换每行中第一个匹配
sed 's/old/new/' file

# 替换所有匹配（g = global）
sed 's/old/new/g' file

# 只替换第 2 行
sed '2s/old/new/g' file

# 删除空行
sed '/^$/d' file

# 删除注释行（以 # 开头的）
sed '/^#/d' file

# 直接修改文件（-i = in-place）
sed -i 's/http:\/\/old/https:\/\/new/g' config.conf

# 使用不同的分隔符（避免转义斜杠）
sed 's|http://old|https://new|g' config.conf

# 在文件开头插入一行
sed '1i\header line' file
```

### 六、grep 的常用技巧

```bash
# 基本搜索
grep "ERROR" app.log

# 忽略大小写
grep -i "error" app.log

# 显示行号
grep -n "ERROR" app.log

# 反向匹配（排除包含 ERROR 的行）
grep -v "DEBUG" app.log

# 正则匹配
grep -E "ERROR|WARN" app.log      # 匹配 ERROR 或 WARN
grep -E "[0-9]{1,3}\.[0-9]{1,3}" file  # 匹配 IP 格式

# 递归搜索目录
grep -r "TODO" src/              # 在 src 目录下所有文件中搜 TODO

# 只显示文件名（不显示匹配内容）
grep -l "ERROR" *.log            # 列出包含 ERROR 的日志文件

# 统计匹配行数
grep -c "ERROR" app.log

# 彩色输出
grep --color "ERROR\|WARN" app.log

# 显示匹配行的前后上下文
grep -B 2 "ERROR" app.log        # 显示匹配行的前 2 行
grep -A 2 "ERROR" app.log        # 显示匹配行的后 2 行
grep -C 2 "ERROR" app.log        # 前后各 2 行
```

## 常见追问与应对

**Q1: awk 和 cut 有什么区别？取列用哪个？**
> `cut` 更简单，适合按单字符分隔取列（如 `cut -d',' -f2`）。`awk` 更强大：支持多字符分隔、条件过滤、计算、格式化输出。简单取列用 `cut` 即可，涉及条件判断或计算用 `awk`。面试中写 `awk` 更加分。

**Q2: sort -n 和 sort 有什么区别？什么时候必须用 -n？**
> 不加 `-n` 按字典序排序：`"100"` 排在 `"20"` 前面（因为字符 `'1'` < `'2'`）。加了 `-n` 按数值排序：`20` 排在 `100` 前面。对数字排序必须加 `-n`，否则结果不对。`uniq -c` 的输出第一列是数字，所以最后 `sort -rn` 要加 `-n`。

**Q3: 如果文件特别大（几十 GB），这些命令还能用吗？**
> `sort` 对大文件会使用外部排序（写临时文件到磁盘），可以用 `-T` 指定临时目录、`-S` 指定内存上限（如 `sort -S 2G`）。`awk` 是流式处理，逐行读取不加载全文件，内存友好。`uniq` 也是流式处理。所以 `awk + sort + uniq` 的管道链天然适合大文件，唯一瓶颈是 `sort` 阶段的磁盘 I/O。

**Q4: 有没有更快的替代方案？**
> 大文件场景可以考虑：1）`awk` 自己做计数（不需要 sort + uniq）：`awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn`，省掉 sort 步骤；2）用 `ripgrep`（`rg`）替代 `grep`，用 Rust 写的，快 10-100 倍；3）如果需要复杂分析，直接用 Python/SQL 更灵活。

## 知识关联

- [Linux 常用命令面试速查](./Linux-常用命令.md)
- [进程、线程、协程的区别与联系](./进程-线程-协程.md)
- [设计一个监控日志分析 Agent](../场景问题/监控日志分析Agent设计.md)
