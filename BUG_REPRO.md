# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

手持终端恢复网络后补传打卡记录失败，请帮我修复。

终端离线期间缓存了三条关键点位打卡，网络恢复后调用补传接口返回 ok，但结果里 clock_ins_synced 一直是 0，任务详情里一条打卡都没有；同时终端本地缓存已经被清空（cleared=true），等于这几条打卡彻底丢了。同一次补传里的离线轨迹点是正常写入的，tracks_synced 数量正确。

期望：补传要把离线缓存的打卡按时间先后全部写入对应任务，重复补传保持幂等、不产生重复记录，轨迹补传行为保持不变。仓库中已有的公开行为测试覆盖了这些场景。修复后请保证 go test ./... 全绿，不要修改或跳过测试。

## 含 Bug 版本

- 仓库：11DingKing/goS-02
- 仓库地址：https://github.com/11DingKing/goS-02.git
- parent SHA：98ffeba9e68fb4f65454c62796e95bf1e7a9bc80

## 复现步骤

```bash
git clone -- https://github.com/11DingKing/goS-02.git bug-repro
cd bug-repro
git checkout --detach 98ffeba9e68fb4f65454c62796e95bf1e7a9bc80
go test ./internal/service -run "^TestOfflineSync" -count=1 -v
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run "^TestOfflineSync" -count=1 -v
=== RUN   TestOfflineSyncReplaysInOrder
    terminal_service_test.go:44: expected 3 synced, got 0
--- FAIL: TestOfflineSyncReplaysInOrder (0.01s)
=== RUN   TestOfflineSyncIdempotency
    terminal_service_test.go:97: first sync: expected 1 ci + 1 track, got ci=0 track=1
--- FAIL: TestOfflineSyncIdempotency (0.00s)
FAIL
FAIL	patrol-platform/internal/service	0.043s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run "^TestOfflineSync" -count=1 -v
=== RUN   TestOfflineSyncReplaysInOrder
    terminal_service_test.go:44: expected 3 synced, got 0
--- FAIL: TestOfflineSyncReplaysInOrder (0.00s)
=== RUN   TestOfflineSyncIdempotency
    terminal_service_test.go:97: first sync: expected 1 ci + 1 track, got ci=0 track=1
--- FAIL: TestOfflineSyncIdempotency (0.00s)
FAIL
FAIL	patrol-platform/internal/service	0.005s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向测试通过：go test ./internal/service -run '^TestOfflineSync' -count=1 -v
全量回归 go test -timeout=120s -count=1 ./... 通过，go build ./... 与 go vet ./... 通过
补传后打卡按时间升序写入、标记为离线补传、重复补传不产生重复记录；不得修改或跳过既有测试
