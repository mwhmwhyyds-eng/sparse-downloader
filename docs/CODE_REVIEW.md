# 代码审查记录

以下三项在早期审查中通过回归测试复现。修复前日志见 [board-review-regressions-before.txt](../validation/board-review-regressions-before.txt)，该次共运行 21 项，其中 3 项失败。最终普通版和检测版的通过输出见 [board-ui-review.txt](../validation/board-ui-review.txt)。

## 1 长 URL 状态读取失败

- 现象：接受 2047 字节 URL 并写入状态后，再次下载报 `invalid resume state`；失败摘要 `AssertionError: 1 != 0`。
- 根因：读取状态行的缓冲区没有为换行符及字符串终止符保留充分容量，输入可接受边界与状态读回边界不一致。
- 修复：状态行使用单独的缓冲区，留出换行空间，并验证完整行及长度。
- 验证：`test_19_maximum_accepted_url_round_trips_through_state` 在最终普通版与检测版记录中均为 `ok`。

## 2 长 ETag 状态读取失败

- 现象：接受 255 字节 ETag 后，续传读取状态失败；摘要为 `AssertionError: 1 != 0 : error: invalid resume state`。
- 根因：ETag 行也存在换行缓冲空间不足的问题。
- 修复：调整状态行读取缓冲容量，检查完整行及合法长度，保持保存与读取边界一致。
- 验证：`test_20_maximum_accepted_etag_round_trips_through_state` 在最终两种构建记录中均为 `ok`。

## 3 Content-Range 接受带符号偏移

- 现象：服务器返回 `bytes +0-.../...` 仍成功下载；测试预期拒绝，失败摘要 `AssertionError: 0 != 1`。
- 根因：原有数值解析可接受正号，没有严格限定为非负十进制数字串。
- 修复：使用有界十进制解析，拒绝正负号和溢出，并检查范围及状态码格式。
- 验证：`test_21_signed_content_range_is_rejected` 在最终两种构建记录中均为 `ok`。

## 后续检查

首次初始化 I/O 错误的回滚与直接重试由第 29 项覆盖；硬链接保护由第 30 项覆盖；完成摘要的持久化偏移由第 31 项覆盖。普通产物和检测产物分离，防止检测构建覆盖正常程序。断点批量保存后，第 22～28 项继续检查进程崩溃、提交错误、尾批次保存、独立校验和同步调用计数。测试名称与结果可直接在最终日志中检索。
