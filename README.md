# 空洞文件断点续传下载器

基于 Linux 文件 I/O 的 C 语言课程设计，在鲁班猫 1（RK3566 / aarch64）上完成编译和测试。实现空洞文件创建、定点写入、文件空间统计以及 HTTP 下载的暂停与续传。

## 文件说明

| 文件 | 用途 |
|---|---|
| `src/sparseget.c` | 文件操作、HTTP 协议处理、断点保存与校验 |
| `src/ui.c`、`src/ui.h` | 终端显示与普通文本输出 |
| `Makefile` | 普通构建、检测构建与测试入口 |
| `tests/test_sparseget.py` | 34 项回归测试 |
| `tests/fault_io.c` | 测试时注入 I/O 错误和进程中断 |
| `tests/demo_server.py` | 回环 HTTP 演示服务器 |
| `tests/demo_resume.py` | 可选自动演示，主动发送 SIGINT，与手动截图实验区分 |
| [AI_COLLABORATION.md](AI_COLLABORATION.md) | 提示与反馈摘要、采纳判断、测试及 README 检查记录 |
| [docs/CODE_REVIEW.md](docs/CODE_REVIEW.md) | 缺陷现象、原因、修复和回归验证 |
| [docs/EXPERIMENTS.md](docs/EXPERIMENTS.md) | 历史板端验证记录与手动运行截图 |

Word 报告通过课程平台单独提交，本仓库提供源码和实验附件。不包含编译产物、演示生成的数据或临时断点文件。

## 编译和测试

在鲁班猫 Linux 终端运行，需要 GCC、GNU Make、Python 3。完整测试还需要 `strace` 和 GCC 的 ASan/UBSan 运行库，建议在支持空洞文件及 `SEEK_DATA` / `SEEK_HOLE` 的 ext4 文件系统上执行。

```sh
git clone https://github.com/mwhmwhyyds-eng/sparse-downloader.git
cd sparse-downloader
make
./sparseget --help
make check
make sanitize
```

普通程序为 `sparseget`，检测程序为 `sparseget-asan`，两者分开构建。`make sanitize` 使用 AddressSanitizer 和 UndefinedBehaviorSanitizer，设置 `detect_leaks=0`，未启用泄漏检测。未安装 `strace` 时第 27 项会显示跳过，不能据此声称完成了同步调用计数验证。`make clean` 只删除两个编译产物。

如果使用 Download ZIP，在解压目录中运行 `make` 即可。以下演示以板端项目路径 `/home/cat/sparse-downloader` 为例；实际解压位置不同时，调整两个终端的 `cd` 路径。

## 空洞创建和定点写入

使用尚不存在的 `hole.bin`：

```sh
cd /home/cat/sparse-downloader
mkdir -p demo/report-manual
cd demo/report-manual
../../sparseget create hole.bin 1M
../../sparseget status hole.bin
../../sparseget write hole.bin 4096 HELLO
../../sparseget status hole.bin
python3 - <<'PY'
from pathlib import Path
actual = Path('hole.bin').read_bytes()
expected = bytearray(1024 * 1024)
expected[4096:4101] = b'HELLO'
assert actual == expected
print('附近字节:', repr(actual[4092:4105]))
print('整文件比对通过：4096 处为 HELLO，其余字节均为 0')
PY
```

`create` 要求大小不小于 128 KiB；`K/M/G` 按 1024 的幂计算。`write` 使用 `lseek + write`，禁止越过逻辑文件末尾。写入 5 字节并不意味着只分配 5 字节磁盘空间，实际占用由文件系统按块分配。`status` 分别显示逻辑长度、以 512 字节为单位的分配块数、实际占用和空洞比例；不支持空洞查询时会明确提示。

第 2 项回归测试另在 900000 偏移写入 `END`，读取整个 1 MiB 文件，与只在这两处修改的预期缓冲区逐字节比较，并检查分配空间仍远小于逻辑长度。

## 手动暂停和续传

在 VS Code 的同一个 SSH 连接中打开两个终端。终端 A 保持服务器运行，终端 B 运行下载器并手动按 Ctrl+C。

### 终端 A 生成 4 MiB 源文件并启动服务器

