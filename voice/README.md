# Voice Learning Directory

This directory stores draft-vs-edit pairs that teach the system your writing style.

## How it works

When you edit a draft before sending, the system saves the original draft and your
edited version as a YAML file. Over time, these pairs are used as few-shot examples
to make future drafts sound more like you.

## File format

Each file is a YAML document with this structure:

    timestamp: 2026-03-04T09:15:00
    context: "Brief description of what the email was about"
    channel: gmail | whatsapp
    recipient: "Name of the recipient"
    original_draft: |
      The system-generated draft text...
    edited_version: |
      Your edited version of the draft...
    tags: [topic1, topic2, tone-descriptor]

## Bootstrapping

To seed the system with your voice before any edits accumulate, create files
with only `edited_version` (no `original_draft`). Paste sent emails you're
proud of as examples of your natural writing style.

## Selection

When drafting a new response, the system scans this directory for the 3-5 most
relevant pairs based on: same recipient, similar topic/tags, same channel.
