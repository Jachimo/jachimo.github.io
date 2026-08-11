---
layout: default
rfcdate: Sat Mar 28 13:49:42 2026 -0400
date: 2026-03-28
title: Apple iMessage Databases on macOS and iOS
---

# Apple iMessage Databases on macOS and iOS

Version 1.0, compiled 2026-03-27  
Covers macOS Tahoe / iOS 26.x and earlier  

**Reverse Engineering Note:** Apple does not publish official
documentation for the `chat.db`/`sms.db` schema; virtually all
information in this report was drawn from public reverse engineering
documentation and open source tool implementations.

**Note:** This report was compiled in part by automation based on
curated sample data, including GitHub repository files, secondary web
sources, and both synthetic and real-world sample data.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Table: `message`](#3-table-message)
4. [Table: `chat`](#4-table-chat)
5. [Table: `handle`](#5-table-handle)
6. [Table: `attachment`](#6-table-attachment)
7. [Table: `chat_message_join`](#7-table-chat_message_join)
8. [Table: `chat_handle_join`](#8-table-chat_handle_join)
9. [Table: `message_attachment_join`](#9-table-message_attachment_join)
10. [Attachment Storage Mechanism](#10-attachment-storage-mechanism)
11. [Special Encodings](#11-special-encodings)
12. [iOS `sms.db` vs macOS `chat.db`](#12-ios-smsdb-vs-macos-chatdb)
13. [iCloud Sync Effects](#13-icloud-sync-effects)
14. [Reconstructing a Full Conversation (SQL)](#14-reconstructing-a-full-conversation-sql)
15. [Tool Comparison Table](#15-tool-comparison-table)
16. [Tool Details](#16-tool-details)
17. [Recommended Archival Approach](#17-recommended-archival-approach)
18. [Key Repositories Summary](#18-key-repositories-summary)
19. [Confidence Assessment](#19-confidence-assessment)
20. [Footnotes & Citations](#20-footnotes--citations)

---

## 1. Executive Summary

Apple's Messages application stores all iMessage, SMS, MMS, and (as of
macOS Sequoia/iOS 18+) RCS messages in a SQLite 3 database called
`chat.db` on macOS (`~/Library/Messages/chat.db`) and `sms.db` on iOS
(`/private/var/mobile/Library/SMS/sms.db`). Both databases share a
nearly identical schema consisting of approximately seven primary
tables linked by join tables, with the `message` table alone carrying
94+ columns as of macOS Tahoe 26. 

The primary encoding challenge is the `attributedBody` BLOB column — a
serialized `NSMutableAttributedString` in Apple's proprietary
`typedstream`/`NSKeyedArchiver` format — which since macOS Ventura
(13) has replaced the plain-text `text` column for many message
types. 

An ecosystem of open-source tools exists for extraction, with
[ReagentX/imessage-exporter](https://github.com/ReagentX/imessage-exporter)
(Rust, 5000+ stars) being the most comprehensive and actively
maintained, offering TXT and HTML export with full attachment
embedding and media
conversion. 

[caleb531/imessage-conversation-analyzer](https://github.com/caleb531/imessage-conversation-analyzer)
(Python) provides a high-level analytics and transcript API built on
pandas/duckdb. Archival best practice is HTML with embedded or
co-located attachments for human readability, plus a parallel
JSON/SQLite dump for machine readability.

---

## 2. Architecture Overview

### File Locations

| Platform | Database Path |
|---|---|
| macOS | `~/Library/Messages/chat.db` |
| macOS Attachments dir | `~/Library/Messages/Attachments/` |
| iOS (device, jailbroken) | `/private/var/mobile/Library/SMS/sms.db` |
| iOS (iTunes/Finder backup) | `Library/Application Support/MobileSync/Backup/<UUID>/3d0d7e5fb2ce288813306e4d4636395e047a3d28` |

**Note:** macOS 10.14+ requires *Full Disk Access* for reading
`chat.db`. Grant access in **System Settings → Privacy & Security →
Full Disk Access**.[^1]

### Entity-Relationship ASCII Diagram

```
 ┌──────────────┐        ┌───────────────────────┐        ┌──────────────┐
 │    handle    │        │   chat_handle_join    │        │     chat     │
 │──────────────│        │───────────────────────│        │──────────────│
 │ ROWID (PK)   │◄───────│ handle_id (FK)        │        │ ROWID (PK)   │
 │ id           │        │ chat_id (FK)          │───────►│ guid         │
 │ country      │        └───────────────────────┘        │ style        │
 │ service      │                                         │ chat_ident.  │
 │ uncanonicali.│                                         │ display_name │
 │ person_centr.│                                         │ room_name    │
 └──────┬───────┘                                         │ service_name │
        │                                                 └──────┬───────┘
        │ handle_id                                              │ chat_id
        ▼                                                        ▼
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                           message                                        │
 │──────────────────────────────────────────────────────────────────────────│
 │ ROWID (PK)  guid  text  attributedBody  handle_id  date  date_read       │
 │ date_delivered  is_from_me  service  is_read  is_delivered  is_sent      │
 │ associated_message_guid  associated_message_type  (tapbacks/reactions)   │
 │ reply_to_guid  thread_originator_guid  (threading)                       │
 │ payload_data  balloon_bundle_id  (rich media / app integrations)         │
 │ group_title  cache_roomnames  item_type  (group chat metadata)           │
 │ date_edited  date_retracted  (edit/unsend, macOS Ventura+)               │
 │ expressive_send_style_id  (bubble/screen effects)                        │
 │ ... (94 total columns as of macOS Tahoe 26)                              │
 └──────────────────────────────┬───────────────────────────────────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        │                                               │
        ▼                                               ▼
 ┌──────────────────────┐                 ┌──────────────────────────┐
 │  chat_message_join   │                 │  message_attachment_join │
 │──────────────────────│                 │──────────────────────────│
 │ chat_id (FK→chat)    │                 │ message_id (FK→message)  │
 │ message_id (FK→msg)  │                 │ attachment_id (FK→att.)  │
 └──────────────────────┘                 └────────────┬─────────────┘
                                                       │
                                                       ▼
                                          ┌────────────────────────┐
                                          │      attachment        │
                                          │────────────────────────│
                                          │ ROWID (PK)  guid       │
                                          │ filename  mime_type    │
                                          │ uti  total_bytes       │
                                          │ transfer_state         │
                                          │ is_outgoing is_sticker │
                                          │ created_date           │
                                          │ ... (24 columns)       │
                                          └────────────────────────┘
```

**Cardinality:**
- `handle` ↔ `chat`: many-to-many via `chat_handle_join` (group chats have N handles)
- `chat` ↔ `message`: many-to-many via `chat_message_join`
- `message` ↔ `attachment`: many-to-many via `message_attachment_join`

---

## 3. Table: `message`

**Location:** `chat.db` and `sms.db`  
**Description:** The central table. Each row is one discrete message
unit, including reactions/tapbacks (which are themselves stored as
message rows with `associated_message_guid` set).[^2]

As of macOS Tahoe 26 / iOS 26, the `message` table has **94
columns**. The full schema below is sourced from documentation in
[ReagentX/imessage-exporter](https://github.com/ReagentX/imessage-exporter).[^3]

### Column Listing

| cid | Column Name | Type | Notes |
|-----|-------------|------|-------|
| 0 | `ROWID` | INTEGER (PK) | Auto-increment primary key |
| 1 | `guid` | TEXT (NOT NULL) | Globally unique message ID; used in `associated_message_guid` for reactions |
| 2 | `text` | TEXT | Plain-text body. **May be NULL** on macOS Ventura+ when body is in `attributedBody` |
| 3 | `replace` | INTEGER | Default 0; internal replacement flag |
| 4 | `service_center` | TEXT | SMS service center number (SMS/MMS only) |
| 5 | `handle_id` | INTEGER | FK → `handle.ROWID`. **0 for self-sent messages** |
| 6 | `subject` | TEXT | Message subject (rarely used) |
| 7 | `country` | TEXT | Country code for SMS routing |
| 8 | `attributedBody` | BLOB | Serialized `NSMutableAttributedString` in `typedstream` format. See §11 |
| 9 | `version` | INTEGER | Internal schema version |
| 10 | `type` | INTEGER | 0=incoming, 1=outgoing (some sources), also used for system messages |
| 11 | `service` | TEXT | `"iMessage"`, `"SMS"`, `"MMS"`, `"RCS"` |
| 12 | `account` | TEXT | Destination account (your Apple ID or phone number that received/sent) |
| 13 | `account_guid` | TEXT | GUID of the account |
| 14 | `error` | INTEGER | Error code (0=none) |
| 15 | `date` | INTEGER | **Nanoseconds since 2001-01-01 UTC**. See §11.1 |
| 16 | `date_read` | INTEGER | Same encoding; when the message was read |
| 17 | `date_delivered` | INTEGER | Same encoding; when delivered |
| 18 | `is_delivered` | INTEGER | Boolean flag |
| 19 | `is_finished` | INTEGER | Send completed |
| 20 | `is_emote` | INTEGER | Emote message flag (legacy) |
| 21 | `is_from_me` | INTEGER | **1=sent by you, 0=received**. Key field for direction |
| 22 | `is_empty` | INTEGER | No content |
| 23 | `is_delayed` | INTEGER | Scheduled delivery pending |
| 24 | `is_auto_reply` | INTEGER | Auto-reply (e.g., Do Not Disturb response) |
| 25 | `is_prepared` | INTEGER | Ready-to-send flag |
| 26 | `is_read` | INTEGER | Whether recipient has read the message |
| 27 | `is_system_message` | INTEGER | System-generated notification (e.g., "person left group") |
| 28 | `is_sent` | INTEGER | Successfully sent |
| 29 | `has_dd_results` | INTEGER | Data detector results present |
| 30 | `is_service_message` | INTEGER | Internal service message |
| 31 | `is_forward` | INTEGER | Forwarded message |
| 32 | `was_downgraded` | INTEGER | Was sent as SMS fallback when iMessage failed |
| 33 | `is_archive` | INTEGER | Archive flag |
| 34 | `cache_has_attachments` | INTEGER | 1 if attachments linked via `message_attachment_join` |
| 35 | `cache_roomnames` | TEXT | Group chat room identifier (mirrors `chat.room_name`) |
| 36 | `was_data_detected` | INTEGER | Data detectors applied |
| 37 | `was_deduplicated` | INTEGER | Duplicate detection flag |
| 38 | `is_audio_message` | INTEGER | Audio message flag |
| 39 | `is_played` | INTEGER | Audio message has been played |
| 40 | `date_played` | INTEGER | When audio message was played |
| 41 | `item_type` | INTEGER | Type of group-chat event (0=message, 1=member added, 2=member removed, 3=name change, etc.) |
| 42 | `other_handle` | INTEGER | Secondary handle reference |
| 43 | `group_title` | TEXT | New group name when `item_type=3` (name change event) |
| 44 | `group_action_type` | INTEGER | Group action subtype |
| 45 | `share_status` | INTEGER | Share status |
| 46 | `share_direction` | INTEGER | Share direction |
| 47 | `is_expirable` | INTEGER | Self-destructing message |
| 48 | `expire_state` | INTEGER | Current expiry state |
| 49 | `message_action_type` | INTEGER | Message action classification |
| 50 | `message_source` | INTEGER | Source of message |
| 51 | `associated_message_guid` | TEXT | **GUID of the message this is a reaction to** (tapbacks) |
| 52 | `associated_message_type` | INTEGER | **Tapback type code** (2000–2005, -2000 to -2005 for removals). See §11.3 |
| 53 | `balloon_bundle_id` | TEXT | Bundle ID for app integrations (e.g., `com.apple.PassbookUIKit`) |
| 54 | `payload_data` | BLOB | `NSKeyedArchiver` plist for rich content (URL previews, Apple Pay, app balloons) |
| 55 | `expressive_send_style_id` | TEXT | Bubble/screen effect identifier (e.g., `com.apple.messages.effect.CKConfettiEffect`) |
| 56 | `associated_message_range_location` | INTEGER | Character offset in target message (for multi-part reactions) |
| 57 | `associated_message_range_length` | INTEGER | Character length of reaction target range |
| 58 | `time_expressive_send_played` | INTEGER | Timestamp for expressive play event |
| 59 | `message_summary_info` | BLOB | Summary metadata blob |
| 60 | `ck_sync_state` | INTEGER | CloudKit sync state |
| 61 | `ck_record_id` | TEXT | CloudKit record ID |
| 62 | `ck_record_change_tag` | TEXT | CloudKit change tag |
| 63 | `destination_caller_id` | TEXT | Your phone number / Apple ID that received the message |
| 64 | `is_corrupt` | INTEGER | Corruption flag |
| 65 | `reply_to_guid` | TEXT | GUID of message this is a direct reply to (threaded replies, macOS Monterey+) |
| 66 | `sort_id` | INTEGER | Sort ordering hint |
| 67 | `is_spam` | INTEGER | Spam filter flag |
| 68 | `has_unseen_mention` | INTEGER | Unread @mention flag |
| 69 | `thread_originator_guid` | TEXT | Root message GUID for threaded reply chains |
| 70 | `thread_originator_part` | TEXT | Part index within multi-part originator message |
| 71 | `syndication_ranges` | TEXT | Syndication text ranges |
| 72 | `synced_syndication_ranges` | TEXT | Synced version of above |
| 73 | `was_delivered_quietly` | INTEGER | Focus/DND quiet delivery |
| 74 | `did_notify_recipient` | INTEGER | Notification sent flag |
| 75 | `date_retracted` | INTEGER | When message was unsent (macOS Ventura+) |
| 76 | `date_edited` | INTEGER | When message was last edited (macOS Ventura+) |
| 77 | `was_detonated` | INTEGER | Self-destruct triggered |
| 78 | `part_count` | INTEGER | Number of parts in multi-part message |
| 79 | `is_stewie` | INTEGER | "Stewie" feature flag (internal) |
| 80 | `is_kt_verified` | INTEGER | Key Transparency verified |
| 81 | `is_sos` | INTEGER | Satellite emergency SOS message |
| 82 | `is_critical` | INTEGER | Critical alert flag |
| 83 | `bia_reference_id` | TEXT | Business iMessage reference |
| 84 | `fallback_hash` | TEXT | Hash for fallback/deduplication |
| 85 | `associated_message_emoji` | TEXT | Custom emoji reaction (iOS 18+ "Emoji Tapback") |
| 86 | `is_pending_satellite_send` | INTEGER | Satellite message queued |
| 87 | `needs_relay` | INTEGER | Relay flag |
| 88 | `schedule_type` | INTEGER | Scheduled message type |
| 89 | `schedule_state` | INTEGER | Scheduled message state |
| 90 | `sent_or_received_off_grid` | INTEGER | Satellite/off-grid flag |
| 91 | `date_recovered` | INTEGER | Date of recovery from deletion |
| 92 | `is_time_sensitive` | INTEGER | Time-sensitive notification |
| 93 | `ck_chat_id` | TEXT | CloudKit chat reference |

**Key Indexes (known):**
- Unique index on `guid`
- Index on `handle_id`
- Index on `date`

---

## 4. Table: `chat`

**Description:** One row per conversation thread. Covers both 1:1
chats and group chats. The `style` field distinguishes them.[^4]

| cid | Column Name | Type | Notes |
|-----|-------------|------|-------|
| 0 | `ROWID` | INTEGER (PK) | |
| 1 | `guid` | TEXT (NOT NULL) | Format: `iMessage;-;<phone/email>` (1:1) or `iMessage;+;<group-uuid>` (group) |
| 2 | `style` | INTEGER | **43 = 1:1 iMessage, 45 = group iMessage, 2 = SMS**[^5] |
| 3 | `state` | INTEGER | Internal state |
| 4 | `account_id` | TEXT | Your Apple ID |
| 5 | `properties` | BLOB | Binary plist of chat properties |
| 6 | `chat_identifier` | TEXT | Phone/email for 1:1; system-assigned UUID for groups |
| 7 | `service_name` | TEXT | `"iMessage"`, `"SMS"` |
| 8 | `room_name` | TEXT | **Group chat's XMPP room name** (present only for group chats) |
| 9 | `account_login` | TEXT | Your sending account |
| 10 | `is_archived` | INTEGER | Archived flag |
| 11 | `last_addressed_handle` | TEXT | Most recent recipient handle |
| 12 | `display_name` | TEXT | **User-set group chat name** (NULL for 1:1 or unnamed groups) |
| 13 | `group_id` | TEXT | Group UUID |
| 14 | `is_filtered` | INTEGER | Unknown filter |
| 15 | `successful_query` | INTEGER | Query cache flag |
| 16 | `engram_id` | TEXT | Siri engram identifier |
| 17 | `server_change_token` | TEXT | CloudKit token |
| 18 | `ck_sync_state` | INTEGER | CloudKit sync state |
| 19 | `original_group_id` | TEXT | Original UUID before merge |
| 20 | `last_read_message_timestamp` | INTEGER | Nanoseconds epoch |
| 21 | `cloudkit_record_id` | TEXT | |
| 22 | `last_addressed_sim_id` | TEXT | SIM used for last message |
| 23 | `is_blackholed` | INTEGER | Messages dropped silently |
| 24 | `syndication_date` | INTEGER | |
| 25 | `syndication_type` | INTEGER | |
| 26 | `is_recovered` | INTEGER | Recovered from deletion |
| 27 | `is_deleting_incoming_messages` | INTEGER | |
| 28 | `is_pending_review` | INTEGER | Spam review |

**`style` values:**

| Value | Meaning |
|-------|---------|
| 43 | 1:1 iMessage |
| 45 | iMessage Group Chat |
| 2 | 1:1 SMS |
| 3 | SMS Group (MMS) |

---

## 5. Table: `handle`

**Description:** Represents a unique contact identifier (phone number
or email). Multiple handles can map to the same real person (e.g.,
phone + email for the same Apple ID).[^6]

| cid | Column Name | Type | Notes |
|-----|-------------|------|-------|
| 0 | `ROWID` | INTEGER (PK) | Referenced by `message.handle_id` |
| 1 | `id` | TEXT (NOT NULL) | Phone number (E.164 or local) or email address |
| 2 | `country` | TEXT | ISO country code |
| 3 | `service` | TEXT (NOT NULL) | `"iMessage"`, `"SMS"`, `"Madrid"` (old SMS), etc. |
| 4 | `uncanonicalized_id` | TEXT | Original unformatted identifier |
| 5 | `person_centric_id` | TEXT | Cross-device person identifier for deduplication |

**Notes on deduplication:** The same physical person may have 2–4
handle rows (SMS phone, iMessage phone, iMessage email). Tools like
imessage-exporter handle this via `person_centric_id` or by matching
on `chat_handle_join`.[^7]

---

## 6. Table: `attachment`

**Description:** Metadata for every file attachment. Does not contain
the file itself — only a path and metadata. The file lives on disk at
the `filename` path.[^8]

| cid | Column Name | Type | Notes |
|-----|-------------|------|-------|
| 0 | `ROWID` | INTEGER (PK) | |
| 1 | `guid` | TEXT (NOT NULL) | Unique attachment ID |
| 2 | `created_date` | INTEGER | Nanoseconds epoch |
| 3 | `start_date` | INTEGER | Transfer start time |
| 4 | `filename` | TEXT | **File path** — typically `~/Library/Messages/Attachments/XX/YY/filename.ext` on macOS |
| 5 | `uti` | TEXT | Uniform Type Identifier (e.g., `public.jpeg`, `public.mpeg-4-audio`) |
| 6 | `mime_type` | TEXT | MIME type (e.g., `image/jpeg`, `video/mp4`) |
| 7 | `transfer_state` | INTEGER | 0=none, 1=downloading, 2=downloaded, 5=failed, 8=not needed |
| 8 | `is_outgoing` | INTEGER | 1=sent by you |
| 9 | `user_info` | BLOB | Serialized metadata plist |
| 10 | `transfer_name` | TEXT | Original filename as sent |
| 11 | `total_bytes` | INTEGER | File size in bytes |
| 12 | `is_sticker` | INTEGER | 1=sticker pack item |
| 13 | `sticker_user_info` | BLOB | Sticker metadata |
| 14 | `attribution_info` | BLOB | Source app attribution data |
| 15 | `hide_attachment` | INTEGER | Hidden (e.g., link previews' internal image) |
| 16 | `ck_sync_state` | INTEGER | CloudKit sync state |
| 17 | `ck_server_change_token_blob` | BLOB | CloudKit token |
| 18 | `ck_record_id` | TEXT | CloudKit record ID |
| 19 | `original_guid` | TEXT (NOT NULL) | Original GUID if reassigned |
| 20 | `is_commsafety_sensitive` | INTEGER | Communication Safety flagged (CSAM) |
| 21 | `emoji_image_content_identifier` | TEXT | Genmoji content ID (iOS 18+) |
| 22 | `emoji_image_short_description` | TEXT | Genmoji prompt text |
| 23 | `preview_generation_state` | INTEGER | Preview thumbnail generation state |

---

## 7. Table: `chat_message_join`

**Description:** Many-to-many join between `chat` and
`message`. Allows a message to belong to a chat.[^9]

```sql
CREATE TABLE chat_message_join (
    chat_id    INTEGER REFERENCES chat(ROWID)    ON DELETE CASCADE,
    message_id INTEGER REFERENCES message(ROWID) ON DELETE CASCADE,
    PRIMARY KEY (chat_id, message_id)
);
```

**Index:** Both `chat_id` and `message_id` are individually indexed for lookup performance.

---

## 8. Table: `chat_handle_join`

**Description:** Many-to-many join between `chat` and
`handle`. Defines group membership. For 1:1 chats, there is exactly
one `handle_id` row.[^10]

```sql
CREATE TABLE chat_handle_join (
    chat_id   INTEGER REFERENCES chat(ROWID)   ON DELETE CASCADE,
    handle_id INTEGER REFERENCES handle(ROWID) ON DELETE CASCADE,
    PRIMARY KEY (chat_id, handle_id)
);
```

---

## 9. Table: `message_attachment_join`

**Description:** Many-to-many join between `message` and `attachment`.[^11]

```sql
CREATE TABLE message_attachment_join (
    message_id    INTEGER REFERENCES message(ROWID)    ON DELETE CASCADE,
    attachment_id INTEGER REFERENCES attachment(ROWID) ON DELETE CASCADE,
    PRIMARY KEY (message_id, attachment_id)
);
```

**Note:** A single message can have multiple attachments (e.g., a photo burst or a message with an image + a document). The `cache_has_attachments` flag on `message` is a performance hint but should not be relied on exclusively.

---

## 10. Attachment Storage Mechanism

### On-Disk Layout (macOS)

```
~/Library/Messages/
├── chat.db                    # Main SQLite database
├── chat.db-shm                # WAL shared memory
├── chat.db-wal                # Write-ahead log (uncommitted transactions)
├── NicknameCaches/            # Cached contact photos
└── Attachments/
    ├── 00/
    │   └── 00/
    │       └── <GUID>/
    │           └── image.jpg
    ├── 01/
    │   └── 01/
    │       └── <GUID>/
    │           └── video.mov
    ...
    └── FF/
        └── ...
```

Attachments are stored in a **two-level hex-prefix directory** scheme
(first two hex digits of a hash form the first directory, second two
form the second), then a GUID-named subdirectory containing the actual
file.[^12]

### `attachment.filename` Format

The `filename` column stores a tilde-expanded absolute path on macOS:

```
~/Library/Messages/Attachments/0A/10/B5C6D2E4-FFFF-AAAA-BBBB-123456789ABC/IMG_0042.HEIC
```

When accessed programmatically, expand `~` with `os.path.expanduser()` in Python or `$HOME` in shell.

### Format Notes

- Images are commonly stored as **HEIC** (High Efficiency Image Container), not JPEG. Older attachments may be JPEG/PNG.
- Videos are stored as **MOV** (QuickTime), sometimes H.265/HEVC encoded.
- Audio messages use **CAF** (Core Audio Format) or **AMR**.
- `imessage-exporter` can optionally convert: HEIC→JPEG, MOV→MP4, CAF/AMR→MP4, HEICS→GIF.[^13]

### iOS Backup Attachments

In an iTunes/Finder backup, attachment files are also hash-renamed
(SHA-1 of domain + relative path), located in:

```
~/Library/Application Support/MobileSync/Backup/<UUID>/
```

The `Manifest.db` SQLite file in the backup root maps the hash
filenames to their original paths.[^14]

### iCloud Optimization

When **iCloud Messages** is enabled with "Optimize Storage",
attachments may be **evicted from local disk** and replaced with a
stub. The `attachment.transfer_state` will reflect this (state `8`
means not locally available). The file referenced by `filename` will
not exist on disk until re-downloaded. The `total_bytes` field still
records the original size.[^15]

---

## 11. Special Encodings

### 11.1 The `date` Column — Apple Epoch (Nanoseconds)

All timestamp columns (`date`, `date_read`, `date_delivered`,
`date_edited`, `date_retracted`, etc.) use the **Core Data timestamp
format**: integer nanoseconds since **2001-01-01 00:00:00 UTC** (the
"Cocoa epoch" or "Apple epoch").

**Conversion to Unix timestamp:**

```python
APPLE_EPOCH_OFFSET = 978_307_200  # seconds from Unix epoch to Apple epoch
unix_timestamp = (apple_ns_timestamp / 1_000_000_000) + APPLE_EPOCH_OFFSET
```

**In SQL (SQLite):**

```sql
-- Convert to human-readable UTC
datetime(date / 1000000000 + strftime('%s', '2001-01-01'), 'unixepoch') AS readable_date
```

This exact expression is taken from the SQL queries of
[caleb531/imessage-conversation-analyzer](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/messages.sql#L6).[^16]

**Historical note:** Prior to macOS Sierra (10.12) / iOS 11, the
`date` column stored **seconds** (not nanoseconds) since the same
epoch. Databases with values < ~1.5×10¹² are in seconds; values ≥
~1.5×10¹² are in nanoseconds. This epoch is distinct from the Unix
epoch (1970-01-01) by exactly 978,307,200 seconds.[^17]

### 11.2 The `attributedBody` BLOB — `typedstream` / `NSKeyedArchiver`

**What it is:** A serialized `NSMutableAttributedString` — a string
with attached ranges of formatting, link metadata, mentions, OTP
codes, and more.

**Why it matters:** Starting with approximately macOS Ventura (13.x),
Apple began storing message text **only** in `attributedBody` (with
`text` being NULL) for some message types, particularly messages
received via iMessage with formatting, stickers, or after the schema
migration.[^18]

**Format:**
- Pre-Ventura: Uses the legacy `typedstream` binary format (GNUstep-era Objective-C serialization)
- Ventura+: May use `NSKeyedArchiver` binary plist, which wraps an NSAttributedString

**Decoding approaches:**

| Approach | Language | Fidelity | Notes |
|----------|----------|----------|-------|
| `pytypedstream` PyPI package | Python | High | Used by `caleb531/imessage-conversation-analyzer` via `pyproject.toml` dependency[^19] |
| `crabstep` Rust crate | Rust | Highest | Used internally by `ReagentX/imessage-exporter`[^20] |
| Binary string scan (heuristic) | Python | Low | Used by `my-other-github-account/imessage_tools` — splits on `NSNumber`, `NSString`, `NSDictionary` markers[^21] |
| `plistlib` (Python stdlib) | Python | Medium | Works for `NSKeyedArchiver` wrapper but not inner typedstream |
| Apple ObjC/Swift native | ObjC/Swift | Perfect | Requires macOS, cannot run cross-platform |

**Heuristic Python decode** (from `my-other-github-account/imessage_tools`):

```python
attributed_body = attributed_body.decode('utf-8', errors='replace')
if "NSNumber" in attributed_body:
    attributed_body = str(attributed_body).split("NSNumber")[0]
    if "NSString" in attributed_body:
        attributed_body = str(attributed_body).split("NSString")[1]
        if "NSDictionary" in attributed_body:
            attributed_body = str(attributed_body).split("NSDictionary")[0]
            attributed_body = attributed_body[6:-12]  # strip framing bytes
```

This is fragile but works for plain-text extraction in many cases.[^22]

**Inner structure of a decoded `attributedBody`:**

```
NSMutableAttributedString {
    string: "Hello @Alice, check this out!",
    attributes: [
        (range: 6-11, NSPersonNameComponentsMentionAttribute: <Alice's handle>),
        (range: 13-28, NSLinkAttributeName: "https://example.com")
    ]
}
```

### 11.3 Reactions / Tapbacks

Tapbacks are stored as **separate `message` rows** with
`associated_message_guid` pointing to the original message's
`guid`. They are identified by `associated_message_type` being in the
range 2000–2005 (received) or -2000 to -2005 (removed).[^23]

**`associated_message_type` values:**

| Code | Tapback | Remove Code |
|------|---------|-------------|
| 2000 | ❤️ Loved | -2000 |
| 2001 | 👍 Liked | -2001 |
| 2002 | 👎 Disliked | -2002 |
| 2003 | 😂 Laughed | -2003 |
| 2004 | ‼️ Emphasized | -2004 |
| 2005 | ❓ Questioned | -2005 |

**iOS 18 / macOS Sequoia+:** Emoji reactions (any emoji) are stored in
`associated_message_emoji` (column 85) with `associated_message_type`
= 2006 for the custom emoji reaction.[^24]

**SQL to find all tapbacks:**

```sql
SELECT
    m.guid,
    m.associated_message_guid,
    m.associated_message_type,
    m.is_from_me,
    h.id AS sender,
    datetime(m.date / 1000000000 + strftime('%s', '2001-01-01'), 'unixepoch') AS sent_at
FROM message m
LEFT JOIN handle h ON m.handle_id = h.ROWID
WHERE m.associated_message_type BETWEEN 2000 AND 2006;
```

### 11.4 `payload_data` — NSKeyedArchiver Rich Content

The `payload_data` BLOB is an `NSKeyedArchiver`-encoded binary
plist. It stores structured data for:

- **URL Previews**: cached title, description, image, canonical URL
- **Apple Pay**: transaction amount, currency, direction
- **App Integrations** (`balloon_bundle_id`): Apple Fitness, SharePlay, Check In, Find My, Polls
- **Apple Music**: album art, preview stream URL, lyrics
- **Apple Maps**: `Placemark` data

Decoding requires either native ObjC/Swift or `crabstep` (Rust). The
imessage-exporter project is the only cross-platform open-source tool
currently known to fully decode this.[^25]

### 11.5 Threaded Replies

Threaded replies (iMessage reply feature, introduced macOS Monterey):

- `reply_to_guid`: direct parent message GUID
- `thread_originator_guid`: the root of the thread chain
- `thread_originator_part`: part index for multi-part root messages

---

## 12. iOS `sms.db` vs macOS `chat.db`

The schemas are **nearly identical**. Key differences:

| Aspect | macOS `chat.db` | iOS `sms.db` |
|--------|-----------------|--------------|
| File path | `~/Library/Messages/chat.db` | `/private/var/mobile/Library/SMS/sms.db` |
| Access method | Full Disk Access on macOS | Jailbreak or backup extraction |
| Attachment root | `~/Library/Messages/Attachments/` | `/private/var/mobile/Library/SMS/Attachments/` |
| Schema version | Tends to be more recent | May lag by a minor version |
| Additional tables | `sync_deleted_messages`, `sync_deleted_chats`, `sync_deleted_attachments` | Same |
| `is_from_me` | Consistent | Consistent |
| iCloud sync | Bidirectionally synced | Source device |

**imessage-exporter explicitly notes:** "Jailbroken iOS filesystem
data uses `sms.db`, which follows the same schema as macOS
`chat.db`. Resolved as a macOS database with an alternate attachment
root."[^26]

**Tables present in both databases:**
- `message`, `chat`, `handle`, `attachment`
- `chat_message_join`, `chat_handle_join`, `message_attachment_join`
- `deleted_messages`, `kvtable`, `_SqliteDatabaseProperties`
- `sync_deleted_messages`, `sync_deleted_chats`, `sync_deleted_attachments`

**iOS backup `sms.db` hash:** In an iTunes/Finder backup, `sms.db` is
stored under the SHA-1 hash filename
`3d0d7e5fb2ce288813306e4d4636395e047a3d28`.[^27]

---

## 13. iCloud Sync Effects

When **Messages in iCloud** is enabled (Settings → [your Apple ID] →
iCloud → Messages):

1. **All devices share one canonical message history.** Deletions on
   one device propagate to all.
2. **The macOS `chat.db` becomes authoritative** — it can contain
   messages from all your Apple devices.
3. **Storage optimization:** Attachments may be locally evicted
   (`transfer_state = 8`). The database row remains but `filename`
   points to a non-existent local file.
4. **`ck_sync_state` and `ck_record_id`** columns track CloudKit
   synchronization status.
5. **Forensic implication:** After deletion with iCloud sync on, the
   message is removed from all devices and backups made after the
   fact. Pre-deletion backups are the only recovery path.[^28]
6. **WAL file:** The `chat.db-wal` (write-ahead log) may contain
   uncommitted or recently committed data not yet checkpointed into
   the main database file. This is relevant for forensic acquisition —
   always capture `chat.db`, `chat.db-shm`, and `chat.db-wal`
   together.[^29]

---

## 14. Reconstructing a Full Conversation (SQL)

The following query reconstructs a complete conversation thread with
sender information, timestamps, and identifies reactions. This is the
pattern used by tools like `caleb531/imessage-conversation-analyzer`.[^30]

```sql
-- Get all messages in a conversation, with sender identity and human-readable timestamps
SELECT
    m.ROWID,
    m.guid,
    -- Prefer text; fall back to note that attributedBody needs decoding
    COALESCE(m.text, '(attributedBody - requires decoding)') AS body,
    datetime(m.date / 1000000000 + strftime('%s', '2001-01-01'), 'unixepoch') AS sent_at,
    CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE h.id END AS sender,
    m.service,
    m.is_from_me,
    -- Identify tapback reactions
    CASE
        WHEN m.associated_message_type BETWEEN 2000 AND 2005 THEN 'TAPBACK'
        WHEN m.associated_message_type BETWEEN -2005 AND -2000 THEN 'TAPBACK_REMOVED'
        ELSE 'MESSAGE'
    END AS message_kind,
    m.associated_message_guid,
    m.associated_message_type,
    m.cache_has_attachments
FROM message m
LEFT JOIN handle h ON m.handle_id = h.ROWID
WHERE m.ROWID IN (
    SELECT message_id
    FROM chat_message_join
    WHERE chat_id = (
        -- Replace with actual chat_identifier, e.g. '+15551234567' or group UUID
        SELECT ROWID FROM chat WHERE chat_identifier = '<TARGET_IDENTIFIER>' LIMIT 1
    )
)
ORDER BY m.date ASC;
```

To also get attachments for each message:

```sql
-- Attachments query (from caleb531/imessage-conversation-analyzer/ica/queries/attachments.sql)
SELECT
    a.ROWID,
    a.mime_type,
    a.filename,
    maj.message_id,
    datetime(m.date / 1000000000 + strftime('%s', '2001-01-01'), 'unixepoch') AS datetime,
    m.is_from_me,
    h.id AS sender_handle
FROM attachment a
INNER JOIN message_attachment_join maj ON a.ROWID = maj.attachment_id
INNER JOIN message m ON m.ROWID = maj.message_id
LEFT JOIN handle h ON m.handle_id = h.ROWID
WHERE maj.message_id IN (
    SELECT message_id
    FROM chat_message_join
    WHERE chat_id IN (/* your chat_ids */)
);
```

---

## 15. Tool Comparison Table

| Tool | Language | Stars¹ | Last Active² | Output Formats | attributedBody | Attachments | Cross-platform | Active |
|------|----------|--------|--------------|----------------|----------------|-------------|----------------|--------|
| [ReagentX/imessage-exporter](https://github.com/ReagentX/imessage-exporter) | Rust | ~5 017 | Mar 2026 | TXT, HTML | ✅ Full (`crabstep`) | ✅ Copy + convert | ✅ (macOS, Linux, Win) | ✅ Very active |
| [caleb531/imessage-conversation-analyzer](https://github.com/caleb531/imessage-conversation-analyzer) | Python | ~33 | Jan 2026 | CSV, XLSX, JSON, Markdown, table | ✅ (`pytypedstream`) | ✅ Metadata only | ❌ macOS only | ✅ Active |
| [kadin2048/ichat_to_eml](https://github.com/kadin2048/ichat_to_eml) | Python | N/A | 2021 | EML, HTML | ✅ (`python-typedstream`, `ccl-bplist`) | ✅ Embedded attachments (NSFileWrapper) | ✅ Cross-platform (Python) | ❌ Low activity |
| [my-other-github-account/imessage_tools](https://github.com/my-other-github-account/imessage_tools) | Python | ~124 | Mar 2026 | dict/print | ⚠️ Heuristic | ❌ | ❌ macOS only | ✅ |
| [niftycode/imessage_reader](https://github.com/niftycode/imessage_reader) | Python | ~116 | Mar 2026 | Terminal, Excel, SQLite | ❌ | ❌ | ⚠️ Linux possible | ✅ |
| [LangChain IMessageChatLoader](https://docs.langchain.com/oss/python/integrations/chat_loaders/imessage) | Python | N/A | 2025 | ChatSession objects | ⚠️ Known issues[^31] | ❌ | ❌ macOS only | ✅ |
| [johnlarkin1/imessage-schema](https://github.com/johnlarkin1/imessage-schema) | SQL | N/A | 2024 | Schema reference only | N/A | N/A | N/A | ❌ |
| [imessagedb (PyPI)](https://imessagedb.readthedocs.io/) | Python | N/A | ~2022 | Python API | ❌ | ❌ | ❌ macOS only | ❌ |

¹ Star counts as of March 2026 via GitHub API.  
² Last commit / push date as of research date.

---

## 16. Tool Details

### 16.1 ReagentX/imessage-exporter

**Repository:** [https://github.com/ReagentX/imessage-exporter](https://github.com/ReagentX/imessage-exporter)  
**Language:** Rust  
**Stars:** ~5,017 (most starred iMessage tool on GitHub)  
**License:** GPL-3.0

**Architecture:** Two crates:
- `imessage-database`: library crate providing Rust models of all iMessage tables and a `typedstream`/`NSKeyedArchiver` decoder (uses the `crabstep` and `crabapple` sub-crates)
- `imessage-exporter`: binary crate providing CLI with `--format txt|html`, `--db-path`, `--attachment-manager`, `--contact-filter`, `--start-date`, `--end-date`

**Supported sources:**
- Local macOS `chat.db`
- Unencrypted iOS backups (resolved normally)
- Encrypted iOS backups (via `crabapple`)
- Jailbroken iOS `sms.db`

**Supported features as of macOS Tahoe 26.4:** iMessage, RCS, SMS,
MMS, multi-part messages, replies/threads, formatted text (mentions,
hyperlinks, OTP/2FA, unit conversions), expressives (bubble+screen),
tapbacks, stickers, Apple Pay, URL previews, App Integrations
(Fitness, SharePlay, Check In, Find My, Polls), handwritten messages
(SVG), Digital Touch, edited/unsent messages, satellite messages,
genmoji.[^32]

**Attachment conversion options:**
- HEIC → JPEG
- HEICS → GIF (animated sticker sequences)
- MOV → MP4
- CAF/AMR → MP4

**Installation:**
```bash
cargo install imessage-exporter
# or
brew install imessage-exporter
```

**Usage example:**
```bash
imessage-exporter --format html --db-path ~/Library/Messages/chat.db \
    --attachment-manager full --output ./export/
```

### 16.2 caleb531/imessage-conversation-analyzer

**Repository:** [https://github.com/caleb531/imessage-conversation-analyzer](https://github.com/caleb531/imessage-conversation-analyzer)  
**Language:** Python 3.9+  
**Stars:** ~33  
**License:** MIT  
**Version:** 3.1.0  
**Package:** `pip install imessage-conversation-analyzer` or `uv tool install imessage-conversation-analyzer`

**Key dependencies** (from `pyproject.toml`):[^33]
- `pandas` — dataframe operations
- `pyarrow` — efficient columnar I/O
- `tabulate` — pretty-print tables
- `openpyxl` — Excel output
- `pytypedstream` — `attributedBody` decoding
- `phonenumbers` — phone number normalization
- `tzlocal` — local timezone detection
- `emoji` — emoji analytics
- `duckdb` — in-memory SQL on dataframes

**Core SQL queries** (in `ica/queries/`):
- `messages.sql` — joins `message`, `handle`, `chat_message_join`; converts date epoch; selects `attributedBody`[^34]
- `attachments.sql` — joins `attachment`, `message_attachment_join`, `message`, `handle`[^35]
- `contact.sql` — resolves contacts from macOS Contacts database

**Built-in analyzers:**
1. `message_totals` — counts by person, reactions, days messaged
2. `attachment_totals` — by MIME type, including Spotify/YouTube/Apple Music link detection
3. `most_frequent_emojis` — top-10 emoji frequency
4. `totals_by_day` — per-day breakdown
5. `transcript` — full printable transcript
6. `count_phrases` — case-insensitive/regex phrase counting
7. `from_sql` — arbitrary SQL against an in-memory DuckDB populated with the conversation dataframes

**Output formats:** `csv`, `excel`/`xlsx`, `markdown`/`md`, `json`, default table

### 16.3 Kadin2048 / ichat_to_eml (upstream)

**Repository:** [https://github.com/kadin2048/ichat_to_eml](https://github.com/kadin2048/ichat_to_eml)  
**Language:** Python  
**License:** GPL-3.0  
**Last active:** 2021

**Purpose:** Convert Apple iChat / Messages archive files (.chat typedstream, .ichat binary plist) into RFC-compliant EML files (with optional HTML part and embedded attachments). The upstream project provides a pure-Python parser for NeXT "typedstream" files and binary PList (.ichat) files, and includes logic to extract NSAttachment/NSFileWrapper serialized attachments.

**Evidence (selected code):**
- Imports the typedstream parser and bplist decoder: `ichat_to_eml.py:28-31`[^k1]
- Detects `.chat` vs `.ichat` and uses `typedstream.unarchive_from_file` and `bplist_decode.bplist_to_conv`: `ichat_to_eml.py:69-82`[^k2]
- Handles NSAttachment / NSFileWrapper serialized attachments and decodes via `rtfd_decode.decode_rtfd`: `ichat_to_eml.py:173-186`[^k3]
- README credits `python-typedstream` and `ccl-bplist`: `README.md`[^k4]

**Implications:**
- Highly relevant to `chat.db` research for decoding older iChat `.chat` typedstream logs and newer `.ichat` binary plists. Use the upstream repo for attribution in any published research.
- Note license: GPL-3.0 — reuse or redistribution must comply with GPL terms.



**Data schema exposed to analyzers:**

*`messages` dataframe:*
| Column | Type | Description |
|--------|------|-------------|
| `ROWID` | int | Primary key |
| `text` | str | Decoded message body |
| `datetime` | datetime | Localized timestamp |
| `sender_display_name` | str | Resolved name or "Me" |
| `sender_handle` | str | Phone/email |
| `is_from_me` | bool | Direction |
| `is_reaction` | bool | Is tapback |

*`attachments` dataframe:*
| Column | Type | Description |
|--------|------|-------------|
| `ROWID` | int | Primary key |
| `filename` | str | File path |
| `mime_type` | str | MIME type |
| `message_id` | int | Parent message ROWID |
| `datetime` | datetime | Localized timestamp |
| `is_from_me` | bool | Direction |
| `sender_handle` | str | Phone/email |

### 16.3 my-other-github-account/imessage_tools

**Repository:** [https://github.com/my-other-github-account/imessage_tools](https://github.com/my-other-github-account/imessage_tools)  
**Language:** Python  
**Stars:** ~124

**Key function `read_messages(db_location, n, self_number, human_readable_date)`:**
- Joins `message` LEFT JOIN `handle`
- Reads `text` column; falls back to binary-parse `attributedBody` using the heuristic string-split method
- Converts date using: `(date + unix_timestamp_of_2001_01_01_in_ns) / 1e9`
- Queries `chat` table for `room_name → display_name` mapping
- Returns list of dicts with: `rowid`, `date`, `body`, `phone_number`, `is_from_me`, `cache_roomname`, `group_chat_name`

**Also provides `send_message(message, phone_number, group_chat=False)`** — sends via `osascript` to the Messages app. Writes message to a temp file to handle special characters.[^36]

### 16.4 niftycode/imessage_reader

**Repository:** [https://github.com/niftycode/imessage_reader](https://github.com/niftycode/imessage_reader)  
**Language:** Python 3.9+  
**Stars:** ~116  
**Package:** `pip install imessage-reader`

**Description:** A forensic-focused tool that reads `chat.db`. Notably can be used on **Linux** by specifying a path to a copied `chat.db`.[^37]

**Output formats:** Terminal display, Excel (`.xlsx` via openpyxl), SQLite database

**Fields extracted:** user_id (phone/email), message, date/time, service (iMessage/SMS), account, is_from_me

**Does not decode** `attributedBody` — relies solely on the `text` column.

---

## 17. Recommended Archival Approach

### For Long-Term Human-Readable Archives

**Primary: HTML Export via imessage-exporter**

```bash
# Install
brew install imessage-exporter

# Export all conversations to HTML with embedded attachments
imessage-exporter \
    --format html \
    --db-path ~/Library/Messages/chat.db \
    --attachment-manager full \
    --output ~/iMessage-Archive/

# Or with attachment format conversion for maximum portability
imessage-exporter \
    --format html \
    --db-path ~/Library/Messages/chat.db \
    --attachment-manager clone \
    --output ~/iMessage-Archive/
```

Produces one HTML file per conversation, with inline images/video/audio, thread rendering, tapbacks, and stickers.[^38]

### For Machine-Readable / LLM / Analysis Archives

**Secondary: JSON via caleb531/imessage-conversation-analyzer**

```bash
ica transcript -c 'Contact Name' -f json -o conversation_export.json
```

Or programmatic:
```python
import ica, json, pandas as pd

dfs = ica.get_dataframes(contacts=["Jane Doe"])
# Export messages as JSON
dfs.messages.to_json("messages.json", orient="records", date_format="iso")
dfs.attachments.to_json("attachments.json", orient="records", date_format="iso")
```

### For Forensic / Database-Level Archives

**Tertiary: SQLite Copy**

Simply copy the database file (and attachments directory) with the WAL files:
```bash
cp ~/Library/Messages/chat.db ~/iMessage-Backup/chat.db
cp ~/Library/Messages/chat.db-shm ~/iMessage-Backup/chat.db-shm
cp ~/Library/Messages/chat.db-wal ~/iMessage-Backup/chat.db-wal
rsync -av ~/Library/Messages/Attachments/ ~/iMessage-Backup/Attachments/
```

### Recommended Archive Checklist

- [ ] Export HTML (imessage-exporter) for browsability
- [ ] Export JSON/CSV (ICA or custom SQL) for machine analysis
- [ ] Copy raw `chat.db` + WAL files for forensic completeness
- [ ] Copy `Attachments/` directory preserving directory structure
- [ ] Convert HEIC/MOV to JPEG/MP4 if long-term accessibility is critical
- [ ] Note the macOS/iOS version at time of export (schema changes between versions)
- [ ] Grant Full Disk Access to Terminal before running any tool
- [ ] If iCloud Messages is enabled, run export promptly before any deletions propagate

### Standard Formats and Interoperability

There is currently **no official or widely-adopted standard format** for iMessage archives. The community has converged on:
- **HTML** (imessage-exporter) — most human-readable, self-contained
- **JSON** — most machine-readable
- **SQLite** (raw `chat.db`) — most faithful, requires tools to query

No XML standard (e.g., like Android's SMS Backup & Restore XML format) exists for iMessage. The LangChain `IMessageChatLoader` exports to ChatSession Python objects, easily serializable to JSON.[^39]

---

## 18. Key Repositories Summary

| Repository | Language | Stars | License | Focus |
|-----------|---------|-------|---------|-------|
| [ReagentX/imessage-exporter](https://github.com/ReagentX/imessage-exporter) | Rust | ~5,017 | GPL-3.0 | Full export (TXT/HTML), cross-platform, iOS backups |
| [caleb531/imessage-conversation-analyzer](https://github.com/caleb531/imessage-conversation-analyzer) | Python | ~33 | MIT | Analytics, transcript, pandas/duckdb API |
| [my-other-github-account/imessage_tools](https://github.com/my-other-github-account/imessage_tools) | Python | ~124 | N/A | Read+send, attributedBody heuristic decode |
| [niftycode/imessage_reader](https://github.com/niftycode/imessage_reader) | Python | ~116 | MIT | Forensic read, Excel/SQLite output |
| [johnlarkin1/imessage-schema](https://github.com/johnlarkin1/imessage-schema) | SQL | N/A | N/A | Schema snapshot reference |
| [langchain-ai/langchain](https://github.com/langchain-ai/langchain) (IMessageChatLoader) | Python | ~100k+ | MIT | LLM fine-tuning / RAG from iMessage |

---

## 19. Confidence Assessment

| Claim | Confidence | Evidence Source |
|-------|-----------|-----------------|
| Full 94-column `message` schema | **High** | Directly fetched from ReagentX/imessage-exporter `docs/tables/messages.md`[^3] |
| `attachment` table (24 columns) | **High** | Directly fetched from ReagentX/imessage-exporter `docs/tables/attachment.md`[^8] |
| `chat` table (29 columns) | **High** | Directly fetched from ReagentX/imessage-exporter `docs/tables/chat.md`[^4] |
| `handle` table (6 columns) | **High** | Directly fetched from ReagentX/imessage-exporter `docs/tables/handle.md`[^6] |
| Date encoding (nanoseconds, 2001 epoch) | **High** | Confirmed in SQL source code of caleb531 + web sources[^16][^17] |
| Tapback codes 2000–2005 | **High** | Multiple independent sources + imessage-exporter features.md |
| `attributedBody` heuristic decode | **High** | Source code directly read from imessage_tools.py[^22] |
| `style` field values (43/45/2/3) | **Medium** | Reported by community; not in official Apple docs |
| iOS `sms.db` path | **High** | Multiple forensic sources |
| iOS backup `sms.db` hash `3d0d7e5fb2ce...` | **High** | Multiple forensic sources |
| iCloud eviction `transfer_state=8` | **Medium** | Documented in community sources, consistent across tools |
| Schema current as of macOS Tahoe 26.4 | **High** | imessage-exporter README states "supports macOS Tahoe 26.4"[^32] |
| Star counts | **High** | Fetched via GitHub API at research time |

**Limitations:**
- Apple does not publicly document the `chat.db` schema; all column semantics are reverse-engineered
- The schema evolves with each major macOS/iOS release; new columns are added (86+ → 94 columns in Tahoe)
- `payload_data` / `attributedBody` internal structure is the least-documented aspect

---

## 20. Footnotes & Citations

[^k1]: `ichat_to_eml.py:28-31` (imports `typedstream`, `rtfd_decode`, `bplist_decode`) — upstream: [kadin2048/ichat_to_eml](https://github.com/kadin2048/ichat_to_eml/blob/main/ichat_to_eml.py)

[^k2]: `ichat_to_eml.py:69-82` (detects `.chat` vs `.ichat` and calls `typedstream.unarchive_from_file` and `bplist_decode.bplist_to_conv`) — upstream: [kadin2048/ichat_to_eml](https://github.com/kadin2048/ichat_to_eml/blob/main/ichat_to_eml.py)

[^k3]: `ichat_to_eml.py:173-186` (extracts NSAttachment/NSFileWrapper serialized attachments and calls `rtfd_decode.decode_rtfd`) — upstream: [kadin2048/ichat_to_eml](https://github.com/kadin2048/ichat_to_eml/blob/main/ichat_to_eml.py)

[^k4]: README acknowledgement of `python-typedstream` and `ccl-bplist` — `README.md` (upstream) — https://github.com/kadin2048/ichat_to_eml/blob/main/README.md

## 20. Footnotes & Citations

[^1]: macOS Full Disk Access requirement — multiple community sources; see [spin.atomicobject.com](https://spin.atomicobject.com/search-imessage-sql/) and [niftycode/imessage_reader README](https://github.com/niftycode/imessage_reader/blob/master/README.md)

[^2]: Tapbacks stored as message rows — [dev.to/arctype: Analyzing iMessage with SQL](https://dev.to/arctype/analyzing-imessage-with-sql-f42); [doubleblak.com Message Reactions](https://www.doubleblak.com/blogPost.php?k=Reactions)

[^3]: Full `message` table schema — [ReagentX/imessage-exporter `docs/tables/messages.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/tables/messages.md) (fetched directly)

[^4]: `chat` table schema — [ReagentX/imessage-exporter `docs/tables/chat.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/tables/chat.md) (fetched directly)

[^5]: `chat.style` values — community reverse engineering; referenced in multiple tools; confirmed via [deepwiki.com/hannesrudolph/imessage-query-fastmcp-mcp-server](https://deepwiki.com/hannesrudolph/imessage-query-fastmcp-mcp-server/6.1-database-structure)

[^6]: `handle` table schema — [ReagentX/imessage-exporter `docs/tables/handle.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/tables/handle.md) (fetched directly)

[^7]: Handle deduplication via `person_centric_id` — [ReagentX/imessage-exporter `docs/tables/duplicates.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/tables/duplicates.md); features.md: "Different handles that belong to the same person are combined"

[^8]: `attachment` table schema — [ReagentX/imessage-exporter `docs/tables/attachment.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/tables/attachment.md) (fetched directly)

[^9]: `chat_message_join` table — web search consensus; [caleb531/imessage-conversation-analyzer `ica/queries/messages.sql`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/messages.sql)

[^10]: `chat_handle_join` table — web search consensus; used in [my-other-github-account/imessage_tools `imessage_tools.py`](https://github.com/my-other-github-account/imessage_tools/blob/master/imessage_tools.py) (`get_chat_mapping`)

[^11]: `message_attachment_join` — [caleb531/imessage-conversation-analyzer `ica/queries/attachments.sql`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/attachments.sql)

[^12]: Attachment directory structure — [spin.atomicobject.com](https://spin.atomicobject.com/search-imessage-sql/); [davidbieber.com](https://davidbieber.com/snippets/2020-05-20-imessage-sql-db/); [GitHub Gist: iMessage Attachments Extractor](https://gist.github.com/rmtbb/c77deb7a697b032d374c57cf96aa98c8)

[^13]: Attachment format conversion — [ReagentX/imessage-exporter `docs/features.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/features.md) (fetched directly): "HEIC→JPEG, HEICS→GIF, MOV→mp4, CAF/AMR→mp4"

[^14]: iOS backup structure — [reincubate.com: iPhone backup files structure](https://reincubate.com/support/how-to/iphone-backup-files-structure/); [richinfante.com: Reverse Engineering the iOS Backup](https://www.richinfante.com/2017/3/16/reverse-engineering-the-ios-backup/)

[^15]: iCloud storage optimization — community documentation; [elcomsoft.com: Forensic View of iMessage Security](https://blog.elcomsoft.com/2020/10/the-forensic-view-of-imessage-security/)

[^16]: Date conversion in SQL — [caleb531/imessage-conversation-analyzer `ica/queries/messages.sql` line 6](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/messages.sql): `datetime("message"."date" / 1000000000 + strftime("%s", "2001-01-01") ,"unixepoch")`

[^17]: Apple epoch (2001-01-01) offset = 978,307,200 seconds — [epochconverter.com/coredata](https://www.epochconverter.com/coredata); [apple.stackexchange.com: Dates format in Message's chat.db](https://apple.stackexchange.com/questions/114168/dates-format-in-messages-chat-db); [LangChain `nanoseconds_from_2001_to_datetime`](https://api.python.langchain.com/en/latest/community/chat_loaders/langchain_community.chat_loaders.imessage.nanoseconds_from_2001_to_datetime.html)

[^18]: `attributedBody` replacing `text` in Ventura — [my-other-github-account/imessage_tools README](https://github.com/my-other-github-account/imessage_tools/blob/master/README.md): "Reads messages not visible through normal SQL queries by parsing attributeBody Blob field (this seems to be necessary to see all messages on MacOS Ventura and later)"; [stackoverflow.com: How can I read the attributedBody column](https://stackoverflow.com/questions/75330393/how-can-i-read-the-attributedbody-column-in-macos-imessage-database)

[^19]: `pytypedstream` dependency — [caleb531/imessage-conversation-analyzer `pyproject.toml`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/pyproject.toml) (fetched directly)

[^20]: `crabstep` Rust crate for typedstream — [ReagentX/imessage-exporter `docs/features.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/features.md): "Parses typedstream message body data using crabstep"; author's blog post: [chrissardegna.com: Reverse Engineering Apple's typedstream Format](https://chrissardegna.com/blog/reverse-engineering-apples-typedstream-format/)

[^21]: Heuristic `attributedBody` decode — [my-other-github-account/imessage_tools `imessage_tools.py` lines 28-38](https://github.com/my-other-github-account/imessage_tools/blob/master/imessage_tools.py) (fetched directly)

[^22]: Source of heuristic decode code — same as [^21]

[^23]: Tapback `associated_message_type` values — [doubleblak.com Message Reactions](https://www.doubleblak.com/blogPost.php?k=Reactions); [dev.to/arctype](https://dev.to/arctype/analyzing-imessage-with-sql-f42); [jakespurlock.com: I Built My Own iMessage Wrapped](https://jakespurlock.com/2026/02/i-built-my-own-imessage-wrapped-and-so-can-you/)

[^24]: iOS 18 emoji reactions column (`associated_message_emoji`) — column 85 in ReagentX message table schema

[^25]: `payload_data` decoding — [ReagentX/imessage-exporter `docs/features.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/features.md): "Parses the NSKeyedArchiver payload to extract preview data"; special thanks note: "Xplist, an invaluable tool for reverse engineering the payload_data plist format"

[^26]: iOS `sms.db` as macOS equivalent — [ReagentX/imessage-exporter `docs/features.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/features.md): "Jailbroken iOS filesystem data: Uses sms.db, which follows the same schema as macOS chat.db"

[^27]: iOS backup `sms.db` hash filename — [reincubate.com](https://reincubate.com/support/how-to/iphone-backup-files-structure/); [blog.digital-forensics.it: iOS 15 Image Forensics Analysis](https://blog.digital-forensics.it/2023/10/ios-15-image-forensics-analysis-and.html)

[^28]: iCloud sync deletion propagation — [elcomsoft.com: Forensic View of iMessage Security](https://blog.elcomsoft.com/2020/10/the-forensic-view-of-imessage-security/); [vestigeltd.com: Forensic Challenges with iPhone Text Message Recovery](https://www.vestigeltd.com/resources/articles/forensic-challenges-and-solutions-with-the-new-iphone-message-deletion/)

[^29]: WAL file importance for forensics — standard SQLite WAL documentation; forensic community best practice

[^30]: Conversation reconstruction SQL pattern — derived from [caleb531/imessage-conversation-analyzer `ica/queries/messages.sql`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/messages.sql) (fetched directly)

[^31]: LangChain `IMessageChatLoader` attributedBody issue — [langchain-ai/langchain GitHub issue #10680](https://github.com/langchain-ai/langchain/issues/10680): "iMessageChatLoader doesn't load new text content, due to ChatHistory only stored in attributedBody"

[^32]: imessage-exporter supported features — [ReagentX/imessage-exporter `docs/features.md`](https://github.com/ReagentX/imessage-exporter/blob/main/docs/features.md) and [README.md](https://github.com/ReagentX/imessage-exporter/blob/main/README.md) (both fetched directly)

[^33]: ICA dependencies — [caleb531/imessage-conversation-analyzer `pyproject.toml`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/pyproject.toml) (fetched directly)

[^34]: ICA messages SQL — [caleb531/imessage-conversation-analyzer `ica/queries/messages.sql`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/messages.sql) (fetched directly)

[^35]: ICA attachments SQL — [caleb531/imessage-conversation-analyzer `ica/queries/attachments.sql`](https://github.com/caleb531/imessage-conversation-analyzer/blob/main/ica/queries/attachments.sql) (fetched directly)

[^36]: `send_message` via osascript — [my-other-github-account/imessage_tools `imessage_tools.py` lines 58-75](https://github.com/my-other-github-account/imessage_tools/blob/master/imessage_tools.py) (fetched directly)

[^37]: niftycode/imessage_reader Linux support — [README.md](https://github.com/niftycode/imessage_reader/blob/master/README.md): "this program can also be used under Linux. In contrast to use under macOS, the path to the chat.db file must then be specified"

[^38]: imessage-exporter HTML export — [ReagentX/imessage-exporter README.md](https://github.com/ReagentX/imessage-exporter/blob/main/README.md) (fetched directly)

[^39]: LangChain IMessageChatLoader — [docs.langchain.com/oss/python/integrations/chat_loaders/imessage](https://docs.langchain.com/oss/python/integrations/chat_loaders/imessage)
