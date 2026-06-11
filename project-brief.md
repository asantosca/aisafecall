# Project Brief: Caregiver-Managed Scam-Call Screening for Elderly Parents

_Last updated: June 2026. Competitive figures (pricing, user counts, OS capabilities) reflect early/mid-2026 research and should be re-verified periodically. Twilio feasibility for the relay, real-time monitoring, in-app signaling, and attestation layers validated June 2026. Canadian carrier forwarding economics (Telus/Rogers allowances, prepaid exclusion) researched June 2026 from carrier support pages — re-verify per carrier before onboarding flows are built._

## One-line summary

A call-screening service that protects an elderly person from scam and spam calls, where the **end user is the parent** but the **buyer and manager is the adult child** acting as caregiver.

## The core split (drives everything)

- **User:** the elderly parent — receives the calls, can't always tell a real bank/family call from a scam.
- **Buyer / manager:** the adult child — feels the worry, sets things up, manages it remotely.
- **Sometimes a third role:** whoever physically sets up the phone.
  This buyer ≠ user dynamic is the central design and go-to-market fact.

## Technical concept (Twilio-based)

1. Install an app in the cell phone of the elderly person. That app will receive the call from Twilio as if the app was a telephone (Twilio Voice SDK client).
2. Set the cell phone to **unconditionally** forward all calls to Twilio. (Conditional/no-answer forwarding would let the call ring the parent directly first, defeating screening — so it must be unconditional.)
   So the public number stays public and unchanged.

**Consequence — the app is the sole delivery path.** Because forwarding is unconditional, Twilio cannot bridge an accepted call back to the parent's _own_ number (it would just forward back to Twilio in a loop), so a screened, accepted call reaches the parent **only** via the app (Voice SDK client, over the push/wake path below). This is clean, but it means app reachability gates whether the parent receives _live_ calls at all — see the next subsection.

### Waking the app (push-to-launch) and reachability

The relay only works if the parent's app is running when a call arrives, so the app must be wake-able even after it's been backgrounded or killed. The mechanism is the standard Twilio Voice SDK inbound-call path: a **VoIP push (PushKit) → CallKit** on iOS, and a **high-priority FCM data message → Telecom/ConnectionService** on Android. Twilio holds the push credentials and fires the push when a call hits the relay; the OS launches the app to handle it and the Voice SDK client answers the leg.

This is **not** the same as PagerDuty's "critical alert" push (the APNs critical-alert entitlement), which exists to break through silent mode / Do Not Disturb for an _alert_. For our purpose — relaunching the app to receive a _call_ — the VoIP push is the purpose-built and stronger tool: a VoIP push **relaunches a force-quit (swiped-away) app on iOS**, which an ordinary remote notification will not. The iOS 13+ rule that every VoIP push must immediately report a call to CallKit is satisfied here because we _are_ reporting a real incoming call.

**Residual hard-failure states (push cannot revive these):**

- **Android Force Stop** — if the user force-stops the app from Settings, all FCM delivery is blocked until the app is _manually_ reopened. No push can revive it.
- **OEM battery management** — Samsung, Xiaomi, Huawei, OnePlus, etc. aggressively kill backgrounded apps and silently break push delivery (the well-known "dontkillmyapp" problem). On an elderly parent's phone with no on-site caregiver, both states are live risks.
  **Architectural implication — app reachability gates _live_ calls, but the failure is graceful, not catastrophic.** If the app becomes unreachable, Twilio simply can't deliver the _live_ leg — but the caller is **not dropped**: Twilio takes them to voicemail server-side (entirely app-independent). So a dead app means calls go _unanswered_, not _lost_. Notification of the waiting voicemail behaves differently by platform: on iOS a standard "you have a voicemail" alert displays even after a force-quit; on Android **Force Stop** that push may not arrive until the app is reopened (FCM is blocked) — but the **caregiver's** alert rides their own device and arrives regardless, which is the reliable backstop. The residual gap is genuinely _live_ / time-sensitive / emergency calls during an outage, for which voicemail isn't equivalent. So:
