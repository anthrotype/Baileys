# Baileys LID `xml-not-well-formed` Investigation

## Issue

GitHub Issue: https://github.com/WhiskeySockets/Baileys/issues/2074

Connection drops with `stream:error` → `xml-not-well-formed` when interacting with LID users.

## Root Cause

Malformed JIDs with double `@` symbols like `XX@lid@s.whatsapp.net`.

### Vulnerable Code

**`src/Utils/process-message.ts:403`** in `LID_MIGRATION_MAPPING_SYNC` handler:

```typescript
for (const { pn, latestLid, assignedLid } of pnToLidMappings) {
    const lid = latestLid || assignedLid
    pairs.push({ lid: `${lid}@lid`, pn: `${pn}@s.whatsapp.net` })
}
```

If `lid` or `pn` from the protobuf already contains `@`, this creates malformed JIDs.

### Error Flow

1. Server sends `LID_MIGRATION_MAPPING_SYNC` with JID values that already include server suffix
2. Baileys appends another suffix → `XX@lid@lid` or `XX@s.whatsapp.net@s.whatsapp.net`
3. Malformed JIDs stored in mapping cache
4. Later encoding of messages/acks with these JIDs:
   - `jidDecode('XX@lid@s.whatsapp.net')` → `user="XX"`, `server="lid@s.whatsapp.net"` (invalid)
   - Binary encoding produces data server can't parse
5. Server responds with `stream:error` → `xml-not-well-formed`

## Proposed Fix

```typescript
for (const { pn, latestLid, assignedLid } of pnToLidMappings) {
    const lid = latestLid || assignedLid
    // Handle case where protobuf values are already complete JIDs
    const lidJid = lid?.includes('@') ? lid : `${lid}@lid`
    const pnJid = pn?.includes('@') ? pn : `${pn}@s.whatsapp.net`
    pairs.push({ lid: lidJid, pn: pnJid })
}
```

Or with proper JID handling:

```typescript
for (const { pn, latestLid, assignedLid } of pnToLidMappings) {
    const lid = latestLid || assignedLid
    const lidDecoded = jidDecode(lid)
    const pnDecoded = jidDecode(pn)

    const lidJid = lidDecoded
        ? jidEncode(lidDecoded.user, 'lid', lidDecoded.device)
        : `${lid}@lid`
    const pnJid = pnDecoded
        ? jidEncode(pnDecoded.user, 's.whatsapp.net', pnDecoded.device)
        : `${pn}@s.whatsapp.net`

    pairs.push({ lid: lidJid, pn: pnJid })
}
```

## Related Code Paths

- `src/WABinary/encode.ts` - JID binary encoding (`writeJid`)
- `src/WABinary/decode.ts` - JID binary decoding (`readJidPair`, `readAdJid`)
- `src/WABinary/jid-utils.ts` - JID parsing (`jidDecode`, `jidEncode`)
- `src/Socket/messages-recv.ts:332-367` - `sendMessageAck()` uses `attrs.from` directly
- `src/Signal/lid-mapping.ts` - LID ↔ PN mapping cache

## Date

2026-01-02
