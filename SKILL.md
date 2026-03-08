---
name: exchcli-link
description: 通过 DavMail + mbsync + mu 打通 Exchange CLI 邮件工作流，提供同步、检索、正文提取、锁文件自愈与本地 SMTP 发送策略。Use when user asks to access Exchange mail from CLI, configure DavMail/mbsync/mu, search or summarize Exchange emails, or build automation around local Maildir workflow.
---

# ExchCLI-Link

将 Exchange 邮箱能力收敛为本地可审计 CLI 流程，默认依赖 `DavMail -> mbsync -> mu`。

## 固定基础设施

- Gateway: DavMail（EWS 转 IMAP/SMTP）
  - IMAP: `127.0.0.1:11143`
  - SMTP: `127.0.0.1:11025`
- Sync: `mbsync`（将远程邮件同步到本地 Maildir）
- Index/Search: `mu`（索引与 JSON 查询）
- 认证：Exchange NTLM（由 DavMail 托管）

## 关键配置路径

- `~/.davmail.properties`
- `~/.mbsyncrc`
- `~/Mail/IWhale/`（Maildir 根目录）

仅在用户明确要求时修改这些配置；修改前先读取原始内容并保留既有字段。

## CLI 原语清单（完整）

以下原语是 `ExchCLI-Link` 的标准能力边界。编排流程时仅使用这些原语，不引入额外隐式步骤。

### A. 环境与依赖原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `health.imap` | 检查 DavMail IMAP 端口连通性 | `nc -z 127.0.0.1 11143` |
| `health.smtp` | 检查 DavMail SMTP 端口连通性 | `nc -z 127.0.0.1 11025` |
| `health.maildir` | 检查 Maildir 根目录存在性 | `test -d ~/Mail/IWhale` |
| `health.mu_db` | 检查 mu 索引库可读 | `mu info` |
| `health.binary_mbsync` | 检查 mbsync 可执行 | `command -v mbsync` |
| `health.binary_mu` | 检查 mu 可执行 | `command -v mu` |
| `health.binary_msmtp` | 检查 msmtp 可执行 | `command -v msmtp` |

### B. 配置检查原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `config.check_davmail` | 校验 DavMail 配置文件存在 | `test -f ~/.davmail.properties` |
| `config.check_mbsync` | 校验 mbsync 配置文件存在 | `test -f ~/.mbsyncrc` |
| `config.show_davmail_target` | 查看 DavMail 关键字段 | `sed -n '/^davmail.url\\|^davmail.imapPort\\|^davmail.smtpPort\\|^davmail.bindAddress/p' ~/.davmail.properties` |
| `config.show_mbsync_target` | 查看 mbsync 关键字段 | `sed -n '/^IMAPAccount\\|^Host\\|^Port\\|^User\\|^Channel/p' ~/.mbsyncrc` |

### C. 同步与索引原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `sync.only` | 执行远程到本地同步 | `mbsync iwhale-sync` |
| `index.refresh` | 重建 mu 索引 | `mu index` |
| `sync.full` | 同步并索引 | `mbsync iwhale-sync && mu index` |
| `sync.unlock` | 清理锁文件 | `rm -f ~/Mail/IWhale/*/.lock ~/Mail/IWhale/*/*/.lock 2>/dev/null` |
| `sync.retry_locked` | 锁冲突后恢复并重试 | `rm -f ~/Mail/IWhale/*/.lock ~/Mail/IWhale/*/*/.lock 2>/dev/null && mbsync iwhale-sync && mu index` |
| `sync.pull_inbox_only` | 仅同步 Inbox（按命名匹配） | `mbsync iwhale-sync -V` |
| `sync.dry_run` | 打印同步过程（调试） | `mbsync -n -V iwhale-sync` |

### D. Maildir 状态原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `maildir.list_folders` | 列出 Maildir 目录 | `ls -1 ~/Mail/IWhale` |
| `maildir.inbox_new_count` | 统计 Inbox/new 条目数 | `ls -A ~/Mail/IWhale/Inbox/new \| wc -l` |
| `maildir.inbox_cur_count` | 统计 Inbox/cur 条目数 | `ls -A ~/Mail/IWhale/Inbox/cur \| wc -l` |
| `maildir.folder_new_count` | 统计任意目录 new 条目 | `ls -A ~/Mail/IWhale/<folder>/new \| wc -l` |
| `maildir.folder_cur_count` | 统计任意目录 cur 条目 | `ls -A ~/Mail/IWhale/<folder>/cur \| wc -l` |
| `maildir.disk_usage` | 查看本地邮件目录容量 | `du -sh ~/Mail/IWhale` |