- **Caregiver reachability heartbeat (not optional):** track app check-ins / push-token freshness and alert the caregiver when the parent's app has gone quiet ("we haven't heard from Mom's phone in 24h — live calls are going to voicemail"). Its job is to keep the outage window short, not to prevent dropped calls (there are none). Fits naturally into the caregiver-managed model.
- **Graceful Twilio fallback:** if the wake push goes unanswered within a timeout, route to voicemail (plus caregiver alert), so an unreachable app degrades to "reviewable voicemail," never to "calls vanish."
  _(Platform specifics — PushKit/CallKit relaunch-after-force-quit behavior, FCM force-stop behavior — are stable but worth a quick re-confirm against current Twilio Voice SDK docs before locking the architecture.)_

### Carrier-side forwarding economics (Canada) — plan verification is an onboarding gate

Unconditional forwarding rides the _parent's_ carrier plan, and Canadian carriers meter it. Researched June 2026 from carrier support pages:

- **Telus:** Call Forwarding North America includes **3,000 forwarded min/month** (Canada → CA/US number), then **75¢/min overage**. If the feature is **not in the rate plan**, forwarded minutes bill at pay-per-use rates. **No forwarding at all on Telus Prepaid.**
- **Rogers:** current 5G plans bundle **~2,500 call-forwarding minutes**; legacy plans vary and may not include it.
- **Bell and flankers (Koodo, Fido, Virgin, etc.):** assume the same pattern (included allowance on current postpaid, plan-dependent on legacy) until verified.
  **The allowance is comfortable; the conditional clause is the risk.** 3,000 min ≈ 100 min/day of _inbound_ talk — far above a typical elder's volume. Note that under the relay, Twilio answers _everything_, so spam that previously rang out unanswered now consumes forwarded minutes — but the challenge layer kills a robocall in under ~30s, so junk burns only a few minutes a day. The cap is not the constraint. Two things are:

1. **Legacy-plan pay-per-use.** The target demographic disproportionately sits on decade-old plans nobody has reviewed. A family that activates on a plan without included forwarding gets a surprise per-minute bill on the parent's next statement — a trust-destroying first impression for a brand selling trust. So **plan verification is a hard onboarding gate, not a FAQ entry**: confirm the feature (or add it / bump the plan) _before_ anything forwards. The forwarding cost is borne by the parent's plan — invisible to us, real to the family — so pricing copy should be transparent: "included in most current postpaid plans; we check yours during setup."
2. **Prepaid carve-out (TAM question).** Telus Prepaid cannot forward, and the pattern likely extends to the budget prepaid flankers where price-sensitive seniors live (Public Mobile, Lucky, Chatr). For that segment, onboarding becomes "migrate the parent's plan," not "dial a forwarding code" — a materially bigger ask. Quantify before build: add **"which carrier and plan is your parent on?"** to discovery interviews and the landing-page funnel; if a large share of the wedge population is on prepaid flankers, that's a finding to have _before_ committing.
   **Deliverable: a carrier compatibility matrix** — Telus / Rogers / Bell, then Koodo / Fido / Virgin, then the prepaid brands — with three columns: forwarding supported, included allowance, and the fix if absent. Small research task; slots naturally into the concierge pilot, where each family's setup gets audited by hand anyway.

### Screening layers

