# Hotline Protocol Extension – Large Files (Draft)

This document describes the large-file extension as implemented by HLServer.

## Table of Contents

- [Scope](#scope)
- [Terminology](#terminology)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [New and Extended Data Objects](#new-and-extended-data-objects)
- [Transaction Changes](#transaction-changes)
  - [Login (107)](#login-107)
  - [Get File Name List (200)](#get-file-name-list-200)
  - [Download File (202)](#download-file-202)
  - [Upload File (203)](#upload-file-203)
  - [Get File Info (206)](#get-file-info-206)
  - [Download Folder (210) / Upload Folder (213)](#download-folder-210--upload-folder-213)
- [Transfer Side-Channel (HTXF)](#transfer-side-channel-htxf)
  - [Handshake Flags and Length](#handshake-flags-and-length)
  - [Flattened File Object Fork Headers](#flattened-file-object-fork-headers)
- [Notes](#notes)

## Scope

Adds 64-bit sizing to control-plane fields (transactions) and transfer-plane headers (HTXF) while remaining backwards compatible with legacy 32-bit clients.

## Terminology

- *Client* / *Server*: Hotline peers.
- *Legacy*: Implementations that only understand 32-bit file sizes.
- *Large-file mode*: Both sides negotiated support for this extension.

## Compatibility and Negotiation

- Large-file mode is enabled server-side (e.g., via `large_file_clients` or a global allow flag) based on the connecting client's version string.
- Clients SHOULD send the 64-bit companion fields listed below; servers that have not enabled large-file mode MAY ignore those 64-bit fields and fall back to the legacy 32-bit values.

## New and Extended Data Objects

| ID (hex) | Name                  | Width | Notes |
|----------|-----------------------|-------|-------|
| 0x01F1   | `DATA_FILESIZE64`     | 64    | File size in bytes. Paired with legacy `DATA_FILESIZE` (32-bit) for compatibility. |
| 0x01F2   | `DATA_OFFSET64`       | 64    | Offset into file (downloads/resume). Paired with legacy `DATA_OFFSET`. |
| 0x01F3   | `DATA_XFERSIZE64`     | 64    | Transfer length (remaining bytes). Paired with legacy `DATA_XFERSIZE`. |
| 0x01F4   | `DATA_FOLDER_ITEM_COUNT64` | 64 | Folder item counts. Paired with legacy `DATA_FOLDER_ITEM_COUNT`. |

Send both 32-bit and 64-bit forms when large-file mode is active; legacy fields remain mandatory for compatibility.

## Transaction Changes

### Login (107)

- No additional data objects are required. Large-file support is granted server-side (version or global enable) using the client version string.

### Get File Name List (200)

- Server: when large-file mode is active, append `DATA_FILESIZE64` immediately after each `DATA_FILE` object to provide the full byte length. The legacy `FileSize` field in `DATA_FILE` remains 32-bit and may clamp at 0xFFFFFFFF.
- Client: prefer `DATA_FILESIZE64` when present; fall back to the 32-bit value otherwise.

### Download File (202)

- Request: clients MAY include `DATA_XFERSIZE64` to request a 64-bit-length subset; include `DATA_OFFSET64` for 64-bit resume offsets. Legacy `DATA_XFERSIZE`/`DATA_OFFSET` SHOULD still be sent with clamped values.
- Reply: servers include both legacy (`DATA_FILESIZE`, `DATA_OFFSET`, `DATA_XFERSIZE`) and 64-bit (`DATA_FILESIZE64`, `DATA_OFFSET64`, `DATA_XFERSIZE64`) sizes when large-file mode is active.
- Transfer side-channel MUST set the HTXF large-file flag (see below) when large-file mode is active.

### Upload File (203)

- Request: clients SHOULD send `DATA_XFERSIZE64` with the full upload size and keep `DATA_XFERSIZE` as the low 32 bits (or clamped). Servers use the 64-bit value when available.
- Reply: unchanged; refnum allocation is 32-bit.
- Transfer side-channel MUST set the HTXF large-file flag when large-file mode is active.

### Get File Info (206)

- Reply: servers include `DATA_FILESIZE64` alongside `DATA_FILESIZE` when large-file mode is active.

### Download Folder (210) / Upload Folder (213)

- Replies and progress packets include `DATA_FOLDER_ITEM_COUNT64` and `DATA_XFERSIZE64` when large-file mode is active; legacy 32-bit fields remain present.

## Transfer Side-Channel (HTXF)

### Handshake Flags and Length

- HTXF flags field (bytes 12–15 of the handshake) uses bit `0x00000001` (`HTXF_FLAG_LARGE_FILE`) to indicate large-file mode. Clients set this flag when large-file mode is negotiated or when the transfer size exceeds 32-bit; servers mirror the flag into transfer state.
- HTXF length field (bytes 8–11) is legacy 32-bit. When the total transfer length exceeds `0xFFFFFFFF`, set this field to **zero** and rely on the 64-bit sizes provided in the transaction control-plane fields (the `DATA_*` objects in the associated control transactions). This prevents unintended clamping of the transfer.

### Flattened File Object Fork Headers

When `HTXF_FLAG_LARGE_FILE` is set, fork headers in the Flattened File Object use high/low 32-bit words to carry 64-bit lengths:

- Bytes 0–3: Fork type (`"DATA"`, `"MACR"`, etc.).
- Bytes 4–7: High 32 bits of the fork length.
- Bytes 12–15: Low 32 bits of the fork length.

Info fork lengths follow the same pattern. Legacy readers that ignore the high word will see truncated sizes and should not enable large-file mode.

## Notes

- Always include legacy 32-bit fields for compatibility, even in large-file mode.
- Resume uses the 64-bit offset fields when present; fallback is the legacy 32-bit offset.
- Resource forks: resource fork headers also use the high/low 64-bit layout under `HTXF_FLAG_LARGE_FILE`.
- Setting HTXF length to zero for >4 GiB is required to avoid early termination by 32-bit peers.

Status: draft; subject to refinement  
Codename: Pitbull? More like Bullshit!
