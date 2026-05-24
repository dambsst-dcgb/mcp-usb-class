# mcp-usb-class

A proposal for a new USB device class — the **Model Interface Device (MID)** — that lets a USB device declare itself as an MCP server its host can discover, enumerate, and call without per-device drivers, vendor SDKs, or out-of-band metadata.

> Status: **Draft 0.1**, design-stage spec, pre–USB-IF submission. License [Apache-2.0](LICENSE). Companion proposal: [`mcp-hid-usage-page`](../mcp-hid-usage-page) (the lighter HID-based alternative).

## The pitch in two paragraphs

Today, an LLM host that wants to use a USB peripheral has to either ship a vendor driver, screen-scrape a HID interface, or be told out-of-band what the device can do. None of these scale to a world where any peripheral — a sensor, an actuator, a lab instrument, an embedded controller — should be addressable by an agent. The data path already exists (USB Bulk + JSON-RPC is fine); what is missing is **standardised, granular, agent-readable capability discovery** at the protocol level.

MID is a USB device class whose entire purpose is to expose an MCP server over the wire. Hosts that understand the class can enumerate every tool, resource, and prompt the device offers — by schema, not by name — and call them through standard MCP semantics. The device side reuses the existing MCP type system: no new schema language, no new method dispatcher. The USB side fits into the existing class architecture: composite-friendly, sits next to CDC / HID / MSC, no protocol abstraction layer.

## Why a new class (and why now)

A new class is the right shape because the *contract* between host and device is genuinely new — "this peripheral exposes itself as a model-callable surface" is not what CDC, HID, MSC, or PLDM were designed for. Repurposing HID via a usage page (see the companion repo) ships sooner but inherits HID's report-table model, which is awkward for MCP's request/response + notifications shape. A dedicated class lets discovery, framing, and the optional notification channel be expressed cleanly.

Now, because (a) MCP is converging on a stable wire format, (b) on-device LLM hosts and edge agents are starting to ship, and (c) the realistic standards arc is **prototype → open spec → MCP transport binding → USB-IF class**, and somebody has to write the open spec.

## What MID gives you

- **Tiered discovery with host caching.** A 32-byte enumeration descriptor lets the host decide whether to bind without claiming any interface. A control-transfer capability index gives a 4–8 KB table of contents even for devices exposing hundreds of tools. Full schemas are fetched lazily over MCP's own `tools/list` / `resources/list` / `prompts/list`. Per-item schema hashes let hosts cache and skip unchanged entries across reconnects.
- **MCP-native binding.** No abstraction layer between USB and MCP. Bulk-IN / Bulk-OUT carries JSON-RPC. Method names, error codes, capability flags, and lifecycle map 1:1 to MCP. Intentional coupling — when MCP iterates, MID iterates.
- **Composite-friendly.** A MID interface coexists with CDC, HID, or MSC interfaces on the same device. A keyboard can also be an MCP server. A logic analyzer can keep its existing vendor interface and add MID for agent access.
- **Optional notifications.** An Interrupt-IN endpoint can carry `notifications/*/list_changed` so dynamic tool sets (e.g. a device whose tools depend on what's plugged into it) don't need polling.
- **Pre-ratification discovery via BOS Platform Capability.** Until `bInterfaceClass = MID` is allocated, prototype devices declare themselves through a BOS Platform Capability descriptor carrying the MID UUID and `bcdMID`, while shipping with `bInterfaceClass = 0xFF` (vendor-specific). Hosts that understand MID find them; everything else ignores them. See [`SPEC.md` §10](SPEC.md#10-pre-ratification-discovery-bos-platform-capability).
- **No-driver-install on the three majors.** MS OS 2.0 descriptors bind the vendor interface to WinUSB on Windows. Linux and macOS already expose raw USB to user space. Hosts can talk to MID-class devices from user space, day one, with no installer.

## What this repo is, and what it isn't

This repo is the open **specification** and reference material for MID. It contains:

- [`SPEC.md`](SPEC.md) — normative draft of the class definition.
- [`comparison.md`](comparison.md) — MID vs. the HID-usage-page alternative, and vs. CDC, HID, MSC, PLDM, Thunderbolt VDM.
- [`examples/`](examples/) — a generic device sketch ([`examples/generic-device.md`](examples/generic-device.md)) and an nRF5340 reference ([`examples/nrf5340.md`](examples/nrf5340.md)).
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to file an issue, propose a normative change, contribute a reference implementation.

This repo is **not** a firmware library, a host SDK, or a certification suite. Those are downstream artifacts that should follow once the spec stabilises.

## Relationship to `mcp-hid-usage-page`

The two repos exist in parallel because they target different points on the **ambition × time-to-ship** curve. MID is the right long-term shape but takes USB-IF work to ratify. The HID usage page is a more compromised design (constrained by HID's report model) but can ship through the HID committee in much less time, and any device that already exposes a HID interface can adopt it tomorrow. See [`comparison.md`](comparison.md) for the full trade.

## Status & roadmap

- **0.1 (current)** — Design locked, spec drafted, no reference implementation.
- **0.2** — nRF5340 reference firmware in [`examples/nrf5340.md`](examples/nrf5340.md), end-to-end host probe on Linux + Windows.
- **0.3** — Second independent implementation (likely RP2040 or STM32), conformance checklist.
- **1.0** — Submission package for the USB-IF Device Working Group.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Spec issues and PRs welcome — particularly from anyone with prior USB-IF, HID-WG, or MCP-spec experience.

## License

[Apache-2.0](LICENSE). The specification text, examples, and any reference code in this repo are licensed permissively so vendors can adopt without legal friction.