- **Allowlist** — known-good numbers, built automatically from the phone's **contacts** (with permission) and **call history** (numbers the parent has actually talked to repeatedly over time), with caregiver add/remove/edit. This is a _pre-computed_ decision, so no human has to act in real time. Three refinements:
  - **Verify, don't just match.** A bare number match is spoofable. Gate the trusted fast-path on **STIR/SHAKEN attestation** — Twilio exposes the `StirVerstat` parameter on inbound calls, where an "A" attestation means the originating carrier confirms the caller both is known and has the right to use that caller ID. Rule: _in-contacts AND valid attestation → fast-path; in-contacts but attestation missing/failed → treat as suspect (challenge + monitor)._ A targeted spoof of a trusted number typically won't carry valid attestation, which is what catches the case a bare allowlist misses. (US-only today; outside the US, fall back to allowlist + monitoring without the attestation signal.)
  - **Allowlist = skip the challenge, NOT skip monitoring.** A spoofed trusted number is the _only_ path a scammer has to reach the parent unchallenged, so allowlisted calls are the ones most worth keeping the AI listening to. Trusted → bypass the CAPTCHA, but simultaneous monitoring still runs.
  - **"Not in contacts" is not a block signal.** Legitimate first-time calls (the bank's fraud team, a new clinic, a hospital) come from unknown numbers — sometimes precisely when something is wrong. Unknown → challenge/whisper, never auto-block.
- **Challenge for unknown callers** — "Press the password to be connected." Kills autodialers and pre-recorded robocalls (a CAPTCHA for calls).
- **Whisper** — Get the call and start an AI to talk to it, assess urgency.
- **Voicemail** — graceful landing for everything that fails screening, _and_ the automatic fallback when the app is unreachable; reviewable by the family. **Enrich it:** transcribe the message, generate an AI summary, and attach a scam-probability signal, surfaced to the caregiver (and optionally the parent) — e.g. _"unknown caller, claims to be the bank's fraud team, urges an urgent callback — high scam likelihood."_ Because this is **async**, it sidesteps the real-time latency constraint that limits live monitoring; it runs only on messages that already reached voicemail (cost-gated); it's presented as a _signal, not a verdict_ (consistent with the rest of the system); and it recovers some scam-detection value even when live monitoring didn't run (e.g. the app was down). **Deliver the summary as spoken audio (TTS)**, not a block of text the parent has to read — see the _audio-over-text_ design default below.
- **Twilio Lookup** — bonus layer for line type / risk / caller-name signals; feeds the new-caller validation enrichment (see _Validating new / unknown legitimate callers_ below). Supplementary, not the backbone.
- **Simultaneous** — keep the AI listening even after the parent answers (including on allowlisted calls). If it detects an exploit, intervene. See _Simultaneous monitoring & real-time intervention_ below.

### Validating new / unknown legitimate callers (the "changed doctors" problem)

When a parent legitimately changes a doctor, bank, or pharmacy, the new number is in neither contacts nor call history, and the bare allowlist would route it to challenge/voicemail. We need a way to vet a genuinely-new legitimate caller _without trusting the inbound number_.

**Key distinction:** "whose number is this _supposed_ to be?" (directory/CNAM) vs. "is this call _actually_ from there?" (attestation/branded). A scammer spoofing the clinic's real number passes the first check and fails the second. No single service answers both — the clean reverse phone book is gone.

**Lookup-as-enrichment (produces a score, not a verdict).** Combine signals to help the caregiver decide, never to auto-admit:

- **CNAM / Caller Name** (Twilio Lookup) — closest descendant of the phone book, but US-only, often stale, and keyed to the _displayed_ number (so a spoof inherits it). Weak corroboration.
- **Reverse business-directory lookup** (Google Places / data aggregators) — does the number publicly list as a medical office? Corroborative, not authoritative, spoofable.
- **Line Type Intelligence** (Twilio Lookup) — legit business line vs. throwaway VoIP.
- **STIR/SHAKEN attestation** — the only signal that speaks to _not-spoofed_.
- **Branded Calling ID (BCID) / Branded Call Display** — the modern gold standard: STIR/SHAKEN-anchored vetted brand name/logo/reason, validated by a third-party vetting agent → identified AND not-spoofed. Catch: opt-in _outbound_ product the business must enroll in; coverage skews to large enterprises (big banks, hospital systems, government), so a small clinic that just switched numbers likely isn't enrolled. High-value when present, silent when absent.
- (Note: Lookup **Identity Match** is the wrong tool — it matches a number against identity data you already supply, it doesn't discover an unknown business.)
  **The actual safety guarantee is workflow, not lookup:**

1. **Capture at the moment of change.** The family _knows_ when they switch providers — make it a deliberate allowlist add (from the appointment confirmation, the provider's website, the insurance portal). A number is most trustworthy entered out-of-band at enrollment, not inferred mid-call.
2. **Graceful unknown + async caregiver confirm.** Unknown caller claiming to be a clinic → challenge/whisper/voicemail, then the caregiver gets "unknown caller said they're [X] re: [reason] — add to allowlist?"
3. **Callback to the published number.** The timeless anti-spoof move: never trust the inbound number; verify by calling the provider's _published_ number back. Product can operationalize this — surface the directory's listed number and offer a one-tap "verify by calling them back."
   Net: lookups become decision-support shown to the caregiver; the safety guarantee comes from out-of-band capture + callback to a published number, which sidesteps spoofing entirely.

### Simultaneous monitoring & real-time intervention

Keep listening while the parent is on the live call. Twilio Media Streams forks the call audio (`<Stream both_tracks>`) to our server in near real-time, running in parallel with the bridge that connects caller↔parent. An AI assesses the conversation as it happens; if it flags an exploit, we intervene.

**Architecture advantage — we own the in-call screen.** Because the call is relayed through _our_ app (Voice SDK client) rather than the native dialer, we control the screen during the call. We can push a real-time signal to the app via Twilio's Voice SDK Call Message Events (rides the existing signaling connection — no separate channel) and paint a warning, a hang-up button, or a confirmation prompt mid-call. The incumbents live in the native call UI and — on iOS especially — cannot draw custom UI over a live call. This is the same "iOS wall" they keep hitting, and our relay choice walks straight through it. A concrete, demoable edge tied directly to the wedge. (Constraint: SDK call messages are capped at ~10 KB and ~10/minute — fine for a flag, not a data channel.)

**Build scope — the detector is deferrable, the intervention is not.** Separate two things that are easy to conflate: the _detector_ (a production-grade real-time scam classifier on streaming audio — the expensive, accuracy-sensitive ML) and the _intervention_ (the in-call warning / audio alert / hang-up — the iOS-wall-walking UX no incumbent can do). The intervention **is** the core wedge capability and the demoable differentiator, so it ships from the start. What grows over time is the detector's _sophistication_ — and it's useful early without a fancy model: a high-precision keyword/phrase tier on the live transcript ("gift cards," "wire transfer," "don't tell anyone," "SSN suspended," "arrest warrant," "bail") covers the common scripts with tunable false positives, with LLM contextual judgment layered on as you scale. For a demo, trigger the intervention on a scripted signal — what you defer is detector _accuracy_, never in-call interception itself. (Live in-call detect-and-stop addresses the _connected, human-manipulation_ phase that voicemail triage structurally never sees; the two are complementary, and this is the wedge.)

**Audio over text (design default).** For this user, spoken beats on-screen. Lead interventions with an injected _audio_ warning ("this call may be a scam — hang up and call your bank on the number on your card") rather than relying on a banner an impaired user may not notice; the on-screen prompt is a supplement, not the primary channel. The same principle drives the TTS voicemail summary above.

**No human can be a real-time veto.** Neither the parent nor the caregiver can reliably receive, parse, and act on a live call in seconds — a caregiver may be asleep or driving, and a confused parent can't judge by definition. So any decision that must happen in real time has to be made by the system on pre-set logic. The caregiver's role is **asynchronous**: curate the allowlist up front, get alerted, review afterward, tune the rules.

**Tiered response by AI confidence** (match the intervention to certainty and to who can act):

- **Low:** silent caregiver notification, no disruption to the call.
- **Medium:** on-screen warning to the parent _plus_ caregiver alert.
- **High:** inject an audio warning and/or terminate (REST API: redirect to warning+`<Hangup>` TwiML, or set call status to `completed`), caregiver alerted regardless.
  **Default-inversion design question.** An on-screen prompt can be framed two ways:
- _"Press to hang up"_ — inertia favors the scammer (doing nothing keeps the call alive, which is the state a scammer engineers).
- _"Press to continue, else terminate"_ — inertia favors safety; the scammer's stalling works against them.
  The inverted default is the better instinct, but the affirmative action must NOT sit on the parent: (a) any action the parent can take, a live scammer can coach them through ("press the green button so we don't get cut off"), and (b) a false positive now drops a _legitimate_ call via a stressful timed interaction the impaired user may fail. The caregiver can't be a synchronous gate either (latency/availability). Resolution: keep elder-facing auto-terminate-on-inaction for **high-confidence flags only**, prefer "parent inaction → escalate to caregiver" over "inaction → hard drop," and lean on good _automatic_ defaults rather than any live human veto.

**Caveats (technical & product):**

- **Latency vs a live scammer.** Detection is audio→STT→LLM→decision→API, i.e. seconds. It cannot prevent the bad utterance; it's a circuit-breaker, not a shield. Strong against mass robocalls, partial against targeted human scams (the wedge).
- **False positives are uniquely costly here** — the parent can't override a wrong block by saying "no, this one's real," so bias toward warn/alert over auto-kill.
- **Cost** — streaming + STT + LLM per call; gate it to calls that already failed the allowlist/challenge layers.
- **Consent** — live monitoring is fine in one-party-consent jurisdictions (e.g. BC); some US states require two-party consent, so a disclosure may be needed depending on where the parent lives.

### Known limitation

- **Caller-ID spoofing — scoped, not eliminated.** An allowlist by number alone is spoofable, but the threat is narrower than it first appears. Mass scam operations don't know a given family's specific contact numbers, so for untargeted traffic the contacts allowlist holds. The residual risk is the **targeted** attack — a scammer who has researched the family and spoofs a known contact (daughter, bank) — which is precisely our wedge population. Mitigation: STIR/SHAKEN attestation gating the trusted fast-path (see Allowlist), plus monitoring that stays on for allowlisted calls. Note that the classic grandparent scam typically does _not_ spoof a known number — it calls from an unknown number claiming to be family — so contacts + suspicion-of-unknown already defends that case.

## Competitive landscape

**Same family (server/network screens the call):**

- **RoboKiller** — audio fingerprinting (resistant to spoofing because it analyzes call audio, not the originating number), sub-millisecond predictive blocking, community reporting, an AI screening layer for unknown callers, and "Answer Bots" that waste spammers' time. ~$4.99/mo or ~$40/yr, iOS + Android. Mature, well-funded incumbent.
- **Nomorobo** — "simultaneous ring": rings your phone and Nomorobo's server at once; server analyzes and drops robocalls, real calls keep ringing. Uses an audio CAPTCHA challenge. Built on Twilio. Historically VoIP/landline-oriented.
  **Different species (caller-ID / reputation engines, no call forwarding):**
- **Truecaller** — community spam database (450M+ users), AI pattern analytics, uses Apple's iOS 18.2 Live Caller ID Lookup (privacy-preserving) on iOS; fuller automatic access on Android.
- **Hiya** — behind-the-scenes engine powering AT&T Call Protect and Samsung Smart Call; 200M+ users, billions of analyzed calls, honeypots.
- Both have added **AI screening assistants** that answer unknown calls and ask who's calling — but the audio/recording side is largely **Android-only** because of Apple's restrictions (the recurring "iOS wall").
  **Native free tools:** Google Call Screen, Apple call screening — genuinely good at generic robocalls.

## The wedge

Incumbents have already solved **"block robocalls"** better than we realistically could. Competing there is a dead end. But they're all built as **self-protection tools for the phone's owner**, and that breaks down for our customer in three ways they don't address:

1. They assume the _user_ makes the final judgment — which fails for an elder who can't reliably tell real from fake.
2. They have **no caregiver model** — no adult child curating an allowlist, managing remotely, or receiving the "whisper" and deciding on the parent's behalf.
3. Audio fingerprinting catches **mass campaigns**; a one-off, human, targeted scam (grandparent-in-trouble call, a live impersonator) has **no fingerprint to match**.
   **The fundable angle:** The real angle is "protect someone who can't judge for themselves, managed by the family who worries".

**Architecture advantage tied to the wedge:** because screening happens in the network (Twilio relay) rather than on-device, this sidesteps the iOS restrictions that hobble the on-device incumbents. And because the call runs through _our app_ rather than the native dialer, we own the in-call screen — custom warnings, buttons, and confirmation prompts mid-call — which on-device iOS incumbents structurally cannot do.

**Refined target customer:** narrower and more desperate than "adult children generally" — families dealing with a parent's **cognitive decline / dementia** or an **elder-financial-exploitation** event (a scam that happened or nearly did).

## Customer discovery plan (current stage)

Goal: **try to kill the idea, not confirm it.** Cheapest possible way to learn the pain isn't there or the free tools are good enough.

**Principles**

- Separate buyer from user from setup-person — learn who actually owns the pain and who would act.
- Behavior over opinions. Worry is cheap and socially expected; _action_ is the only honest signal. The whole game is the gap between stated concern and what they've actually done.
  **Questions that work**
- "Walk me through your parent's phone — what do they have, who set it up, who manages it?"
- "Tell me about the last scam/suspicious call. How did you find out?"
- "Has anyone in the family lost money, or come close, to a phone scam?" (separates anxiety from real pain)
- "What have you already done about it, if anything? Did you set up Call Screen? What prompted it, and how's it going?"
- "Of everything you worry about with your parent, where do scam calls rank?"
  **Questions to avoid** (they mislead)
- "Would you use an app that screens calls?"
- "Do you think scam calls are a problem?"
- "How much would you pay?"
  **Listen for:** existing workarounds (any rigged-up solution = real pain); whether free tools are "good enough" _in their words_ and whether anything bad slipped through; a sharper sub-problem hiding inside the broad one.

**Free signal before interviews:** read reviews of existing tools (Truecaller, RoboKiller, Nomorobo, Hiya, native) and lurk in eldercare/caregiver communities — unprompted complaints about what those tools _miss_ are unbiased market language.

## Reaching the adult children (GTM thinking)

Buying happens at **trigger moments**: a scam loss or near-miss, a dementia/cognitive-decline diagnosis, a holiday visit where the parent seemed frailer.

**Channels (clustered by the refined target)**

- **Communities where the pain lives:** dementia/Alzheimer's caregiver groups (Alzheimer's Association forums, r/dementia, r/AgingParents), elder-fraud channels (AARP Fraud Watch Network). Also the recruiting ground for discovery interviews.
- **High-intent search:** not "call blocker" but "how to stop scammers calling my mom with dementia," "protect elderly parent from phone scams." Incumbents under-optimize for this.
- **Financial-protection intermediaries** (fits the wedge better than the usual healthcare channel): banks/credit unions, elder-law attorneys, financial advisors managing an aging client's money, geriatric care managers — all present when a family discovers a loss.
  **Messaging**
- Lead with the **concrete financial-loss fear** (money lost to a live impersonator), not vague "safety" — real dollars convert better than background worry.
- Pair with the **dignity frame** the product structurally supports: _you_ curate who can reach Mom, the whisper preserves her agency, you manage it remotely without taking her phone or making her feel surveilled.
- For interviews specifically: the ask is "can I hear your story," not a pitch — and keep listening for action over stated worry.

## Resolved decisions

- **Forwarding, not porting (porting rejected).** Use unconditional call-forwarding to Twilio rather than porting the parent's number in. Porting is the wrong primitive: it would force the parent's physical phone onto a _new_ underlying number, drag in LOA / losing-carrier paperwork (and, depending on ambition, carrier licensing to become a number-mover at scale), and take ≥2 days with the number in limbo — the opposite of what a panic-trigger purchase needs. Forwarding sets up in minutes and leaves the public number unchanged. Onboarding velocity is matched to how the customer actually buys. _(Twilio absorbs the port-in carrier role, so the blocker is paperwork/delay, not a license — but porting still loses on every axis here.)_
- **Unconditional forwarding ⟹ the app is the sole live-delivery path.** A consequence of the above (see Technical concept): an accepted call can't be bridged back to the parent's own number without looping, so it reaches the parent only via the app. App reachability therefore gates _live_ calls.
- **App-unreachable degrades gracefully, not catastrophically.** A killed/blocked app means live calls go _unanswered_, not _dropped_ — Twilio takes the caller to voicemail server-side and a push prompts review (reliably to the caregiver; to the parent on iOS even post-force-quit, but not on Android Force Stop). The reachability heartbeat exists to keep the _live-call_ outage window short, since voicemail isn't equivalent for an emergency.
- **Voicemail is an active triage surface, not just a mailbox.** Transcribe + AI-summarize + attach a scam-probability signal, delivered as spoken audio (TTS). Async, so no live-latency constraint; cost-gated to messages that reached voicemail; a signal, not a verdict.
- **Audio over text (design default).** For an elderly/impaired user, spoken output beats on-screen text — applies to the voicemail summary (TTS) and to in-call warnings (injected audio leads; on-screen banner supplements).
- **MVP scope — in-call interception is core; detector sophistication is deferrable.** Keep live detect-and-stop central (it's the wedge: the connected human-manipulation phase voicemail never sees). What's deferrable is the _accuracy_ of the real-time detector — start with a high-precision keyword tier, layer LLM judgment as you scale; for the demo, trigger the intervention on a scripted signal.
- **Carrier plan verification is a hard onboarding gate.** Forwarding is metered on the _parent's_ plan (Telus: 3,000 min/month included if the feature is in the plan, 75¢/min overage, pay-per-use if not, no forwarding on prepaid; Rogers: ~2,500 min on current plans). The included allowances are comfortable; the risk is legacy plans without the feature (surprise per-minute billing — trust-destroying) and the prepaid carve-out (Telus Prepaid and likely Public Mobile / Lucky / Chatr can't forward at all, turning onboarding into a plan migration). Verify-or-fix the plan _before_ activation; be transparent that the forwarding cost rides the parent's plan.

## Open items / next steps

1. Run discovery interviews to validate caregiver-managed pain vs. robocall pain.
2. **Spoofing mitigation (approach decided, sub-questions open).** Gate the trusted/allowlist fast-path on STIR/SHAKEN attestation (`StirVerstat`), keep simultaneous monitoring running on allowlisted calls, and treat "unknown number" as challenge-worthy rather than blockable. Open sub-questions: behavior outside the US (no attestation framework yet); how to handle attestation "B"/"C"/missing without over-blocking legitimate calls.
3. Confirm the network-screening / iOS-wall advantage holds up technically. _(Relay, Media Streams monitoring, in-app Call Message Events signaling, and STIR/SHAKEN verification all confirmed available on Twilio as of June 2026.)_
4. **Keep the app-reachability outage window short.** A killed/force-stopped/battery-managed app means _live_ calls go to voicemail unanswered (not dropped). Confirm VoIP-push/CallKit relaunch and FCM behavior on real devices/carriers, build the caregiver reachability heartbeat, and define the wake-timeout → voicemail fallback. Open sub-question: detecting and surfacing the Android Force-Stop state, where even the parent's "check voicemail" push won't arrive until the app is reopened.
5. **Decide automatic-intervention defaults.** No human can be a real-time veto, so real-time decisions run on pre-set logic. Define confidence tiers, the warn-vs-terminate thresholds, and the "press to continue" / escalation behavior. Tune false-positive handling — the parent can't override a wrong block, so the cost of a false positive is asymmetric.
6. **Build the new-caller validation flow** (changed-doctor problem). Wire up the enrichment lookups (CNAM, business-directory, line type, attestation, BCID where present) as caregiver decision-support, plus the workflow guarantee: capture-at-change, async caregiver confirm, and one-tap callback-to-published-number. Open sub-questions: which directory data source(s) and their licensing/cost; how to present a confidence score without implying certainty.
7. **Build the carrier compatibility matrix and quantify the prepaid carve-out.** Verify Bell and the flankers (Koodo, Fido, Virgin) plus the prepaid brands (Public Mobile, Lucky, Chatr) for: forwarding supported, included allowance, fix if absent. Add "which carrier and plan is your parent on?" to discovery interviews and the landing-page funnel — if a large share of the wedge population is on prepaid flankers, the onboarding story changes materially. Slots into the concierge pilot's per-family setup audit.
8. Re-verify competitive specifics (pricing, user counts, OS framework capabilities) before any major decision.
