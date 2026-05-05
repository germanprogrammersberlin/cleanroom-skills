---
name: gmail-connector
description: Send and read e-mails through a Gmail account using OAuth2 + Gmail API. Use when an agent needs to dispatch correspondence (creditors, contracts, official letters) on behalf of a human, with optional Send-As alias for a different "From" address (e.g. forwarding-domain like david.kaiser@gmx.ch routed via a Gmail mailbox).
---

# Gmail Connector

Programmatic Gmail access (read + send + labels + Send-As management) via Google's official Gmail API. Used by Butler-style agents that act as a personal assistant — dispatching correspondence, archiving incoming mail, watching for bounces, etc.

## When to Use

- Dispatching letters/emails on behalf of the user (creditors, support tickets, contracts)
- Reading recent mail to react to incoming requests
- Bulk-labelling or archiving incoming mail (filing creditor letters into folders)
- Verifying delivery status (Mailer-Daemon bounces, "from": mailer-daemon@googlemail.com)
- Tracking outgoing correspondence in a local audit log

## When NOT to Use

- Sending one-off mail from your own dev box → use a normal mail client
- Mass marketing — Gmail's send quota is ~500/day per account and abuse triggers an account lock
- Anything where the user has not explicitly authorized the dispatch (see "Hard rule" below)

## Hard rule — never send without explicit human confirmation

The agent MUST follow this loop on every outgoing message:

1. Show the draft to the user (To, Subject, Body)
2. Ask "Send? Yes/No?"
3. WAIT for explicit "yes/ok/send" — silence is NOT consent
4. Only then call `sende_email(...)`

Skipping step 2 is the most common abuse and the user will lose trust in the agent immediately.

## Setup (one-time)

### 1. Google Cloud project + OAuth credentials

In Google Cloud Console (cloud.google.com), logged in with the Gmail account that should own the connector:

- Create a project (e.g. `gmail-connector-XXXXX`)
- Enable **Gmail API** in the API library
- Configure **OAuth Consent Screen** → External, Testing mode, add the same Gmail address as Test User
- Create credentials → **OAuth client ID** → Application type: **Desktop**
- Download the JSON → save as `gmail_credentials.json` in the project working directory

### 2. Required OAuth scopes

```python
SCOPES = [
    'https://www.googleapis.com/auth/gmail.readonly',         # read
    'https://www.googleapis.com/auth/gmail.modify',           # labels, mark as read
    'https://www.googleapis.com/auth/gmail.send',             # send
    'https://www.googleapis.com/auth/gmail.settings.basic',   # list send-as aliases
    'https://www.googleapis.com/auth/gmail.settings.sharing', # update send-as SMTP credentials
]
```

`gmail.settings.sharing` is needed only if the agent must repair Send-As SMTP passwords programmatically. The other four are the everyday scopes.

### 3. First authentication

Run an `authenticate()` helper that uses `google-auth-oauthlib`'s `InstalledAppFlow.run_local_server(port=0)`. A browser window opens; user logs in with the Gmail account and accepts the scopes. The resulting `gmail_token.json` is the persistent refresh-token store — keep it private.

### 4. Send-As alias (optional, for cross-domain "From")

If you want outgoing mail to appear as `someone@otherdomain.com` while routed through Gmail:

- In Gmail web UI → Settings → Accounts and Import → "Send mail as" → Add another email address
- Enter SMTP server of the other domain (e.g. for GMX: `mail.gmx.com:587`, STARTTLS)
- Confirmation code is mailed to that address — paste it back

**Note:** Gmail validates the SMTP login on send, not on save. If you mistype the password, the alias *appears* configured but every send bounces with `535 Authentication credentials invalid`. To fix, click "Edit info" on the alias and re-enter the password manually (do not trust browser autofill).

### 5. Store the alias' SMTP password in macOS Keyring (or equivalent)

```python
import keyring
keyring.set_password('butler_gmail', 'gmx_alias_smtp', '<the-smtp-password>')
```

Used later to repair Send-As via `users.settings.sendAs.patch` if needed.

## Send pattern

