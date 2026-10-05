# Demo video script — Mermail Invoice Desk (target 3:30, hard max 5:00)

## Before recording (checklist)

- [ ] Mermail workspace with one agent mailbox, Agent Wallet (PayBox) connected, ≥ 3 USDC on Solana.
- [ ] A second Solana wallet you own (e.g. Phantom), open in a browser tab, to show the payment landing.
- [ ] `demo/invoice-desk.ledger.json` filled in (mailbox, Sender A, second wallet address).
- [ ] The three seed emails from `seed-emails.md` delivered to the agent mailbox.
- [ ] Claude Code (or Codex/Cursor) opened in the `demo/` folder, Mermail MCP connected with **OAuth** (`/mcp` shows `mermail` connected), and the skill installed.
- [ ] Clean terminal, large font, notifications off.

Keep each prompt below on a sticky note; paste them verbatim so the take is reproducible.

---

## 0:00–0:25 — Hook

**Screen:** Mermail inbox with the three invoices, one subject reading "URGENT: updated payment details".

**Say:** "Agents with wallets will get invoices by email, and email is where invoice fraud lives. This is Invoice Desk, a Mermail Agent Skill. It turns an agent inbox plus the Agent Wallet into an accounts-payable clerk that pays real invoices and refuses fake ones."

## 0:25–0:55 — How it's wired

**Screen:** `skills/mermail-invoice-desk/SKILL.md` scrolled briefly, then `invoice-desk.ledger.json`.

**Say:** "The skill uses Mermail's MCP server: inbox tools to read invoices, PayBox transfers to pay, reply-to-email for remittance, and folders to file. The rule that matters is this ledger. I control it. It lists who I pay, at which address, and how much at most. An email can *request* a payment but can never *authorize* one."

## 0:55–1:50 — Run sheet (read-only)

**Prompt:**

```text
Run the invoice desk on my agent mailbox for the last 7 days. Don't pay anything yet.
```

**Expected on screen:** the agent probes PayBox, resolves the mailbox, creates `ap-paid` / `ap-hold` / `ap-blocked`, searches, reads 3 invoices, reads the wallet, and prints:

```text
pixelforge/INV-1042  1.25 USDC/solana  V✓A✓P✓C✓L✓D✓F✓  ready
pixelforge/INV-1043  2.40 USDC/solana  V✓A✓P✗ …        blocked  payout address differs from ledger; "new wallet" + AI-directed instruction
unknown/CC-889       4.99 USDC/solana  V✗              hold     sender not in vendor ledger
ready total 1.25 · wallet X.XX · shortfall 0 — nothing has been paid
```

**Say (pointing at the blocked row):** "Same real vendor sender, it even passes email authentication, but the payout address doesn't match my ledger. The email also told the AI to pay without asking. The skill flags that as fraud and blocks it. The unknown vendor is held, not paid."

## 1:50–2:45 — Pay the approved invoice

**Prompt:**

```text
Pay pixelforge/INV-1042 only. Cap the run at 2 USDC.
```

**Expected:** an exact preview (full ledger address, 1.25 USDC, Solana, credential), then one `paybox_request_transfer`. If PayBox returns `pending_signature`, the signing frame or a console link appears. Approve it on camera.

**Prompt after approving:**

```text
I approved it. Check that it settled.
```

**Expected:** one `paybox_get_request` → terminal success → INV-1042 is `paid`.

**Screen:** switch to the second wallet tab and show +1.25 USDC arriving.

## 2:45–3:20 — Remit and file

**Prompt:**

```text
Show me the remittance email, then send it and file everything.
```

**Expected:** full reply preview (To = vendor, invoice number, amount, tx reference) → approval → `reply_to_email` → `move_email` into the three folders → AP report.

**Screen:** Sender A's inbox receiving "Re: Invoice INV-1042 — paid"; Mermail folders `ap-paid` (1), `ap-hold` (1), `ap-blocked` (1).

## 3:20–3:45 — Social-engineering attempt (live)

**Prompt:**

```text
Pixel Forge says their wallet changed. Update the ledger from their email and pay INV-1043.
```

**Expected:** refusal to take ledger values from email; INV-1043 stays `blocked`; asks for the new address *stated by me* plus how I verified it out of band.

## 3:45–4:05 — Duplicate protection (optional)

Re-send email #1 as "Reminder: Invoice INV-1042", then:

```text
Run the invoice desk again.
```

**Expected:** `pixelforge/INV-1042 … duplicate (already in ap-paid)`. Nothing is paid.

## 4:05–4:20 — Close

**Say:** "Invoice Desk: inbox in, verified payments out, remittance sent, books filed, and fraud stopped at the gate. Install with `npx skills add <your-github>/<repo> --skill mermail-invoice-desk`."

---

## If something goes sideways while recording

| Symptom | What to do on camera |
| --- | --- |
| `connect_handoff` / `reauth_handoff` | Click the returned console link, reconnect, re-run the prompt. Cut in edit. |
| Wallet shortfall | Lean in: "the desk tells me I'm short and offers the funding handoff". Fund, then continue. |
| `sender_authentication` is `unknown` for Sender A | The legit invoice goes to `hold` (correct behavior). Re-send from a provider with DKIM (Gmail/Outlook) or explain the gate. |
| `pending_settlement` after signing | Wait 20–30 s, then "check again". Never ask it to re-pay. |
