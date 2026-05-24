# Contributing

Thanks for looking. This is a pre-submission specification, so the contribution model is closer to a standards working group than a typical library repo.

## How to contribute

### Spec issues
File a GitHub issue. Label appropriately:
- `spec-normative` — proposes a change to MUST/SHOULD/MAY text in `SPEC.md`.
- `spec-editorial` — wording, typos, clarification of intent without changing the contract.
- `comparison` — corrections or additions to `comparison.md`.
- `example` — issues with `examples/`.
- `question` — open questions; may become a normative issue if a decision is needed.

For normative changes, include:
- The section / paragraph being changed.
- The proposed wording.
- The rationale (what breaks today, or what gets clearer / cheaper / safer).
- Any backward-compatibility implications (does this break v0.1 implementations?).

### Pull requests
PRs against `SPEC.md` should change exactly one normative concept per PR — easier to review, easier to revert. Editorial PRs may be bulkier.

Run the markdownlint-recommended ruleset before submitting. There is no CI yet; checks are by inspection.

### Reference implementations
If you have a working MID device or host, open an issue describing it. The plan is to enumerate known implementations in the README once there are at least two from different authors.

### Code of conduct
The Contributor Covenant 2.1 applies. Disagreement is welcome; disrespect is not.

## What we're particularly looking for

- **Independent implementation** — a second device beyond the nRF5340 reference. RP2040, STM32, ESP32-S3 all suit.
- **Host-side prototyping** — a small libusb-based prober that enumerates an MID device, reads Tier 1 + 2, then drives an MCP `initialize` over Bulk. This would be the natural first artifact in a downstream `mcp-usb-host` repo.
- **Review from USB-IF / HID-WG veterans** — particularly on the BOS Platform Capability layout and the IAD composition story.
- **MCP-side review** — anyone close to the MCP spec who can flag mis-mappings between the bitmap in §5 and current MCP capability semantics.

## What we're not looking for (yet)

- Premature optimisation of the wire format (CBOR vs. JSON for Tier 2 is open; let's not relitigate before there are two implementations).
- Adding extension points "in case." Every extension point is a future compatibility hazard. Bias toward `bcdMID` bumps over open-ended fields.
- Discussion of certification, branding, or test suites. Those follow ratification, not precede it.

## Decision-making

Until v1.0 freeze, the maintainers decide. After freeze, normative changes go through the same review the original spec did — and at that point this repo likely becomes the open mirror of a USB-IF working-group draft, with the actual normative work happening there.
