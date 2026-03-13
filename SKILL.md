---
name: exchcli-link
description: 通过 DavMail + mbsync + mu + msmtp + himalaya 打通 Exchange CLI 邮件工作流，提供收件同步、索引检索、正文提取、发送、锁文件自愈与故障回退策略。Use when user asks to access Exchange mail from CLI, configure DavMail/mbsync/mu/msmtp/himalaya, run end-to-end mail tests, search/summarize emails, or automate local Maildir workflows.
---

# ExchCLI-Link

将 Exchange 邮箱能力收敛为本地可审计 CLI 流程，当前标准栈为：

`DavMail -> mbsync -> mu -> (msmtp | himalaya)`

其中：

- 收件链路：`DavMail(IMAP) -> mbsync -> Maildir -> mu`
- 发送链路：
  - `himalaya message send`（推荐，Sent 一致性更好）
  - `msmtp`（可选，需要本机认证机制匹配）

补充：`DavMail + himalaya` 组合本身即可实现收发（send + receive）。
- 接收：`himalaya envelope/message` 可直接通过 DavMail IMAP 读取邮件
- 发送：`himalaya message send` 可通过 DavMail SMTP 发送并保存 Sent
- 当你只需要基础收发时，可仅使用 `DavMail + himalaya`；当你需要本地索引检索/批处理时，再叠加 `mbsync + mu`

## 适用场景

- 用户要求：CLI 访问 Exchange 邮件
- 用户要求：同步收件箱、搜索邮件、读取正文、提取链接/邮箱
- 用户要求：自动化邮件摘要或 test-all 联调
- 用户要求：诊断 DavMail/mbsync/mu/msmtp/himalaya 问题

## 名词约定

- 文档中 `mbssync` 视为 `mbsync` 的常见误写，统一按 `mbsync` 执行。
- `SYNC_CHANNEL` 指 `~/.mbsyncrc` 中 `Channel` 名称。
- `MAILDIR_ROOT` 指本地 Maildir 根目录名（例如 `IWhale`）。

## 关键配置路径

- `~/.davmail.properties`
- `~/.mbsyncrc`
- `~/.msmtprc`（可选）
- `~/Mail/<MAILDIR_ROOT>/`（Maildir 根目录）

仅在用户明确要求时修改这些配置；修改前先读取原始内容并保留既有字段。

## 固定基础设施

- Gateway: DavMail（EWS -> IMAP/SMTP）
  - IMAP: `127.0.0.1:11143`
  - SMTP: `127.0.0.1:11025`
- Sync: `mbsync`（远程 -> 本地 Maildir）
- Index/Search: `mu`（索引与 JSON 检索）
- Send:
  - `himalaya`（推荐）
  - `msmtp`（可选）
- 认证边界：
  - Exchange 认证由 DavMail 托管
  - 修改 `~/.msmtprc` 只影响发送，不影响 `mbsync` 收件
- 轻量模式：
  - `DavMail + himalaya` = 可直接收发
  - `DavMail + mbsync + mu` = 本地索引检索与批处理增强

## 标准执行顺序（Agent 必须遵守）

1. 先做健康检查（端口、二进制、配置文件）。
2. 再做同步与索引（`mbsync <SYNC_CHANNEL> && mu index`）。
3. 再做检索与读取（`mu find` / `mu view`）。
4. 最后做发送（默认 `himalaya`，`msmtp` 仅在兼容时使用）。
5. 每步输出结构化结果并给出下一步动作。

## test-all 最小闭环（推荐）

1. `health.imap` + `health.smtp`
2. `config.check_davmail` + `config.check_mbsync`
3. `sync.incremental_and_index`
4. `mail.search_latest`
5. `mail.view_snippet`
6. `mail.send_himalaya`（或用户指定 `msmtp`）
7. 再次 `sync.incremental_and_index` + 定向查询验证发送结果

## CLI 原语清单（完整版）

以下原语是 `ExchCLI-Link` 的标准能力边界。编排流程时仅使用这些原语，不引入额外隐式步骤。