```python
from gmail_integration import sende_email

sende_email(
    an='recipient@example.com',
    betreff='Subject line',
    text="""Body — plain text. Free-flowing paragraphs. Empty line between paragraphs.""",
    von='alias@otherdomain.com',   # optional — uses default account if omitted
    anhang_pfad='/path/to/attachment.pdf',  # optional
)
```

The `sende_email()` wrapper:

- Creates a MIME message (multipart if attachment)
- Adds `Bcc:` automatically to a configured `BCC_ADRESSE` for audit trail
- Base64-url-encodes and dispatches via `users.messages.send`
- Logs to a sync log file
- Tracks correspondence in a local JSON ledger (recipient, subject, timestamp)

## Read pattern

```python
service = authenticate()
results = service.users().messages().list(
    userId='me',
    q='from:creditor@example.com newer_than:7d',
    maxResults=10
).execute()

for m in results.get('messages', []):
    msg = service.users().messages().get(userId='me', id=m['id'], format='full').execute()
    # msg['payload']['headers'] for From/Subject/Date
    # msg['payload']['body']['data'] or recurse parts for body
```

Search query syntax = standard Gmail search ([reference](https://support.google.com/mail/answer/7190)).

## Bounce / failure detection

After every send, check for delivery failures:

```python
res = service.users().messages().list(
    userId='me',
    q='from:mailer-daemon newer_than:1h',
    maxResults=5
).execute()
```

Common failure modes:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Bounce body says "535 Authentication credentials invalid" | Send-As SMTP password wrong on Gmail side | Re-enter password in Gmail "Edit info" UI, or patch via `users.settings.sendAs.patch` |
| 403 "Missing required scope" | Token issued before scope was added | Delete `gmail_token.json` and re-auth |
| 403 "Account Restricted" | Workspace-managed account with org policy blocking the app | Use a non-Workspace Gmail, or get admin to whitelist the OAuth client |
| 429 / quota exceeded | Sent too many in too short a window | Back off; daily limit is ~500/day for free Gmail |

## Send-As repair via API (no UI clicks)

```python
service.users().settings().sendAs().patch(
    userId='me',
    sendAsEmail='alias@otherdomain.com',
    body={
        'sendAsEmail': 'alias@otherdomain.com',
        'displayName': 'Display Name',
        'replyToAddress': 'alias@otherdomain.com',
        'treatAsAlias': True,
        'smtpMsa': {
            'host': 'mail.otherdomain.com',
            'port': 587,
            'username': 'alias@otherdomain.com',
            'password': keyring.get_password('butler_gmail', 'gmx_alias_smtp'),
            'securityMode': 'starttls',
        },
    },
).execute()
```

Requires the `gmail.settings.sharing` scope (see Setup).

## Direct SMTP fallback (when Send-As is broken)

If Gmail Send-As is unreliable for whatever reason, you can bypass it and dispatch directly through the alias domain's SMTP:

```python
import smtplib
from email.mime.text import MIMEText

msg = MIMEText(text, 'plain', 'utf-8')
msg['From'] = 'alias@otherdomain.com'
msg['To'] = recipient
msg['Subject'] = subject

with smtplib.SMTP('mail.otherdomain.com', 587, timeout=30) as smtp:
    smtp.starttls()
    smtp.login('alias@otherdomain.com', keyring.get_password(...))
    smtp.sendmail('alias@otherdomain.com', [recipient], msg.as_string())
```

Trade-off: the message will *not* appear in the Gmail Sent folder. For an audit trail, BCC yourself.

## Required Python packages

```
google-api-python-client
google-auth-oauthlib
google-auth-httplib2
keyring
```

## Files the agent expects

- `gmail_credentials.json` — OAuth client (downloaded from Cloud Console). Never commit to git.
- `gmail_token.json` — refresh token, generated on first auth. Never commit to git.
- A `BCC_ADRESSE` constant in the integration module — typically the user's primary inbox so every dispatched mail leaves an audit copy.

## Reference implementation

`gmail_integration.py` in the Butler project (`/Applications/XAMPP/xamppfiles/htdocs/butler/`) is the working reference. It implements: 3-stage incoming filter (always-import / project-keywords / known-senders), labelled archiving, attachment extraction, send + Send-As, correspondence tracking. Steal liberally.
