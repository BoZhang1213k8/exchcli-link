# ExchCLI-Link 命令参考（CLI 原语全集）

## A) 环境与依赖

```bash
# health.imap
nc -z 127.0.0.1 11143

# health.smtp
nc -z 127.0.0.1 11025

# health.maildir
test -d ~/Mail/<MAILDIR_ROOT>

# health.mu_db / diag.mu_info
mu info

# health.binary_mbsync
command -v mbsync

# health.binary_mu
command -v mu

# health.binary_msmtp
command -v msmtp
```

## B) 配置检查

```bash
# config.check_davmail
test -f ~/.davmail.properties

# config.check_mbsync
test -f ~/.mbsyncrc

# config.show_davmail_target
sed -n '/^davmail.url\|^davmail.imapPort\|^davmail.smtpPort\|^davmail.bindAddress/p' ~/.davmail.properties

# config.show_mbsync_target
sed -n '/^IMAPAccount\|^Host\|^Port\|^User\|^Channel/p' ~/.mbsyncrc
```

## C) 同步与索引

```bash
# sync.only
mbsync <SYNC_CHANNEL>

# index.refresh
mu index

# sync.full
mbsync <SYNC_CHANNEL> && mu index

# sync.unlock
rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null

# sync.retry_locked
rm -f ~/Mail/<MAILDIR_ROOT>/*/.lock ~/Mail/<MAILDIR_ROOT>/*/*/.lock 2>/dev/null && mbsync <SYNC_CHANNEL> && mu index

# sync.pull_inbox_only (调试观察)
mbsync <SYNC_CHANNEL> -V

# sync.dry_run
mbsync -n -V <SYNC_CHANNEL>
```

## D) Maildir 状态

```bash
# maildir.list_folders
ls -1 ~/Mail/<MAILDIR_ROOT>

# maildir.inbox_new_count
ls -A ~/Mail/<MAILDIR_ROOT>/Inbox/new | wc -l

# maildir.inbox_cur_count
ls -A ~/Mail/<MAILDIR_ROOT>/Inbox/cur | wc -l

# maildir.folder_new_count
ls -A ~/Mail/<MAILDIR_ROOT>/<folder>/new | wc -l

# maildir.folder_cur_count
ls -A ~/Mail/<MAILDIR_ROOT>/<folder>/cur | wc -l

# maildir.disk_usage
du -sh ~/Mail/<MAILDIR_ROOT>
```

## E) 检索（mu find JSON）

```bash
# mail.search
mu find '<query>' --format=json --limit=5

# mail.search_from
mu find 'from:<sender> <query>' --format=json --limit=5

# mail.search_to
mu find 'to:<recipient> <query>' --format=json --limit=5

# mail.search_cc
mu find 'cc:<recipient> <query>' --format=json --limit=5

# mail.search_subject
mu find 'subject:"<subject>" <query>' --format=json --limit=5

# mail.search_body
mu find 'body:"<keyword>" <query>' --format=json --limit=5

# mail.search_msgid
mu find 'msgid:<message_id>' --format=json --limit=5

# mail.search_maildir
mu find 'maildir:/<folder> <query>' --format=json --limit=5

# mail.search_unread
mu find 'flag:unread <query>' --format=json --limit=5

# mail.search_flagged
mu find 'flag:flagged <query>' --format=json --limit=5

# mail.search_attachments
mu find 'flag:attach <query>' --format=json --limit=5

# mail.search_date_range
mu find 'date:<start>..<end> <query>' --format=json --limit=5

# mail.search_since
mu find 'date:<start>..now <query>' --format=json --limit=5

# mail.search_before
mu find 'date:2000-01-01..<end> <query>' --format=json --limit=5

# mail.search_latest
mu find '<query>' --sortfield=date --reverse --format=json --limit=5

# mail.search_thread
mu find 'subject:"<subject>"' --sortfield=date --reverse --format=json --limit=20
```

## F) 读取与提取（mu view）

```bash
# mail.view_plain
mu view <msg_path> --format=plain

# mail.view_headers
mu view <msg_path> --format=plain | sed -n '1,80p'

# mail.view_snippet
mu view <msg_path> --format=plain | sed -n '1,200p'

# mail.view_tail
mu view <msg_path> --format=plain | tail -n 120

# mail.batch_view_plain
for p in <msg_paths>; do
  mu view "$p" --format=plain
done

# mail.extract_links
mu view <msg_path> --format=plain | sed -n 's#.*\(https\?://[^ ]*\).*#\1#p'

# mail.extract_emails
mu view <msg_path> --format=plain | sed -n 's/.*\([A-Za-z0-9._%+-]\+@[A-Za-z0-9.-]\+\.[A-Za-z]\{2,\}\).*/\1/p'
```

## G) 发送（msmtp）

```bash
# mail.send_msmtp
printf "Subject: <subject>\n\n<body>\n" | msmtp -h 127.0.0.1 -P 11025 <to>

# mail.send_with_from
printf "From: <from>\nTo: <to>\nSubject: <subject>\n\n<body>\n" | msmtp -h 127.0.0.1 -P 11025 -f <from> <to>

# mail.send_with_cc_bcc
printf "To: <to>\nCc: <cc>\nBcc: <bcc>\nSubject: <subject>\n\n<body>\n" | msmtp -h 127.0.0.1 -P 11025 <to> <cc> <bcc>

# mail.send_rfc822
msmtp -h 127.0.0.1 -P 11025 <to> < /tmp/mail.eml

# mail.send_check_only / diag.msmtp_serverinfo
msmtp --serverinfo --host=127.0.0.1 --port=11025
```

## H) 诊断与帮助

```bash
# diag.mu_fields
mu fields

# diag.mu_help_find
mu find --help

# diag.mbsync_verbose
mbsync -V <SYNC_CHANNEL>
```

如果本机参数风格不同，先运行对应工具的 `--help`，再按本机版本调整。
