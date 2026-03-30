---
name: temp-email
description: "Manage temporary/disposable email inboxes via TempMail API for E2E testing workflows. Use this skill when the user needs to create temporary email addresses, check inbox contents, read email bodies, or manage disposable mailboxes. Trigger on phrases like: '建立臨時信箱', 'temp mail', 'temp email', '臨時 email', '建一個測試信箱', '查收信件', '收驗證信', 'check inbox', '取得信件內容', '讀取 email', or any context involving disposable/temporary email for testing, signup verification, or E2E test scenarios."
---

# Temp-Email Skill

Manage temporary email inboxes through the TempMail API. This skill is designed for E2E testing workflows where disposable email addresses are needed — signup verification, invitation acceptance, password reset flows, etc.

## Prerequisites

Two environment variables are required:

| Variable | Purpose |
|---|---|
| `TempMail_Jwt` | Bearer token for API authentication |
| `TempMail_RapidApiKey` | RapidAPI subscription key |

If either variable is missing, stop and inform the user which variable needs to be set.

## API Base

All requests go to `https://tempmail-so.p.rapidapi.com` with these common headers:

```
Authorization: Bearer $TempMail_Jwt
Content-Type: application/json
x-rapidapi-host: tempmail-so.p.rapidapi.com
x-rapidapi-key: $TempMail_RapidApiKey
```

## Operations

### 1. Create Inbox

Create a new temporary email inbox.

```bash
curl --silent --location 'https://tempmail-so.p.rapidapi.com/inboxes' \
  --header "Authorization: Bearer $TempMail_Jwt" \
  --header 'Content-Type: application/json' \
  --header 'x-rapidapi-host: tempmail-so.p.rapidapi.com' \
  --header "x-rapidapi-key: $TempMail_RapidApiKey" \
  --data '{
    "name": "<local-part>",
    "domain": "mailshan.com",
    "lifespan": 300
  }'
```

**Parameters:**
- `name` — local part of the email address (e.g., `e2e-test1` produces `e2e-test1@mailshan.com`)
- `domain` — default `mailshan.com`
- `lifespan` — seconds until inbox expires, default `300`

**Response:**
```json
{
  "data": { "id": "<inbox-id>" },
  "code": 0,
  "message": "Success"
}
```

**After creation**, append the new inbox to `temp-mails.md` in the current working directory. Create the file if it doesn't exist. Use this format:

```markdown
# Temporary Mailboxes

| Email | Inbox ID | Created | Lifespan |
|---|---|---|---|
| e2e-test1@mailshan.com | A58F253F-29C7-7AAB-D95F-A2491535408D | 2026-03-20 14:30:00 | 300s |
```

Each new inbox is appended as a new row. This file serves as the registry of all created inboxes in the current session, so subsequent operations can reference it without the user needing to remember IDs.

### 2. List Emails in Inbox

Retrieve all emails received by a specific inbox.

```bash
curl --silent --location "https://tempmail-so.p.rapidapi.com/inboxes/<inbox-id>/mails" \
  --header "Authorization: Bearer $TempMail_Jwt" \
  --header 'Content-Type: application/json' \
  --header 'x-rapidapi-host: tempmail-so.p.rapidapi.com' \
  --header "x-rapidapi-key: $TempMail_RapidApiKey"
```

**Response:**
```json
{
  "data": [
    {
      "from": "sender@example.com",
      "read": false,
      "id": "<mail-id>",
      "received": 1773986105,
      "forwarded": false,
      "subject": "Subject line"
    }
  ],
  "code": 0
}
```

When the inbox is empty or expired, `data` returns as `{}` (empty object) rather than an empty array. Treat both `{}` and `[]` as "no emails".

Present results as a readable table showing: sender, subject, received time (convert unix timestamp to human-readable), and read status.

### 3. Read Email Content

Fetch the full content of a specific email.

```bash
curl --silent --request GET \
  --url "https://tempmail-so.p.rapidapi.com/inboxes/<inbox-id>/mails/<mail-id>" \
  --header "Authorization: Bearer $TempMail_Jwt" \
  --header 'Content-Type: application/json' \
  --header 'x-rapidapi-host: tempmail-so.p.rapidapi.com' \
  --header "x-rapidapi-key: $TempMail_RapidApiKey"
```

**Response fields:**
- `data.textContent` — plain text version
- `data.htmlContent` — HTML version
- `data.from` — sender address
- `data.subject` — email subject
- `data.received` — unix timestamp

When presenting the email content to the user:
- Show the subject, sender, and received time first
- Display `textContent` for readability
- If the user specifically needs HTML content or needs to extract links/tokens from HTML, use `htmlContent`

## Workflow Patterns

### E2E Test: Signup Verification

A common pattern when used with E2E testing:

1. Create inbox with a descriptive name (e.g., `e2e-signup-{timestamp}`)
2. Use the generated email address in the signup form
3. Poll the inbox for the verification email (retry a few times with short delays — emails typically arrive within 5-30 seconds)
4. Extract the verification link or code from the email content
5. Continue the E2E flow with the extracted data

When polling for emails, use 15 second intervals and retry up to 3 times (45 seconds total) before reporting that no email was received.

### Extracting Links and Codes

When the user needs to extract verification links or codes from emails:
- Parse `htmlContent` for `<a href="...">` tags containing verification/confirmation URLs
- Look for numeric codes (typically 4-8 digits) in `textContent`
- Return the extracted data directly so it can be used in subsequent test steps

**Shell extraction pattern (Windows / Git Bash compatible):**

Do NOT use `python3` (not available on Windows) and do NOT write to `/tmp/` (resolves to `C:\tmp\` which does not exist). Instead, pipe the curl response directly to Node.js via stdin:

```bash
MAIL=$(curl --silent --location "https://tempmail-so.p.rapidapi.com/inboxes/<inbox-id>/mails/<mail-id>" \
  --header "Authorization: Bearer $TempMail_Jwt" \
  --header 'Content-Type: application/json' \
  --header 'x-rapidapi-host: tempmail-so.p.rapidapi.com' \
  --header "x-rapidapi-key: $TempMail_RapidApiKey")

# Extract all href links
echo "$MAIL" | node -e "
  let d='';
  process.stdin.on('data', c => d += c);
  process.stdin.on('end', () => {
    const body = JSON.parse(d);
    const html = body.data.htmlContent;
    const links = [...html.matchAll(/href=\"(https?:\/\/[^\"]+)\"/g)]
      .map(m => m[1].replace(/&amp;/g, '&'));
    links.forEach(l => console.log(l));
  });
"

# Extract 6-digit verification code from textContent
echo "$MAIL" | node -e "
  let d='';
  process.stdin.on('data', c => d += c);
  process.stdin.on('end', () => {
    const body = JSON.parse(d);
    const text = body.data.textContent;
    const match = text.match(/\b(\d{6})\b/);
    if (match) console.log(match[1]);
  });
"
```

## Error Handling

- HTTP 401/403: Token expired or invalid — inform the user to check `TempMail_Jwt`
- HTTP 429: Rate limited — wait and retry
- `code` != 0 in response: API-level error — show the `message` field to the user
- Empty `data` array on list: No emails received yet — suggest polling with delay