### A. 环境与依赖原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `health.imap` | 检查 DavMail IMAP 端口连通性 | `nc -z 127.0.0.1 11143` |
| `health.smtp` | 检查 DavMail SMTP 端口连通性 | `nc -z 127.0.0.1 11025` |
| `health.maildir` | 检查 Maildir 根目录存在性 | `test -d ~/Mail/<MAILDIR_ROOT>` |
| `health.mu_db` | 检查 mu 索引库可读 | `mu info` |
| `health.binary_mbsync` | 检查 mbsync 可执行 | `command -v mbsync` |
| `health.binary_mu` | 检查 mu 可执行 | `command -v mu` |
| `health.binary_msmtp` | 检查 msmtp 可执行 | `command -v msmtp` |
| `health.binary_himalaya` | 检查 himalaya 可执行 | `command -v himalaya` |

### B. 配置检查原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `config.check_davmail` | 校验 DavMail 配置文件存在 | `test -f ~/.davmail.properties` |
| `config.check_mbsync` | 校验 mbsync 配置文件存在 | `test -f ~/.mbsyncrc` |
| `config.check_msmtp` | 校验 msmtp 配置文件存在 | `test -f ~/.msmtprc` |
| `config.show_davmail_target` | 查看 DavMail 关键字段 | `sed -n '/^davmail.url\\|^davmail.imapPort\\|^davmail.smtpPort\\|^davmail.bindAddress/p' ~/.davmail.properties` |
| `config.show_mbsync_target` | 查看 mbsync 关键字段 | `sed -n '/^IMAPAccount\\|^Host\\|^Port\\|^User\\|^Channel/p' ~/.mbsyncrc` |

### C. 同步与索引原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `sync.incremental` | 执行远程到本地增量同步（推荐） | `mbsync <SYNC_CHANNEL>` |
| `sync.only` | 与 `sync.incremental` 等价（兼容别名） | `mbsync <SYNC_CHANNEL>` |
| `index.refresh` | 重建 mu 索引 | `mu index` |
| `sync.incremental_and_index` | 增量同步并更新索引（推荐） | `mbsync <SYNC_CHANNEL> && mu index` |
| `sync.full` | 与 `sync.incremental_and_index` 等价（兼容别名） | `mbsync <SYNC_CHANNEL> && mu index` |
| `sync.unlock` | 清理锁文件 | `rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null` |
| `sync.retry_locked` | 锁冲突后恢复并重试 | `rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null && mbsync <SYNC_CHANNEL> && mu index` |
| `sync.pull_inbox_only` | 仅同步 Inbox（按命名匹配） | `mbsync <SYNC_CHANNEL> -V` |
| `sync.dry_run` | 打印同步过程（调试） | `mbsync -n -V <SYNC_CHANNEL>` |

### D. Maildir 状态原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `maildir.list_folders` | 列出 Maildir 目录 | `ls -1 ~/Mail/<MAILDIR_ROOT>` |
| `maildir.inbox_new_count` | 统计 Inbox/new 条目数 | `ls -A ~/Mail/<MAILDIR_ROOT>/Inbox/new \| wc -l` |
| `maildir.inbox_cur_count` | 统计 Inbox/cur 条目数 | `ls -A ~/Mail/<MAILDIR_ROOT>/Inbox/cur \| wc -l` |
| `maildir.folder_new_count` | 统计任意目录 new 条目 | `ls -A ~/Mail/<MAILDIR_ROOT>/<folder>/new \| wc -l` |
| `maildir.folder_cur_count` | 统计任意目录 cur 条目 | `ls -A ~/Mail/<MAILDIR_ROOT>/<folder>/cur \| wc -l` |
| `maildir.disk_usage` | 查看本地邮件目录容量 | `du -sh ~/Mail/<MAILDIR_ROOT>` |

