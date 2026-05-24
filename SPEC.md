# Model Interface Device (MID) Class Specification

**Version:** 0.1 (draft)
**Status:** Pre-submission. Normative language is RFC 2119; nothing here has been ratified by USB-IF.
**Companion:** [`mcp-hid-usage-page`](../mcp-hid-usage-page).

---

## 1. Scope

This document specifies the **Model Interface Device** (MID) class — a USB device class whose function is to expose a [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server to a connected host. MID covers:

- The class, subclass, and protocol code assignment.
- The interface descriptor and the MID-specific functional descriptor.
- Endpoint structure (Bulk pair plus optional Interrupt-IN).
- Framing for JSON-RPC over Bulk.
- A tiered discovery model with host-side caching.
- A pre-ratification discovery primitive via BOS Platform Capability.
- Auto-binding to WinUSB through MS OS 2.0 descriptors.

MID does **not** specify the MCP protocol itself; that is defined by the MCP specification and referenced normatively. MID is a USB transport binding for it.

## 2. Terminology

- **MID device** — a USB device exposing at least one MID interface.
- **MID host** — a USB host implementation that recognises and binds MID interfaces.
- **MCP server / client** — the application-layer endpoints; the MID device contains the server, the MID host contains the client.
- **Tool / resource / prompt** — as defined in the MCP specification.
- **Descriptor cache hash** — a stable hash of the device's full capability set, used by hosts to skip re-fetching schemas across reconnects.
- **Capability index** — the compact name + per-item schema-hash table fetched via control transfer.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY are to be interpreted as in RFC 2119.

## 3. Class identifiers

| Field | Value | Notes |
|---|---|---|
| `bInterfaceClass` | `MID` (TBD by USB-IF) | Pre-ratification: `0xFF` + BOS Platform Capability (§10). |
| `bInterfaceSubClass` | `0x01` | MID v1 base subclass. Future subclasses for profiled variants (e.g. low-power). |
| `bInterfaceProtocol` | `0x01` | "JSON-RPC over Bulk" protocol. Reserved range for future framings (e.g. CBOR-RPC). |

Composite devices SHOULD use an Interface Association Descriptor (IAD) to group the MID interface with any related interfaces.

## 4. Interface and endpoint structure

An MID interface MUST contain:

- One **Bulk-OUT** endpoint (host → device) for JSON-RPC requests and notifications from the client.
- One **Bulk-IN** endpoint (device → host) for JSON-RPC responses and server-initiated requests.
- An MID-specific **functional descriptor** (§5) appearing after the standard interface descriptor and before the endpoint descriptors.

An MID interface MAY additionally contain:

- One **Interrupt-IN** endpoint for asynchronous `notifications/*/list_changed` and other low-latency MCP notifications. If present, the functional descriptor's `bmCapabilities` bit 0 MUST be set.

An MID interface MUST NOT use Isochronous endpoints. An MID interface MUST NOT contain more than one Bulk-OUT, one Bulk-IN, or one Interrupt-IN endpoint.

`wMaxPacketSize` for the Bulk endpoints SHOULD be the device's natural maximum (64 for Full-Speed, 512 for High-Speed, 1024 for SuperSpeed Bulk). The framing is length-prefixed (§7), so packet size does not constrain message size.

## 5. MID functional descriptor

The MID functional descriptor sits inside the interface descriptor's class-specific region and conveys the discovery summary (Tier 1, §6). Its layout:

```
Offset  Size  Field              Notes
0       1     bLength            = 32
1       1     bDescriptorType    = 0x24 (CS_INTERFACE)
2       1     bDescriptorSubType = 0x01 (MID_HEADER)
3       2     bcdMID             BCD MID version, e.g. 0x0100 for 1.0
5       1     bmCapabilities     bit 0: interrupt-in notifications present
                                  bit 1: server supports sampling/createMessage
                                  bit 2: server supports roots
                                  bit 3: server supports prompts
                                  bit 4: server supports tools
                                  bit 5: server supports resources
                                  bit 6: server supports resource subscriptions
                                  bit 7: reserved (MBZ)
6       2     wToolCount         Number of tools exposed (informational; full list via T3)
8       2     wResourceCount     Number of resources exposed
10      2     wPromptCount       Number of prompts exposed
12      16    descriptorCacheHash  SHA-256/128 of the full capability set (low 128 bits)
28      1     bDataEndpointIn    Bulk-IN endpoint address
29      1     bDataEndpointOut   Bulk-OUT endpoint address
30      1     bNotifyEndpointIn  Interrupt-IN endpoint address, or 0 if absent
31      1     bIndexMaxSize      Capability-index max size in 256-byte units (§6.2)
```

Total: 32 bytes. Devices MUST NOT extend this descriptor; future versions bump `bcdMID` and define a new layout.

`descriptorCacheHash` is the truncated SHA-256 of the canonical serialisation of the full MCP capability set (tools, resources, prompts, with their full schemas) as defined in §6.3. Hosts MAY use this hash to skip re-fetching anything the device has not changed since the last connection.

## 6. Tiered discovery

MID specifies three discovery tiers. Each tier reveals more detail; hosts pay for what they need.

### 6.1 Tier 1 — Enumeration

Available during USB enumeration without claiming any interface. Carried by the MID functional descriptor (§5). At this tier the host learns:

- That the device is an MID device, and the MID version.
- The MCP capability bitmap (which of tools / resources / prompts / sampling / roots / subscriptions the server exposes).
- Coarse counts of tools / resources / prompts.
- The descriptor cache hash and the data endpoint addresses.

A host that has previously talked to this device (matched by VID/PID/serial + descriptorCacheHash) MAY skip Tiers 2 and 3 entirely and use its cached schemas.

### 6.2 Tier 2 — Capability index (control transfer)

Fetched via a class-specific GET_CAPABILITY_INDEX request:

```
bmRequestType  0xA1 (device-to-host, class, interface)
bRequest       0x01 (GET_CAPABILITY_INDEX)
wValue         0x0000
wIndex         interface number
wLength        up to (bIndexMaxSize × 256) bytes
```

The response is a CBOR document (chosen over JSON to keep the index dense):

```cbor
{
  "v": 1,
  "tools":     [{"n": "<name>", "t": "<short title>", "h": h128(schema)}, ...],
  "resources": [{"u": "<uri>",  "t": "<short title>", "h": h128(schema)}, ...],
  "prompts":   [{"n": "<name>", "t": "<short title>", "h": h128(args_schema)}, ...]
}
```

Each entry is 32–64 bytes. 200 tools fit in ~6–8 KB. Hosts use the per-item `h` hashes to fetch only changed schemas in Tier 3. `bIndexMaxSize` in the functional descriptor advertises the upper bound; hosts MUST NOT request `wLength` greater than `bIndexMaxSize × 256`.

Devices with capability indexes larger than 64 KB MUST chunk the index using a GET_CAPABILITY_INDEX_CHUNK class request (`bRequest = 0x02`, `wValue` = zero-indexed chunk number). This case is expected to be rare.

### 6.3 Tier 3 — Full schemas (over the MCP channel)

Hosts that need full schemas claim the MID interface and speak MCP over the Bulk endpoints. The standard MCP methods apply unchanged:

- `tools/list`, optionally with pagination cursor.
- `resources/list`, `resources/templates/list`.
- `prompts/list`.
- Per-item fetches (`resources/read`, etc.) as needed.

A device MUST be prepared to answer the MCP `initialize` handshake immediately after the host claims the interface. The MCP server inside the device MUST advertise only the capabilities consistent with the Tier-1 bitmap.

### 6.4 Hash construction

`descriptorCacheHash` and per-item `h` are the low 128 bits of SHA-256 over a canonical JSON serialisation of the underlying object. Canonicalisation:

- Object keys sorted lexicographically (UTF-8 code-point order).
- No insignificant whitespace.
- Numbers in shortest round-trippable form.
- Strings in NFC.

This matches JCS (RFC 8785) for the subset MCP schemas use.

## 7. Framing on Bulk endpoints

JSON-RPC messages are framed as:

```
4 bytes  uint32, little-endian  message length in bytes (excluding this header)
N bytes  UTF-8 JSON-RPC message
```

A message MAY span multiple Bulk transfers. A short-packet (i.e. a packet smaller than `wMaxPacketSize`, including zero-length) is not significant — receivers MUST rely on the length field, not packet boundaries.

Implementations SHOULD pipeline: the host MAY issue a new request before the previous one's response arrives, and the device MAY interleave responses on the Bulk-IN endpoint (JSON-RPC's `id` field disambiguates).

