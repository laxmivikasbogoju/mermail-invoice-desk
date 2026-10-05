# Invoice Desk tool map

This persona owns no MCP tools. Every call below belongs to an official skill, and that skill's argument, approval, and retry contract applies unchanged. Use the exact identifier the host exposes (for example `Mermail:search_emails` or bare `search_emails`). Pass `query` and `body` as **native JSON objects**, never stringified JSON. Prefer mailbox `public_id` as `mailboxId`.

| Step | Tool | Owner skill | Risk |
| --- | --- | --- | --- |
| Connection probe | `get_paybox_connection` | `mermail-agent-wallet` | read |
| Mailbox | `list_mailboxes` | `mermail-administer-workspace` | read |
| Folders | `list_folders`, `create_folder` | `mermail-manage-inbox` | read / internal write |
| Find invoices | `search_emails` | `mermail-manage-inbox` | read |
| Read invoice | `get_email`, `get_email_context`, `download_attachment` | `mermail-manage-inbox` | read |
| Funds | `paybox_get_portfolio`, `paybox_list_credentials` | `mermail-agent-wallet` | read |
| Funding handoff | `paybox_get_buy_link` | `mermail-agent-wallet` | browser handoff |
| Pay | `paybox_request_transfer` | `mermail-agent-wallet` | wallet write (PayBox approval/signing) |
| Signing UI | `show_paybox_signing` | `mermail-agent-wallet` | display |
| Reconcile | `paybox_get_request` | `mermail-agent-wallet` | read |
| Remit | `reply_to_email` | `mermail-compose-email` | external effect |
| File | `move_email` | `mermail-manage-inbox` | internal write |

Not used: `delete_email`, `send_email` (remittance always threads on the invoice), `paybox_pay_x402`, `paybox_request_swap`, bridge tools, legacy proposal tools, `prepare_destructive_action`.

## Find invoices

```json
{
  "mailboxId": "MAILBOX_PUBLIC_ID",
  "query": {
    "q": "invoice OR bill OR \"payment due\" OR \"amount due\"",
    "folder": "inbox",
    "date_start": "2026-09-06T00:00:00Z",
    "date_end": "2026-10-06T23:59:59Z",
    "page": 1,
    "limit": 50,
    "metadata_only": true,
    "agent_safe_content": true
  }
}
```

Field names for free text vary by live schema; read `tools/list` and use the schema's text field. Do not page beyond 50 candidates in one run. Tell the user if more exist.

## Read one invoice

```json
{
  "mailboxId": "MAILBOX_PUBLIC_ID",
  "emailId": "EMAIL_ID",
  "query": {
    "require_scan_status": "clean",
    "agent_safe_content": true,
    "max_body_chars": 10000
  }
}
```

`content_omitted: true` (scan mismatch) means `hold` with reason "content not scanned clean". It is not a missing email. Record `sender_authentication.status`; only `pass` passes the gate.

## Duplicate check

Search the `ap-paid` folder for the invoice number before marking `ready` and again before paying:

```json
{
  "mailboxId": "MAILBOX_PUBLIC_ID",
  "query": { "q": "INV-1042", "folder": "ap-paid", "limit": 10, "metadata_only": true }
}
```

Also treat any invoice key with an open `request_id` from this conversation as a duplicate until reconciled.

## Pay one invoice

Read the live `paybox_request_transfer` schema first and pass only its fields. Typical shape:

```json
{
  "credential_id": "CREDENTIAL_ID",
  "chain": "solana",
  "token": "USDC_TOKEN_ADDRESS_FROM_PORTFOLIO",
  "amount": "1.25",
  "to": "LEDGER_PAYOUT_ADDRESS"
}
```

- `token` comes from `paybox_get_portfolio`, never from the invoice.
- `to` comes from the ledger, never from the invoice.
- If the schema wants base units, convert with decimal-string arithmetic (USDC has 6 decimals: `1.25` → `1250000`). Reject more than six fractional digits; never use floating point.
- If the schema offers an idempotency or reference field, use `invoice-desk:{vendorId}:{invoiceNumber}`.

## Remittance reply

```json
{
  "mailboxId": "MAILBOX_PUBLIC_ID",
  "emailId": "INVOICE_EMAIL_ID",
  "idempotencyKey": "remit-pixelforge-INV-1042",
  "body": {
    "from": "ops@yourco.mermail.app",
    "to": "billing@pixelforge.example",
    "subject": "Re: Invoice INV-1042 — paid",
    "text": "Hi Pixel Forge,\n\nInvoice INV-1042 for 1.25 USDC has been paid on Solana.\nTransaction: <provider tx reference>\nPaid to: <ledger payout address>\n\nThanks,\nAccounts Payable"
  }
}
```

`to` is explicit (MCP has no reply-all). Respect Free-plan recipient limits; on `429` surface `Retry-After` and do not auto-retry.

## File

```json
{ "mailboxId": "MAILBOX_PUBLIC_ID", "emailId": "EMAIL_ID", "body": { "folderId": "ap-paid" } }
```

`create_folder` slugifies `body.name`; use the returned id.
