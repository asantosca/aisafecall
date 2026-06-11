# Demo screens

`index.html` is a self-contained mockup deck for the SafeCall demo — open it in any browser.
It shows the full product lifecycle from both points of view:

1. **Caregiver setup** — welcome, add parent, carrier-plan verification gate (pass + blocked variants), trusted-caller list, handoff to the parent's phone.
2. **Parent phone setup** — pairing, plain-language permissions, Android battery exemption, one-tap forwarding activation, verified-by-test-call.
3. **Elder daily use** — home screen, trusted incoming call, screened unknown caller, mid-call warning (medium confidence), auto-terminated scam call (high confidence), voicemail with spoken AI summary.
4. **Caregiver daily use** — dashboard with reachability heartbeat, live scam alert, voicemail triage, new-caller validation ("changed doctors" flow), allowlist management, heartbeat outage alert.
5. **Uninstall** — pause vs. remove, forwarding-off-before-delete sequencing, verified-off confirmation.
6. **iPhone vs Android** — the same incoming call on CallKit vs Telecom/ConnectionService, with the platform caveats that matter.

Elder screens use Atkinson Hyperlegible at large sizes, high contrast, and at most two actions per
screen; caregiver screens use standard density. Captions under each phone tie the screen back to
the decisions in `../project-brief.md`. "SafeCall" and all names/numbers are placeholders.

The `preview-*.png` files are rendered snapshots of each section for quick viewing without a browser.