Devices SHOULD impose a maximum message size and reject larger messages with JSON-RPC error `-32600` (Invalid Request). A reasonable default ceiling is 256 KB.

## 8. Notifications (Interrupt-IN)

If the optional Interrupt-IN endpoint is present, the device uses it to deliver MCP notifications whose payload is small and latency-sensitive — primarily `notifications/tools/list_changed`, `notifications/resources/list_changed`, `notifications/prompts/list_changed`, and `notifications/resources/updated`.

Framing on Interrupt-IN is identical to §7 (4-byte length prefix + UTF-8 JSON-RPC). Devices MUST NOT send request messages on the Interrupt-IN endpoint; only JSON-RPC notifications. Larger notifications and all responses MUST be carried on the Bulk-IN endpoint.

## 9. WinUSB auto-binding (MS OS 2.0 descriptors)

To avoid driver installation on Windows, MID devices SHOULD ship a Microsoft OS 2.0 descriptor set containing:

- A compatible ID feature setting the compatible ID to `WINUSB`.
- A registry-property feature setting `DeviceInterfaceGUIDs` to `{B7D5A2C3-7E8E-4F8A-9F0C-3D2A1E6F4B70}` (the MID class GUID — reserved by this draft).

Microsoft's BOS descriptor pulls this on first connection; no INF, no installer. Linux (libusb) and macOS (IOKit) require no equivalent.

