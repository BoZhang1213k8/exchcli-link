# ExchCLI-Link

基于 `DavMail + mbsync + mu + msmtp` 的 Exchange CLI 能力桥接层。  
目标是把 Exchange 邮件访问统一为本地可审计、可编排、可恢复的原语集合，便于 Agent 或自动化脚本调用。

---

## 1. 设计目标

- 将 Exchange EWS 访问下沉为本地 CLI 能力，减少上层复杂度
- 统一同步、检索、读取、发送的命令入口和约定
- 支持锁文件自愈、阶段化执行、结构化结果输出
- 与 OpenClaw Skill 体系兼容，可被 Agent 稳定触发

---

## 2. 架构总览

`ExchCLI-Link` 采用四层链路：

1. **Gateway 层（DavMail）**
   - 将 EWS 暴露为本地 IMAP/SMTP
   - 默认端口：IMAP `11143`，SMTP `11025`
2. **Sync 层（mbsync/isync）**
   - 将远程邮箱同步到本地 Maildir
3. **Index/Search 层（mu）**
   - 对 Maildir 建索引，提供 JSON 检索
4. **Send 层（msmtp）**
   - 通过 DavMail 本地 SMTP 发信

推荐链路：

`Exchange(EWS) -> DavMail(IMAP/SMTP) -> mbsync(Maildir) -> mu(index/search/view) -> Agent`

---

## 3. 目录结构

```text
exchcli-link/
├── SKILL.md
├── README.md
├── _meta.json
└── references/
    └── commands.md
```

本地邮件目录（示例）：

```text
~/Mail/<MAILDIR_ROOT>/
├── Archive/
│   ├── cur
│   ├── new
│   └── tmp
└── Inbox/
    ├── cur
    ├── new
    └── tmp
```

---

## 4. 依赖与前置条件

请先确保本机可用以下工具：

- `davmail`
- `mbsync`（来自 `isync`）
- `mu`
- `msmtp`（可选，发送能力）
- `nc`（网络连通检查）

### 4.1 使用 Homebrew 安装（macOS）

安装核心组件：

```bash
brew install davmail isync mu
```

如果需要发送能力，再安装 `msmtp`：

```bash
brew install msmtp
```

启动 DavMail（默认使用 Homebrew service）：

```bash
# 后台服务（默认方式，推荐）
brew services start davmail
```

仅在调试时使用前台运行：

```bash
davmail
```

升级后重启服务：

```bash
brew services restart davmail
```

查看服务状态与日志：

```bash
brew services list | grep davmail
tail -f ~/Library/Logs/Homebrew/davmail.log
```

版本与路径检查：

```bash
brew list --versions davmail isync mu msmtp
command -v davmail
command -v mbsync
command -v mu
command -v msmtp
```

快速检查：

```bash
command -v davmail
command -v mbsync
command -v mu
command -v msmtp
command -v nc
```

---

## 5. 关键配置

### 5.1 DavMail 配置（`~/.davmail.properties`）

```properties
davmail.url=https://mail.<EXCHANGE_DOMAIN>/EWS/Exchange.asmx
davmail.mode=Base
davmail.imapPort=11143
davmail.smtpPort=11025
davmail.enableEws=true
davmail.bindAddress=127.0.0.1
```

### 5.2 mbsync 配置（`~/.mbsyncrc`）

```ini
IMAPAccount <IMAP_ACCOUNT>
Host 127.0.0.1
Port 11143
User <EXCHANGE_USERNAME>
Pass <EXCHANGE_PASSWORD>
SSLType None
AuthMechs LOGIN

IMAPStore <REMOTE_STORE>
Account <IMAP_ACCOUNT>

MaildirStore <LOCAL_STORE>
Path ~/Mail/<MAILDIR_ROOT>/
Inbox ~/Mail/<MAILDIR_ROOT>/Inbox

Channel <SYNC_CHANNEL>
Far :<REMOTE_STORE>:
Near :<LOCAL_STORE>:
Patterns *
Create Near
SyncState *
```

> 建议：生产场景避免将明文密码长期写入配置，优先使用安全凭据管理方案。

---

## 6. 快速开始

### 步骤 1：安装依赖（Homebrew）

若尚未安装，先执行 `4.1` 中的 Homebrew 安装命令。

### 步骤 2：检查端口与配置

```bash
nc -z 127.0.0.1 11143
nc -z 127.0.0.1 11025
test -f ~/.davmail.properties
test -f ~/.mbsyncrc
test -d ~/Mail/<MAILDIR_ROOT>
```

### 步骤 3：执行增量同步并索引

```bash
mbsync <SYNC_CHANNEL> && mu index
```

说明：`mbsync` 默认是增量同步；文档里旧称 `sync.full` 的动作仅为兼容命名，推荐使用 `sync.incremental` / `sync.incremental_and_index`。

