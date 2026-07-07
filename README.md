# Linkedin-Post-Generation
# AI-Powered LinkedIn Post Generation Agent

**Platform:** n8n (Cloud)
**Trigger:** Google Sheets (new row added)
**AI Model:** NVIDIA `llama-3.3-nemotron-super-49b-v1`
**Human-in-the-loop:** Gmail approval + revision workflow
**Publishing:** LinkedIn API (native post)

## Overview

This is an autonomous content pipeline that turns a single topic idea into a published LinkedIn post — with a human approval step in between, so nothing goes live without a review.

You drop a topic into a Google Sheet. The agent writes a first draft, rewrites it to sound more natural, emails it to you for approval, and — once you approve — publishes it directly to LinkedIn. If you're not happy with the draft, you can request a revision by email instead of touching the sheet or the workflow again.

## How It Works (Step by Step)

**1. Trigger — Google Sheets**
A Google Sheet has a `Topic` column. Adding a new row with a topic (e.g. "Why AI agents matter for solo founders") fires the workflow within about a minute. The trigger is configured to fire only on **new rows** (not edits), and only reads the `Topic` column.

**2. Draft Generation — AI Agent #1**
An AI agent (backed by NVIDIA's Llama 3.3 Nemotron model) takes the topic and writes a first-pass LinkedIn post: engaging tone, relevant hashtags, under 1300 characters. The prompt explicitly forbids meta-commentary — the model must output only the post text itself, no headers or explanations.

**3. Humanizing Pass — AI Agent #2**
A second AI agent rewrites the draft to sound more natural and conversational, softening anything that reads as too "AI-generated" while keeping the key message intact.

**4. Cleanup**
A small code step trims whitespace and strips any stray quote characters the model might have added, so the final text is publish-ready.

**5. Approval Email — Gmail**
You receive an email with the drafted post and two buttons:
- **✅ Approve & Publish**
- **✏️ Request Revision**

The workflow pauses here and waits (up to 60 minutes) for your response.

**6a. If Approved:**
The post is published directly to your LinkedIn profile via the LinkedIn API, and you get a confirmation email with a link/URN to the live post.

**6b. If Revision Requested:**
You're sent a second email asking for free-text feedback (e.g. "make it shorter" or "add a question at the end"). A third AI agent revises the post based on your feedback, and you're sent a new approval email — looping back to step 5. This can repeat as many times as needed until you approve.

## Tech Stack

| Component | Tool |
|---|---|
| Orchestration | n8n (cloud) |
| Trigger | Google Sheets Trigger node |
| LLM | NVIDIA NIM API (Llama 3.3 Nemotron 49B) |
| Approval / Notifications | Gmail (`sendAndWait` nodes) |
| Publishing | LinkedIn API (OAuth2, OpenID Connect scopes) |
| Logic | n8n Set, Code (JavaScript), and If nodes |

## Setup Requirements

To reuse this workflow, you'll need:

1. **A Google Sheet** with a column named exactly `Topic`. New rows trigger the workflow.
2. **Google Sheets OAuth2 credential** in n8n (read access to the sheet).
3. **An NVIDIA API key** (from [build.nvidia.com](https://build.nvidia.com)) for the LLM calls.
4. **Gmail OAuth2 credential** in n8n for sending approval/notification emails.
5. **A LinkedIn Developer App** with:
   - "Sign In with LinkedIn using OpenID Connect" and "Share on LinkedIn" products enabled (both auto-enabled by default on new apps)
   - Scopes: `openid`, `profile`, `w_member_social`
   - Your **Person URN** (`urn:li:person:<id>`), obtained via the `/v2/userinfo` endpoint after OAuth, hardcoded into the two publish nodes

## Notes & Known Limitations

- **Person URN is hardcoded**, not fetched dynamically — LinkedIn's API restricts programmatic profile lookups for most developer apps, so this has to be grabbed once manually and pasted into the workflow.
- **Manual "Execute Workflow" testing** in the n8n editor will reprocess *every* row in the sheet (not just new ones) — this is standard n8n behavior for polling triggers in test mode, not a bug. Real automatic runs (while the workflow is Active) only process genuinely new rows.
- Approval and revision emails time out after 60 and 30 minutes respectively if left unanswered.
- Only one topic should be added per row/per test to avoid duplicate content generation.

## Example Flow

```
Google Sheet row added: "Topic: Why small teams should automate first"
        ↓
AI drafts post → AI humanizes post → cleanup
        ↓
Email: "📝 LinkedIn Post Draft: [text] — Approve & Publish / Request Revision"
        ↓
   [Approve]                          [Request Revision]
        ↓                                     ↓
Published to LinkedIn              Email asks for feedback text
        ↓                                     ↓
Confirmation email sent            AI revises post → new approval email (loop)
```