```sh
cd /home/cat/sparse-downloader
mkdir -p demo/report-manual
python3 - <<'PY'
from pathlib import Path
import hashlib
p = Path('demo/report-manual/source.bin')
data = bytes(range(256)) * (4 * 1024 * 1024 // 256)
if p.exists():
    assert p.read_bytes() == data, '已有源文件内容不同，请使用新的实验目录'
else:
    with p.open('xb') as f:
        f.write(data)
print('bytes=', p.stat().st_size)
print('sha256=', hashlib.sha256(data).hexdigest())
PY
python3 tests/demo_server.py --file demo/report-manual/source.bin --port 18766
```

服务器显示 `Serving http://127.0.0.1:18766/file.bin` 后，保持终端 A 运行。这里明确使用 `--file`；省略该参数时服务器默认数据为 8 MiB，与报告中的 4 MiB 实验不同。

### 终端 B 下载并手动暂停

```sh
cd /home/cat/sparse-downloader/demo/report-manual
../../sparseget download http://127.0.0.1:18766/file.bin download.bin --delay-ms 500
```

进度达到约 20%～60% 时，聚焦终端 B，手动按 **Ctrl+C**。看到“已暂停，断点已保存”及保存偏移后运行：

```sh
../../sparseget status download.bin
```

再执行完全相同的下载命令：

```sh
../../sparseget download http://127.0.0.1:18766/file.bin download.bin --delay-ms 500
sha256sum source.bin download.bin
```

续传起始偏移应等于上次暂停时保存的偏移，完成后的保存偏移为 `4194304`。两行 SHA-256 应一致，本例预期值为：

```text
2b07811057df887086f06a67edc6ebf911de8b6741156e7a2eb1416a4b8b1b2e
```

`--delay-ms` 仅用于放慢演示，正常下载可省略。Ctrl+C 后退出码为 130，属于已处理的暂停，VS Code 命令旁可能显示非零退出标记，应结合程序的保存结果判断。实验完成后可在终端 A 按 Ctrl+C 停止服务器。

### 重复演示

暂停后继续下载时，应保持相同的 URL、目标文件及对应的 `download.bin.resume`，并重新执行原下载命令。服务器也须保持运行。若目标文件已经完整下载，再次运行会检查远端版本和本地前缀，通过后直接显示完成，不再下载正文。

开始新一轮演示时，使用尚不存在的目标文件名（如 `download-round2.bin`）或新的实验目录，保留上一轮结果；此后的暂停和续传都使用新文件名。重做空洞写入实验也应换用新文件名，不要覆盖已有结果。

## 断点机制与范围

累计写入约 1 MiB 时保存断点，并在正常完成、已处理的暂停等退出路径补存剩余进度。提交顺序为数据 `fdatasync`、临时状态写入及 `fsync`、原子 `rename`、父目录 `fsync`。进度条的“已写”与“已存”可以不同，应以暂停或完成摘要中的保存偏移为准。

首份有效断点已经建立后，遭遇 SIGKILL 时可从最近有效断点恢复，尚未提交的后缀会重新下载。测试覆盖的是进程崩溃及 I/O 故障注入，未进行物理断电实验。首次有效状态发布前若遭 SIGKILL，不能保证自动恢复；首次初始化的已处理 I/O 错误会尝试安全回滚，持续文件系统故障可能使回滚也失败，此时明确报错。

协议限于明文 HTTP、HEAD、Content-Length、强 ETag 和单段 Range，不支持 HTTPS、重定向、分块传输及压缩响应。ETag 是远端版本标识，不必是 SHA-256；本地 FNV 校验用于检查已完成前缀是否变化，完整内容使用独立的 SHA-256 比较。

写入和下载只接受单硬链接的普通文件，拒绝符号链接，并通过锁阻止协作进程同时写入。已有目标文件没有有效断点时不会被直接接管或截断。这些措施不保证阻止任意外部程序绕过锁修改文件。

交互终端显示中文摘要与进度条；输出重定向或 `TERM=dumb` 时使用普通文本；设置 `NO_COLOR=1` 可禁用颜色。退出码：0 成功、1 运行错误、2 用法错误、130 暂停。