### 步骤 4：JSON 搜索

```bash
mu find 'from:bob subject:"合同"' --format=json --limit=5
```

### 步骤 5：读取正文

```bash
mu view <msg_path> --format=plain
```

### 步骤 6：发送邮件（可选）

```bash
printf "Subject: 测试邮件\n\n这是一封测试邮件。\n" | msmtp -h 127.0.0.1 -P 11025 recipient@example.com
```

---

## 7. CLI 原语能力（完整分层）

完整命令模板请看：`references/commands.md`  
这里给出能力分组概览，便于编排：

- **A. 环境与依赖**
  - 端口探活、二进制存在性、`mu info`
- **B. 配置检查**
  - `~/.davmail.properties`、`~/.mbsyncrc` 存在与关键字段
- **C. 同步与索引**
  - `sync.incremental`、`sync.incremental_and_index`、`index.refresh`、`sync.retry_locked`
- **D. Maildir 状态**
  - 目录列表、`new/cur` 计数、磁盘占用
- **E. 检索（mu find JSON）**
  - `from/to/cc/subject/body/msgid/maildir/flag/date` 等维度
- **F. 读取与提取（mu view）**
  - 正文、头部片段、尾部片段、链接提取、邮箱提取
- **G. 发送（msmtp）**
  - 基础发送、指定发件人、cc/bcc、RFC822 文件发送
- **H. 诊断与调试**
  - `mu fields`、`mu find --help`、`mbsync -V`

---

## 8. 推荐编排工作流

### 8.1 标准“同步-检索-读取-总结”

1. `sync.full`
   （推荐新名：`sync.incremental_and_index`）
2. `mail.search`
3. `mail.view_plain`（或 `mail.view_snippet`）
4. 输出结构化总结（发件人/主题/时间/待办/风险）

### 8.2 锁冲突自愈流程

当 `mbsync` 报 `is locked`：

1. `sync.unlock`
2. `sync.retry_locked`
3. 若仍失败，执行 `diag.mbsync_verbose` 并保留日志

### 8.3 查询改写流程（自然语言 -> mu）

- “某人发来的邮件” -> `from:<sender>`
- “主题包含 X” -> `subject:"X"`
- “正文提到 X” -> `body:"X"`
- “今天/最近” -> `date:today..now` 或显式日期区间

---

## 9. 输出协议建议（给 Agent/脚本）

建议统一输出 JSON：

```json
{
  "status": "ok|error",
  "stage": "health|sync|search|view|send|diag",
  "primitive": "mail.search",
  "command": "mu find 'subject:\"合同\"' --format=json --limit=5",
  "summary": "5 results returned",
  "next_action": "mail.view_plain"
}
```

必要字段建议：

- `status`
- `stage`
- `primitive`
- `command`
- `summary`
- `next_action`
- `error`（失败时）

---

## 10. 故障排查

### 10.1 端口不可达

症状：`nc -z` 失败  
处理：

1. 确认 DavMail 已启动
2. 检查 `~/.davmail.properties` 端口和 `bindAddress`
3. 排查本机防火墙或端口占用

### 10.2 同步失败 / 锁冲突

症状：`mbsync` 报 `is locked`  
处理：

```bash
rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null
mbsync <SYNC_CHANNEL> && mu index
```

### 10.3 `mu find` 无结果

处理顺序：

1. `mu info` 检查索引状态
2. 重跑 `mu index`
3. 放宽查询条件
4. 检查 `maildir:/<folder>` 过滤是否过窄

### 10.4 `msmtp` 发送失败

处理顺序：

1. `msmtp --serverinfo --host=127.0.0.1 --port=11025`
2. 确认 DavMail SMTP 已监听
3. 检查邮件头格式（`From/To/Subject`）

---

## 11. 安全建议

- 不在脚本和日志中回显明文密码
- 不将凭据提交到 git
- 对邮件正文、附件路径、收件人地址做最小暴露
- 发送前先校验收件人和抄送列表，避免误发
- 对自动发送流程增加人工确认开关

---

## 12. 与 Skill 的关系

- `SKILL.md`：给 Agent 的触发与执行规则（偏“策略”）
- `README.md`：给开发者/维护者的完整说明（偏“文档”）
- `references/commands.md`：可复制运行的命令模板（偏“操作手册”）

---

## 13. 版本与后续扩展

当前版本：`0.1.0`（见 `_meta.json`）

建议后续增强：

1. 新增统一入口脚本：`scripts/exchcli_link.py --op <primitive_id>`
2. 固化 JSON 输出协议与错误码
3. 增加 smoke test（health/sync/search/view/send）
4. 增加 CI 校验（命令模板与文档一致性）