### E. 检索原语（mu find JSON）

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `mail.search` | 通用检索 | `mu find "<query>" --format=json --limit=<n>` |
| `mail.search_from` | 按发件人检索 | `mu find 'from:<sender> <query>' --format=json --limit=<n>` |
| `mail.search_to` | 按收件人检索 | `mu find 'to:<recipient> <query>' --format=json --limit=<n>` |
| `mail.search_cc` | 按抄送人检索 | `mu find 'cc:<recipient> <query>' --format=json --limit=<n>` |
| `mail.search_subject` | 按主题检索 | `mu find 'subject:"<subject>" <query>' --format=json --limit=<n>` |
| `mail.search_body` | 按正文关键词检索 | `mu find 'body:"<keyword>" <query>' --format=json --limit=<n>` |
| `mail.search_msgid` | 按 Message-ID 检索 | `mu find 'msgid:<message_id>' --format=json --limit=<n>` |
| `mail.search_maildir` | 按目录检索 | `mu find 'maildir:/<folder> <query>' --format=json --limit=<n>` |
| `mail.search_unread` | 检索未读 | `mu find 'flag:unread <query>' --format=json --limit=<n>` |
| `mail.search_flagged` | 检索已标星 | `mu find 'flag:flagged <query>' --format=json --limit=<n>` |
| `mail.search_attachments` | 检索含附件邮件 | `mu find 'flag:attach <query>' --format=json --limit=<n>` |
| `mail.search_date_range` | 时间范围检索 | `mu find 'date:<start>..<end> <query>' --format=json --limit=<n>` |
| `mail.search_since` | 起始时间后检索 | `mu find 'date:<start>..now <query>' --format=json --limit=<n>` |
| `mail.search_before` | 截止时间前检索 | `mu find 'date:2000-01-01..<end> <query>' --format=json --limit=<n>` |
| `mail.search_latest` | 最近邮件（按 date desc） | `mu find '<query>' --sortfield=date --reverse --format=json --limit=<n>` |
| `mail.search_thread` | 按主题近似线程聚合 | `mu find 'subject:\"<subject>\"' --sortfield=date --reverse --format=json --limit=<n>` |

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

### G. 发送原语（msmtp）

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `mail.send_msmtp` | 发送基础文本邮件 | `printf "Subject: <subject>\n\n<body>\n" \| msmtp -h 127.0.0.1 -P 11025 <to>` |
| `mail.send_with_from` | 指定发件人发送 | `printf "From: <from>\nTo: <to>\nSubject: <subject>\n\n<body>\n" \| msmtp -h 127.0.0.1 -P 11025 -f <from> <to>` |
| `mail.send_with_cc_bcc` | 带抄送/密送发送 | `printf "To: <to>\nCc: <cc>\nBcc: <bcc>\nSubject: <subject>\n\n<body>\n" \| msmtp -h 127.0.0.1 -P 11025 <to> <cc> <bcc>` |
| `mail.send_rfc822` | 发送 RFC822 文件 | `msmtp -h 127.0.0.1 -P 11025 <to> < <mail_file>` |
| `mail.send_check_only` | 发送前校验配置 | `msmtp --serverinfo --host=127.0.0.1 --port=11025` |

### H. 诊断与调试原语

| 原语 ID | 作用 | 命令模板 |
| --- | --- | --- |
| `diag.mu_info` | 查看 mu 数据库信息 | `mu info` |
| `diag.mu_fields` | 查看可搜索字段 | `mu fields` |
| `diag.mu_help_find` | 查看 find 帮助 | `mu find --help` |
| `diag.mbsync_verbose` | 同步详细日志 | `mbsync -V iwhale-sync` |
| `diag.msmtp_serverinfo` | 查看 SMTP server 能力 | `msmtp --serverinfo --host=127.0.0.1 --port=11025` |

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

## 执行工作流

1. 先执行同步与索引。
2. 将自然语言查询转为 `mu` 查询语句，再执行 `mu find --format=json`。
3. 从返回 JSON 提取 `path`，按需执行 `mu view --format=plain`。
4. 对正文做摘要时，优先输出：发件人、主题、时间、待办事项、风险点。

## 查询改写规则（自然语言 -> mu）

- “某人发来的邮件” -> `from:<name_or_email>`
- “主题包含 X” -> `subject:"X"`
- “正文提到 X” -> `body:"X"`
- “最近/今天” -> 与日期过滤组合（如 `date:today..now`）
- 多条件组合时默认使用空格（AND 语义）

示例：

- `查找 Bob 关于合同的邮件` -> `from:bob subject:"合同"`
- `今天提到发票的邮件` -> `date:today..now body:"发票"`

## 故障处理

当 `mbsync` 报错包含 `is locked`：

1. 删除 `~/Mail/IWhale/` 下残留 `.lock` 文件。
2. 重试 `mbsync iwhale-sync`。
3. 成功后执行 `mu index`。

建议命令：

```bash
rm -f ~/Mail/IWhale/**/.lock 2>/dev/null
mbsync iwhale-sync && mu index
```

若 shell 不支持 `**`，改用：

```bash
rm -f ~/Mail/IWhale/*/.lock ~/Mail/IWhale/*/*/.lock 2>/dev/null
```

## 发送能力（进阶）

优先使用本地 SMTP（DavMail `127.0.0.1:11025`）通过 `msmtp` 发送。发送前检查：

- 收件人、主题、正文是否齐全
- 是否需要抄送/密送
- 是否附带附件与路径可读性

未确认时先生成草稿文本与拟执行命令，待用户确认后再发送。

## 输出约定

- 默认返回结构化结果：`status`、`command`、`stdout/stderr 摘要`、`next_action`。
- 涉及多步流程时，显示当前阶段（sync/search/view/send）与失败回退动作。
- 不回显明文凭据，不写入密码到脚本或日志。

## 补充参考

- 查询与命令模板：`references/commands.md`
