# MID vs. the alternatives

Why a new class, and how it relates to the obvious alternatives.

## MID vs. `mcp-hid-usage-page` (the companion proposal)

Both proposals solve the same problem — granular, agent-readable USB capability discovery — at different points on the **ambition × time-to-ship** curve.

| | **MID (this repo)** | **HID usage page (companion)** |
|---|---|---|
| Standards path | New USB device class via USB-IF Device WG | New HID usage page via HID WG |
| Time-to-ratify (estimate) | 18–36 months | 6–12 months |
| Composite story | First-class; MID interface sits next to any other interface | Already first-class — HID is the most-composited class in practice |
| Discovery overhead | 32 B at enumeration, then control-transfer index | Constrained by HID report descriptor format |
| Data path | Bulk-IN/OUT, JSON-RPC, length-prefixed | HID feature reports (small) or a secondary endpoint (bigger payloads) |
| Notifications | Optional Interrupt-IN | HID input reports (always available) |
| Schema density | High (JSON over Bulk is unconstrained) | Lower (HID reports are short and structured) |
| Driver story (Windows) | WinUSB via MS OS 2.0 | HID driver, already present |
| Risk of having to redo it | Higher — fresh design surface, more to get wrong | Lower — bolted onto a stable substrate |
| Reach if shipped | Any new MID device | Any existing HID device, retrofit-friendly |

The short version: **HID-usage-page is the conservative bet, MID is the right long-term shape.** Shipping both is deliberate. If HID-usage-page ratifies first and gets adopted, MID can either follow as the next-generation cleanup or merge in.

## MID vs. CDC (Communications Device Class)

CDC was designed for serial-modem semantics: a byte stream with control-line metadata. Vendors today abuse CDC as "USB serial that any OS can open," then layer their own protocol inside. That works, but it gives the host nothing structured — the host learns "this is a serial port," nothing more. There is no place to declare *what speaks back* on the port, let alone in machine-readable form.

MID is exactly the missing layer above CDC's "I am a port." CDC could be used as MID's transport (instead of Bulk), but it adds Notification element / line-state semantics that MID doesn't need, and removes the ability to length-prefix cleanly.

## MID vs. HID (Human Interface Device)

HID is famously flexible — a report descriptor can describe almost any input/output topology. Vendors who want capability discovery sometimes graft it onto HID feature reports today. The companion `mcp-hid-usage-page` repo formalises that approach.

The reasons MID doesn't just live inside HID:

- HID's report-descriptor model assumes the host parses the device's reports as typed data, not as messages with bodies. JSON-RPC over HID feature reports works but is awkward — you end up using HID as a length-prefixed channel, ignoring the report-descriptor type system, which is most of why HID exists.
- HID input reports are great for notifications but cap out around 1 KB per report on Full-Speed. Anything larger needs chunking.
- HID has hard-coded host expectations (the Windows HID stack, in particular, will second-guess unusual report structures). A new HID usage page can sidestep some of this; a new class sidesteps all of it.

If your device is already a HID, the HID-usage-page proposal is the right call. If you are designing fresh, MID is cleaner.

## MID vs. MSC / SCSI

MSC (Mass Storage Class) over SCSI has a discovery model (INQUIRY, READ CAPACITY, etc.) but it is fundamentally about blocks. Some vendors expose configuration via tunnelled SCSI commands; this scales badly and ties capability discovery to the SCSI command set's evolution, which is glacial.

## MID vs. PLDM (Platform Level Data Model)

PLDM is the natural comparison for "structured, machine-readable, vendor-neutral discovery." It is a DMTF standard, widely deployed in server BMC ecosystems, and has type registries for everything from firmware update to monitor & control. It runs over MCTP, which can run over USB.

Reasons MID is not just "PLDM types for MCP":

- PLDM's type model is designed for predictable, low-churn capability sets in tightly-controlled deployments (BMCs, FPGAs, datacenter peripherals). MCP's tool / resource / prompt model assumes the capability set can change at runtime and is described by JSON Schema, not a numeric type code. Reconciling these models is harder than writing a fresh binding.
- PLDM's reach is real but narrow — it is not present on consumer/maker hosts, has no Windows/macOS user-space story, and requires an MCTP stack on the device.
- The MID + MCP coupling is the point. Adopters get MCP semantics on the device with no protocol abstraction layer to debug.

PLDM remains the right answer for datacenter telemetry. MID is the right answer for "this peripheral exposes itself to an agent."

## MID vs. Thunderbolt / USB4 vendor-defined messages

Thunderbolt and USB4 have VDM (Vendor-Defined Message) carriers, and TBT3 in particular has been used for some device-discovery experiments. VDMs live a layer below the USB class model — they are useful for cable-and-controller-level negotiation, not for application-layer capability discovery. Different problem.

## MID vs. mDNS / DNS-SD on USB NCM

A workable approach for richer devices is to expose USB NCM (network class), run a microservice, and discover it via mDNS. Several vendors do this. It works, but:

- Requires an IP stack on the device. nRF5340 can carry it; many cheaper SoCs can't.
- The host has to dynamically add a network interface and trust the device on it.
- Discovery is good only as fine-grained as the listening service exposes — back to the original problem.

MID does in 32 bytes of enumeration descriptor what mDNS-on-NCM does after the device has booted a TCP/IP stack.

## When *not* to use MID

- **Pure block storage.** MSC exists; use it.
- **Pure keyboard / mouse / gamepad.** HID is fine. Add an MID interface only if you genuinely want agent-level capability discovery in addition.
- **Webcams.** UVC is fine. Same caveat.
- **Devices that already have a stable, well-adopted vendor SDK and serve a market that doesn't care about agent integration.** Don't ship MID for the sake of shipping it.

## When MID is the right call

- Lab instruments, test equipment, programmable controllers — anything where "what can this device do?" doesn't reduce to a fixed protocol.
- Sensor packs and IoT bridges where the tool set changes with what's wired up.
- Educational and maker hardware where students should be able to point an agent at the board and have it work.
- Anything that today ships a vendor SDK that wraps a USB serial port.
