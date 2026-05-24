# Example: generic MID device

A walk-through of what an MID device looks like on the wire, independent of any specific silicon. Use this to sanity-check the spec against a concrete scenario; the nRF5340 reference in [`nrf5340.md`](nrf5340.md) does the same exercise for a specific SoC.

The example device is a hypothetical environmental sensor pack: one tool (`take_reading`), one resource (a streaming `live` feed), no prompts, no sampling.

## Device-side: descriptors

The device exposes a composite configuration with two interfaces:

- **Interface 0**: CDC ACM (for legacy serial-port shell access — unrelated to MID).
- **Interface 1**: MID (our MCP server).

### Device descriptor

Standard. `bcdUSB = 0x0210` (USB 2.1, required so the host fetches BOS).

### BOS descriptor

Carries the Platform Capability descriptor of `SPEC.md` §10. UUID `B7D5A2C3-7E8E-4F8A-9F0C-3D2A1E6F4B70`, `bcdMID = 0x0100`, `bInterface = 1`, `bmCapabilities = 0b00110001` (interrupt notifications + tools + resources).

Also carries the MS OS 2.0 Platform Capability descriptor (`SPEC.md` §9), which Windows will use to bind interface 1 to WinUSB.

### Configuration descriptor

Pre-ratification: interface 1 has `bInterfaceClass = 0xFF` (vendor-specific). Once USB-IF allocates a code, this changes to that code; nothing else moves.

### MID functional descriptor (interface 1)

Per `SPEC.md` §5:

| Field | Value |
|---|---|
| `bLength` | 32 |
| `bDescriptorType` | 0x24 |
| `bDescriptorSubType` | 0x01 |
| `bcdMID` | 0x0100 |
| `bmCapabilities` | 0b00110001 (interrupt-in notifications + tools + resources) |
| `wToolCount` | 1 |
| `wResourceCount` | 1 |
| `wPromptCount` | 0 |
| `descriptorCacheHash` | (128-bit hash of the capability set; see below) |
| `bDataEndpointIn` | 0x82 |
| `bDataEndpointOut` | 0x02 |
| `bNotifyEndpointIn` | 0x83 |
| `bIndexMaxSize` | 1 (so the index fits in ≤256 bytes — plenty for two items) |

### Endpoints (interface 1)

- `0x02` Bulk-OUT, `wMaxPacketSize = 64`.
- `0x82` Bulk-IN, `wMaxPacketSize = 64`.
- `0x83` Interrupt-IN, `wMaxPacketSize = 16`, `bInterval = 16` ms.

## Host-side: the discovery flow

A host that already knows this device (matched by VID/PID/serial + descriptor cache hash) can short-circuit to step 5.

### 1. Enumeration
Host fetches device descriptor → BOS → recognises the MID Platform Capability UUID → notes that interface 1 is an MID interface. No interface claimed yet.

### 2. Functional descriptor
Host fetches the class-specific descriptors on interface 1, finds the MID functional descriptor of §5. Reads counts and `descriptorCacheHash`.

### 3. Cache check
Host looks up `(VID, PID, serial, descriptorCacheHash)` in its local cache. If present, jump to step 5.

### 4. Tier 2 — capability index
Host issues `GET_CAPABILITY_INDEX` on the interface (class-specific control transfer). Receives a CBOR document like:

```cbor
{
  "v": 1,
  "tools": [
    {"n": "take_reading", "t": "Read all sensors", "h": h128(...)}
  ],
  "resources": [
    {"u": "live://sensors", "t": "Streaming feed", "h": h128(...)}
  ],
  "prompts": []
}
```

Host checks the per-item `h` hashes against its cache; only fetches schemas for items it doesn't already have.

### 5. Claim interface, initialize MCP
Host issues `SET_INTERFACE`, claims interface 1, and the device is ready. Host opens the Bulk pair and sends:

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
  "protocolVersion":"2025-06-18",
  "capabilities":{},
  "clientInfo":{"name":"example-host","version":"0.1"}
}}
```

framed as 4-byte length + UTF-8 bytes on Bulk-OUT.

Device responds on Bulk-IN with the standard MCP `initialize` result, advertising tools + resources (the same capabilities the §5 bitmap promised).

### 6. Fetch any missing schemas
For each tool / resource whose `h` didn't match cache, host issues the standard MCP listing methods (`tools/list`, etc.) over the same Bulk channel. Pagination per the MCP spec applies; the device caps `nextCursor` chunks so each response fits comfortably in a Bulk transfer.

### 7. Notifications
Device pushes `notifications/resources/updated` on the Interrupt-IN endpoint when the streaming feed changes — same JSON-RPC framing, no host polling required.

## What the device firmware looks like (sketch)

Pseudocode for the device-side dispatcher (transport-agnostic; substitute your USB stack's Bulk read/write):

```c
typedef struct { uint32_t len; uint8_t buf[256 * 1024]; } message_t;

static void mid_task(void) {
  message_t msg;
  for (;;) {
    if (!bulk_out_read_framed(&msg)) continue;            // §7 framing
    json_t *req = json_parse_n(msg.buf, msg.len);
    json_t *res = mcp_dispatch(req);                       // your MCP server
    json_free(req);
    uint8_t *out; size_t out_len;
    json_serialize(res, &out, &out_len);
    bulk_in_write_framed(out, out_len);                    // §7 framing
    json_free(res);
    free(out);
  }
}

static void on_sensor_change(void) {
  static const char *notif =
    "{\"jsonrpc\":\"2.0\",\"method\":\"notifications/resources/updated\","
    "\"params\":{\"uri\":\"live://sensors\"}}";
  interrupt_in_write_framed((const uint8_t*)notif, strlen(notif));  // §8
}
```

The `mcp_dispatch` call is your existing MCP server logic — MID is purely the transport binding. The same dispatcher works over stdio, WebSocket, or MID.

## What's not shown

- The actual MCP `initialize` / `tools/list` JSON: standard MCP, not MID-specific.
- The capability hash construction: see `SPEC.md` §6.4.
- The WinUSB / MS OS 2.0 descriptor byte layout: see `SPEC.md` §9 and Microsoft's [MS OS 2.0 descriptors spec](https://docs.microsoft.com/en-us/windows-hardware/drivers/usbcon/microsoft-os-2-0-descriptors-specification). The HID-usage-page proposal works around this differently; see `../mcp-hid-usage-page` if Windows-driver-free is your top constraint.

## What to verify against the spec

If you're implementing MID, walk through this device with `SPEC.md` open and check:

- `bmCapabilities` bits in §5 match what your MCP server advertises in its `initialize` response.
- `descriptorCacheHash` is recomputed only when the capability set genuinely changes (don't bust caches on every boot).
- Framing matches §7 — short packets are not message boundaries; the 4-byte length is authoritative.
- BOS Platform Capability UUID matches §10 exactly; one bit wrong and no host will find you.