### E. 检索原语（mu find JSON）

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `mail.search` | 通用检索 | `mu find "<query>" --format=json -n <n>` |
| `mail.search_from` | 按发件人检索 | `mu find 'from:<sender> <query>' --format=json -n <n>` |
| `mail.search_to` | 按收件人检索 | `mu find 'to:<recipient> <query>' --format=json -n <n>` |
| `mail.search_cc` | 按抄送人检索 | `mu find 'cc:<recipient> <query>' --format=json -n <n>` |
| `mail.search_subject` | 按主题检索 | `mu find 'subject:"<subject>" <query>' --format=json -n <n>` |
| `mail.search_body` | 按正文关键词检索 | `mu find 'body:"<keyword>" <query>' --format=json -n <n>` |
| `mail.search_msgid` | 按 Message-ID 检索 | `mu find 'msgid:<message_id>' --format=json -n <n>` |
| `mail.search_maildir` | 按目录检索 | `mu find 'maildir:/<folder> <query>' --format=json -n <n>` |
| `mail.search_unread` | 检索未读 | `mu find 'flag:unread <query>' --format=json -n <n>` |
| `mail.search_flagged` | 检索已标星 | `mu find 'flag:flagged <query>' --format=json -n <n>` |
| `mail.search_attachments` | 检索含附件邮件 | `mu find 'flag:attach <query>' --format=json -n <n>` |
| `mail.search_date_range` | 时间范围检索 | `mu find 'date:<start>..<end> <query>' --format=json -n <n>` |
| `mail.search_since` | 起始时间后检索 | `mu find 'date:<start>..now <query>' --format=json -n <n>` |
| `mail.search_before` | 截止时间前检索 | `mu find 'date:2000-01-01..<end> <query>' --format=json -n <n>` |
| `mail.search_latest` | 最近邮件（按 date desc） | `mu find '<query>' --sortfield=date --reverse --format=json -n <n>` |
| `mail.search_thread` | 按主题近似线程聚合 | `mu find 'subject:\"<subject>\"' --sortfield=date --reverse --format=json -n <n>` |

### F. 读取与提取原语（mu view）

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `mail.view_plain` | 读取完整纯文本 | `mu view <msg_path> --format=plain` |
| `mail.view_headers` | 读取头部片段 | `mu view <msg_path> --format=plain \| sed -n '1,80p'` |
| `mail.view_snippet` | 读取正文前 200 行 | `mu view <msg_path> --format=plain \| sed -n '1,200p'` |
| `mail.view_tail` | 读取正文末尾片段 | `mu view <msg_path> --format=plain \| tail -n 120` |
| `mail.batch_view_plain` | 批量读取正文 | `for p in <msg_paths>; do mu view "$p" --format=plain; done` |
| `mail.extract_links` | 提取正文 URL | `mu view <msg_path> --format=plain \| sed -n 's#.*\\(https\\?://[^ ]*\\).*#\\1#p'` |
| `mail.extract_emails` | 提取正文邮箱地址 | `mu view <msg_path> --format=plain \| sed -n 's/.*\\([A-Za-z0-9._%+-]\\+@[A-Za-z0-9.-]\\+\\.[A-Za-z]\\{2,\\}\\).*/\\1/p'` |

### G. 发送原语（msmtp / himalaya）

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `mail.send_msmtp` | 发送基础文本邮件 | `printf "Subject: <subject>\n\n<body>\n" \| msmtp -a default <to>` |
| `mail.send_with_from` | 指定发件人发送 | `printf "From: <from>\nTo: <to>\nSubject: <subject>\n\n<body>\n" \| msmtp -a default -f <from> <to>` |
| `mail.send_with_cc_bcc` | 带抄送/密送发送 | `printf "To: <to>\nCc: <cc>\nBcc: <bcc>\nSubject: <subject>\n\n<body>\n" \| msmtp -a default <to> <cc> <bcc>` |
| `mail.send_rfc822` | 发送 RFC822 文件 | `msmtp -a default <to> < <mail_file>` |
| `mail.send_check_only` | 发送前校验配置 | `msmtp -a default --serverinfo` |
| `mail.send_himalaya` | 发送并保存到 Sent（推荐） | `printf "From: <from>\nTo: <to>\nSubject: <subject>\n\n<body>\n" \| himalaya message send` |

### H. 诊断与调试原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `diag.mu_info` | 查看 mu 数据库信息 | `mu info` |
| `diag.mu_fields` | 查看可搜索字段 | `mu fields` |
| `diag.mu_help_find` | 查看 find 帮助 | `mu find --help` |
| `diag.mbsync_verbose` | 同步详细日志 | `mbsync -V <SYNC_CHANNEL>` |
| `diag.msmtp_serverinfo` | 查看 SMTP server 能力 | `msmtp -a default --serverinfo` |