## 10. Pre-ratification discovery (BOS Platform Capability)

Until USB-IF allocates a class code for MID, devices and hosts use a BOS Platform Capability descriptor as the discovery primitive. Devices ship with `bInterfaceClass = 0xFF` (vendor-specific) for the would-be MID interface, plus a BOS descriptor advertising MID support.

The Platform Capability layout:

```
Offset  Size  Field            Notes
0       1     bLength          = 28
1       1     bDescriptorType  = 0x10 (DEVICE_CAPABILITY)
2       1     bDevCapabilityType = 0x05 (PLATFORM)
3       1     bReserved        = 0
4       16    PlatformCapabilityUUID
                                 {B7D5A2C3-7E8E-4F8A-9F0C-3D2A1E6F4B70}
                                 (reserved by this draft; will be replaced
                                  by the USB-IF class-code path once allocated)
20      2     bcdMID           BCD MID version
22      1     bInterface       Interface number of the MID interface
23      1     bmCapabilities   Same bit layout as §5
24      4     descriptorCacheHash   Low 32 bits of the §5 hash (collision-tolerant; not
                                    a security boundary at this resolution)
```

Hosts that recognise the UUID treat the indicated interface as MID-class regardless of `bInterfaceClass`. Hosts SHOULD still fetch the full functional descriptor (§5) from the interface for the full 128-bit cache hash and counts.

## 11. Error handling

JSON-RPC errors follow the MCP spec. USB-level errors (STALL, NAK timeouts) are handled per the USB Common Class Specification. The device MUST clear a STALLed Bulk endpoint on receipt of CLEAR_FEATURE(ENDPOINT_HALT) and resume from a quiesced state — there MUST NOT be any partial JSON-RPC message left in flight after a stall-and-clear cycle.

## 12. Power and bandwidth

MID does not mandate a power class. Devices MAY be bus-powered or self-powered. Bandwidth is whatever the host scheduler grants the Bulk endpoints; devices that need predictable latency for notifications SHOULD use the Interrupt-IN endpoint (§8).

## 13. Security considerations

MCP itself does not currently specify in-band authentication suitable for a wire transport. MID devices SHOULD treat any locally-connected host as having the rights granted by physical access, the same posture as HID and MSC. Devices that need finer-grained access control SHOULD implement application-layer authentication inside the MCP layer (e.g. a `session/authenticate` tool that returns a capability token, with subsequent tools gated on it).

A malicious device on a shared host can already exfiltrate via HID and MSC; MID does not increase that surface meaningfully. A malicious host can call any exposed tool; devices SHOULD scope tool behavior accordingly (a tool that erases firmware SHOULD require confirmation through a side channel, not solely an MCP call).

## 14. Conformance

A conforming MID device MUST:

- Present the functional descriptor of §5 with `bcdMID = 0x0100` for v1.0.
- Implement `GET_CAPABILITY_INDEX` (§6.2).
- Speak MCP over Bulk with the framing of §7.
- Respond to MCP `initialize` consistently with the Tier-1 capability bitmap.

A conforming MID host MUST:

- Recognise MID interfaces via class code (post-ratification) or BOS Platform Capability (§10).
- Honour the `descriptorCacheHash` for caching decisions.
- Speak MCP over Bulk with the framing of §7.

A conforming implementation MAY skip the Interrupt-IN endpoint (§8) and the Tier-2 capability index (§6.2), at the cost of less efficient discovery.

## Appendix A — Reserved values

| Resource | Value | Notes |
|---|---|---|
| MID class code | TBD by USB-IF | Use `0xFF` + BOS until allocated. |
| MID Platform Capability UUID | `B7D5A2C3-7E8E-4F8A-9F0C-3D2A1E6F4B70` | Reserved by this draft. |
| Class device interface GUID (Windows) | `B7D5A2C3-7E8E-4F8A-9F0C-3D2A1E6F4B70` | Reused; not a security boundary. |
| GET_CAPABILITY_INDEX | `bRequest = 0x01` | §6.2 |
| GET_CAPABILITY_INDEX_CHUNK | `bRequest = 0x02` | §6.2 |

## Appendix B — Open questions

- Should the capability index be CBOR or a custom packed format? CBOR is verbose but tool-friendly; a packed format saves ~30%.
- Should `descriptorCacheHash` cover the MCP server's `serverInfo` / `instructions` fields, or just the capability set? Current draft: capability set only, so instruction tweaks don't bust the cache.
- Sampling and `createMessage` reverse the data flow; the framing of §7 supports it, but should the spec mandate that devices declaring sampling capability also implement flow control beyond what USB provides?
- Multi-interface MID devices: should `descriptorCacheHash` cover one interface or the union? Current draft: one interface.

Issues and PRs welcome; see [`CONTRIBUTING.md`](CONTRIBUTING.md).