参数约定：

- `<query>`：mu 查询语句
- `<n>`：返回条数上限，默认 `5`
- `<msg_path>`：`mu find` 返回结果中的 `path`
- `<msg_paths>`：多个 `path`，由调用方展开
- `<to>`：收件人邮箱地址
- `<subject>/<body>`：邮件主题与正文
- `<sender>`：发件人邮箱或名称
- `<recipient>`：收件人邮箱或名称
- `<keyword>`：正文关键词
- `<start>/<end>`：日期范围边界（如 `2026-03-01`、`now`）
- `<mail_file>`：完整 RFC822 邮件文件路径
- `<folder>`：Maildir 子目录（如 `Inbox`、`Archive`）
- `<message_id>`：Message-ID 值
- `<from>/<cc>/<bcc>`：发件、抄送、密送地址

## 发送策略（必须清晰执行）

1. 默认优先 `mail.send_himalaya`（Sent 一致性最佳）。
2. 用户明确指定 `msmtp` 时，先做 `mail.send_check_only`。
3. `msmtp` 失败时执行回退：
   - 若报 `support for authentication method NTLM is not compiled in`：改用 `mail.send_himalaya`。
   - 若报 `530 Authentication required`：检查 `msmtp` 认证配置。
   - 若报 `535 Authentication failed`：检查账号/密码或认证机制。
4. 命令约束：
   - `msmtp -P` 是 `--pretend`，不会真正发送，禁止用于发送命令。
   - `--host/--port` 会切换到命令行临时账户，可能覆盖 `~/.msmtprc` 账号参数；默认优先 `-a default`。
4. 回退后重新同步索引并检索发送结果。

## 查询改写规则（自然语言 -> mu）

- “某人发来的邮件” -> `from:<name_or_email>`
- “主题包含 X” -> `subject:"X"`
- “正文提到 X” -> `body:"X"`
- “最近/今天” -> 与日期过滤组合（如 `date:today..now`）
- 多条件组合时默认使用空格（AND 语义）

示例：

- `查找 Bob 关于合同的邮件` -> `from:bob subject:"合同"`
- `今天提到发票的邮件` -> `date:today..now body:"发票"`

## 关键故障处理

### 1) mbsync 锁冲突

当 `mbsync` 报错包含 `is locked`：

1. 删除 `~/Mail/<MAILDIR_ROOT>/` 下残留 `.lock` 文件。
2. 重试 `mbsync <SYNC_CHANNEL>`。
3. 成功后执行 `mu index`。

建议命令：

```bash
rm -f ~/Mail/<MAILDIR_ROOT>/**/.lock 2>/dev/null
mbsync <SYNC_CHANNEL> && mu index
```

若 shell 不支持 `**`，改用：

```bash
rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null
```

### 2) mu 参数不兼容

- `mu find` 统一使用 `-n`，不要使用 `--limit`。
- 若字段名是 `:path` / `:subject`（不同版本 JSON 风格），调用方需兼容解析。

### 3) msmtp 认证失败

- `support for authentication method NTLM is not compiled in`
  - 原因：本机 msmtp 二进制不支持 NTLM
  - 处理：改用 `mail.send_himalaya` 或调整 msmtp 认证机制
- `530 Authentication required`
  - 原因：未提供有效认证
  - 处理：检查 `~/.msmtprc` 认证项
- `535 Authentication failed`
  - 原因：凭据不正确或机制不匹配
  - 处理：核对账号密码、认证方式、服务端支持机制

注意：修改 `~/.msmtprc`（如 `auth ntlm` -> `auth login/plain`）仅影响发送链路，不影响 `mbsync` 收件链路。

## Agent 输出要求

- 默认返回结构化结果：`status`、`stage`、`primitive`、`command`、`summary`、`next_action`
- 涉及多步流程时，必须标注当前阶段：`health|sync|search|view|send|diag`
- 失败时包含 `error` 字段与回退动作
- 不回显明文凭据，不写入密码到脚本或日志

## 补充参考

- 查询与命令模板：`references/commands.md`
