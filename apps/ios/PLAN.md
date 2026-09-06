# iOS Chat Visual Polish Plan

Investigation-only. No visual code ships in this document. The goal is a
scoped, upstream-ready path to make Chat feel more native on iOS without
becoming a Messages clone or fighting OpenClaw's design language.

**Audience:** implementers preparing one or two small PRs for
`ymolodtsov/openclaw-ios`, then a follow-up upstream PR to
`openclaw/openclaw`.

**Scope of this pass:** `apps/ios` Chat/Design hosts, plus the shared canvas
they actually render (`OpenClawChatUI`). Bubbles, transcript, and composer
are not owned by `apps/ios/Sources/Chat`.

---

## 1. Chat UI architecture

iOS Chat is a thin host around a shared SwiftUI canvas. Changing only
`apps/ios/Sources/Chat` cannot polish bubbles or the composer.

```text
RootTabs / RootSidebar
        │
        ├── Chat tab ──────── ChatProTab (iOS host)
        │                         │
        │                         ├── toolbar: agent identity, gateway dot,
        │                         │   expandable status, actions popover
        │                         └── OpenClawChatView (shared canvas)
        │                                   ├── transcript (LazyVStack)
        │                                   ├── empty / loading / error overlays
        │                                   ├── working claw + streaming bubble
        │                                   └── OpenClawChatComposer (.clean)
        │
        └── Control / sidebar ── CommandCenterTab + RootSidebar
                                  └── CommandSessionRow (session list)
```

### iOS host (`apps/ios`)

| Surface | File | Role |
|---|---|---|
| Chat tab | `Sources/Design/ChatProTab.swift` | Builds `OpenClawChatView` with `composerChrome: .clean`, hides avatars, opts into grey assistant bubbles, owns toolbar/gateway chrome, starter prompts, and the `Preparing Chat` fallback |
| Chat actions | same file + `Sources/Design/ChatModelControlsMenu.swift` | Ellipsis popover: new session, model, reasoning toggle, export |
| Gateway status | `ChatProTab` + `Sources/Status/GatewayStatusBuilder.swift` | Compact avatar dot; expands when unhealthy or tapped |
| Session list | `Sources/Design/CommandCenterTab.swift`, `CommandCenterSupport.swift`, `Sources/RootSidebar.swift` | Recent sessions and sidebar navigation. Chat tab sets `showsSessionSwitcher: false` |
| Session dashboard | `Sources/Chat/SessionDashboardScreen.swift` | Control UI WebView, not the transcript |
| Transport | `Sources/Chat/IOSGatewayChatTransport.swift` | Gateway history/send/sessions; not visual |
| Design tokens | `Sources/Design/OpenClawProComponents.swift`, `OpenClawBrand.swift`, `OpenClawTypography.swift` | iOS chrome tokens. Chat canvas uses `OpenClawChatTheme` instead |
| Design rules | `DESIGN.md` | Native containers, Liquid Glass on chrome only, semantic status colors, Dynamic Type / Reduce Motion / 44pt targets |

`ChatProTab` already documents two intentional iOS choices:

```swift
// iMessage-style grey bubbles for agent replies in the clean chrome.
.environment(\.openClawAssistantBubblesInCleanChrome, true)
showsAssistantAvatars: false
composerChrome: .clean
```

### Shared canvas (`apps/shared/OpenClawKit/Sources/OpenClawChatUI`)

| Surface | File | Role |
|---|---|---|
| Canvas shell | `ChatView.swift` | `OpenClawChatView`: transcript, overlays, jump-to-latest, keyboard live-edge restore |
| Bubbles | `ChatMessageViews.swift` | `ChatMessageBubble`, `ChatBubbleShape`, typing/streaming/pending-tool bubbles, outbox footer |
| Transcript rows | `ChatTranscriptRows.swift` | Message / system notice / history divider |
| Composer | `ChatComposer.swift`, `ChatComposer+CleanControls.swift`, `CleanChatComposerControls.swift`, `ChatComposerTextViewIOS.swift` | Clean iOS composer: glass/material field, plus / permissions, model + effort, mic↔send swap |
| Theme | `ChatTheme.swift` | iOS canvas gradient, red user bubble, `systemGray5` assistant bubble |
| Session sheets | `ChatSheets.swift`, `ChatSessionSidebar.swift`, `ChatSessionManagementViews.swift` | Shared session UI (macOS switcher, iOS new-session options) |
| Working state | `ChatWorkingClawView.swift`, `ChatStreamingReveal.swift` | Branded claw typing; word-fade streaming (Reduce Motion aware) |
| State | `ChatViewModel.swift` and extensions | `isLoading`, errors, outbox, streaming, tool activity |

### Current iOS Chat configuration (deliberate)

- Clean composer, no in-chat session picker.
- No assistant avatars in the transcript (identity lives in the toolbar).
- Bubble tails exist only for `style == .onboarding`, not standard Chat.
- Mic while the draft is empty, send once there is text (`cleanTrailingControl`).
- Working indicator is the OpenClaw claw, not Messages three-dots.
- Streaming already fades words in (`ChatStreamingReveal`).
- Jump-to-latest is already gated so it does not flash on every reply.
- Offline outbox is a product feature: placeholder becomes
  `Message {agent}; sends when connected`.

---

## 2. Punch list

Ordered by **impact / risk**. Each item is one implementable unit.

Motion rules for every animation item:

- Gate with `@Environment(\.accessibilityReduceMotion)`.
- Prefer `.snappy` (system) or `0.18...0.24s` ease-out. No bounce, no spring
  overshoot on layout.
- Do not animate markdown reflow, streaming token paint (already owned), or
  scroll-to-bottom (already a nil-animation transaction).

### P1. Unify empty / loading / error / preparing surfaces

**Impact:** high. **Risk:** medium.

**What.** Chat currently has four competing empty-ish surfaces:

1. `ChatProTab` `ContentUnavailableView("Preparing Chat")` while `viewModel == nil`.
2. `ChatLoadingBubble` ("Loading chat") when clean chrome is loading and the
   transcript is empty.
3. `ChatNoticeCard` / `ContentUnavailableView("Chat", "Type a message below to start.")`
   when empty, not loading, and the intro is hidden.
4. `ChatAssistantIntroCard` (grey bubble + starter chips) when connected (or
   offline-queueable) and empty.

The intro is suppressed unless `isComposerEnabled` is true
(`visibleEmptyAssistantIntro`). A reconnect or first attach therefore flashes
Preparing → Loading capsule → generic "Chat" → intro. Disconnected empty
sessions drop to the generic card even though the toolbar already says Offline.

**Why.** This is the main "weird UI state." Messages keeps the thread chrome
stable and only swaps content. OpenClaw should keep the canvas and composer
mounted and change one presentation, not four.

**Files.**

- `apps/ios/Sources/Design/ChatProTab.swift`
- `apps/shared/OpenClawKit/Sources/OpenClawChatUI/ChatView.swift`
- Existing proof: `apps/ios/Tests/GatewayStatusBuilderTests.swift`
- Add focused presentation tests next to
  `apps/shared/OpenClawKit/Tests/OpenClawKitTests/ChatReaderScrollStateTests.swift`

**Approach.**

1. Extract a pure presentation enum, e.g. `ChatSurfacePresentation`:
   `preparing`, `loading`, `emptyIntro`, `emptyUnavailable`, `error`,
   `transcript`.
2. Drive it from already-public facts: `viewModel == nil`,
   `viewModel.isLoading`, `showsEmptyState`, `activeErrorText`,
   `isComposerEnabled`, `hasVisibleMessageListContent`.
3. Keep the last good transcript on screen while refreshing. Never cover a
   non-empty thread with `ContentUnavailableView`.
4. Keep the composer mounted in every state except true `preparing` (no
   view model yet). Prefer shrinking `preparing` to a short first-attach only.
5. Cross-fade overlays with:

```swift
.animation(.snappy(duration: 0.22), value: presentation)
.contentTransition(.opacity)
```

   Use `Transaction(animation: nil)` when swapping cached transcript → live
   transcript so messages do not fade as a block.

**Acceptance.**

- Cold launch with a cached session: no `Preparing Chat` flash if a view
  model can be built from stored routing identity.
- Empty connected session: only the intro bubble + chips, never the generic
  "Chat / Type a message below" card.
- Empty disconnected session: one unavailable state that names the next
  step (Connect / Settings). Composer stays visible with the existing
  `Connect to a gateway` placeholder.
- Loading a session that already has messages: spinner or inline capsule
  only if there is no cached transcript; no full-canvas replacement.
- Error with messages: banner only (`ChatNoticeBanner`). Error with no
  messages: one `ContentUnavailableView` plus Refresh.
- Reduce Motion: opacity only, no slide.

### P2. Stop full-canvas remounts on session / owner change

**Impact:** high. **Risk:** medium-high (lifecycle / outbox pinning).

**What.** `ChatProTab` does `.id(ObjectIdentifier(viewModel))` on
`OpenClawChatView`. Any view-model rebuild (gateway owner, transport agent,
unpin after voice-note) destroys the scroll view, composer focus, and
keyboard avoidance, then runs `Preparing`/`load()` again.

**Why.** Session switches and reconnects feel like the tab was recreated.
Messages keeps the thread container and replaces rows.

**Files.**

- `apps/ios/Sources/Design/ChatProTab.swift` (`syncChatViewModel()`,
  `makeChatViewModel()`, `.id(...)`)
- `apps/shared/OpenClawKit/Sources/OpenClawChatUI/ChatView.swift`
  (`onChange(of: viewModel.sessionKey)`)

**Approach.**

1. Treat `.id(ObjectIdentifier(viewModel))` as guilty until proven required.
   Prefer `viewModel.syncSession(to:)` (already used when owner is unchanged).
2. If a rebuild is required (owner/agent change), keep the `OpenClawChatView`
   identity stable and pass the new model in, or animate only the transcript
   stack.
3. Preserve `isAttachmentOwnerPinned` behavior. Do not "smooth" a pin by
   showing the wrong agent's messages.
4. After a session key change, keep the existing nil-animation live-edge
   restore. Do not add a fly-in of the whole history.

**Acceptance.**

- Switching sessions from Command Center / sidebar does not show
  `Preparing Chat`.
- Composer text focus is lost only when the session actually changes, not
  when gateway metadata refreshes.
- Pinned voice-note / attachment owner still blocks a visual session steal.
- Cached transcript paints before `isLoading` flips true on a warm session.

### P3. Subtle bubble insertion and send-control motion

**Impact:** high. **Risk:** low-medium.

**What.** New user bubbles and the working-claw row appear with no insertion
transition. The mic→send swap is already the right iOS pattern but it pops.

**Why.** A short insert is what makes Chat feel native. Tails, blue bubbles,
and three-dot typing would make it a clone and fight the brand.

**Files.**

- `ChatView.swift` (`messageRow`, `showsWorkingIndicator`)
- `ChatMessageViews.swift` (only if a shared `chatRowInsertion` helper is
  needed)
- `ChatComposer.swift` (`cleanTrailingControl`, `sendButton`)
- `ChatHaptics.swift` already fires `.messageSent` — keep it

**Approach.**

```swift
// Transcript rows, iOS + clean chrome only
.transition(.asymmetric(
    insertion: .scale(scale: 0.98, anchor: isUser ? .bottomTrailing : .bottomLeading)
        .combined(with: .opacity),
    removal: .opacity))

withAnimation(reduceMotion ? nil : .snappy(duration: 0.20)) { /* send */ }

// Mic ↔ send
Image(...).contentTransition(.symbolEffect(.replace))
```

Do **not** enable `ChatBubbleShape` tails on standard Chat. Tails are
onboarding-only today (`bubbleTail`).

Do **not** restyle user bubbles to system blue. User fill is
`userAccent ?? OpenClawChatTheme.userBubble` (OpenClaw red / gateway accent).

**Acceptance.**

- First user send: bubble eases in from the trailing edge; composer does
  not jump the transcript by more than the new row height.
- Working claw appears with the same insert, then streaming bubble replaces
  it without a second bounce.
- Mic/send swap uses a symbol replace; hit target stays 44pt
  (`CleanChatComposerMetrics.controlTouchSize`).
- Reduce Motion: instant swap, no scale.
- macOS full chrome unchanged unless the helper is a no-op there.

### P4. Composer and accessory height jumps

**Impact:** medium-high. **Risk:** medium.

**What.** Clean composer uses a 104pt `restingMinHeight`. Attachments, reply
preview, talk strip, capability notices, progress card, turn recap, and
swarm each insert into the `VStack` above the field with no height
animation. Keyboard show already restores live-edge; the extras still shove
the transcript.

**Why.** Messages grows the input bar in place. Sudden 40–80pt jumps feel
like a custom control.

**Files.**

- `ChatView.swift` (`content` VStack: progress, recap, swarm, composer)
- `ChatComposer.swift` (`cleanComposerCard`, `composerContextRows`)
- `CleanChatComposerControls.swift` (`CleanChatComposerSurface`, metrics)
- `ChatTalkActivityViews.swift`, `ChatReplyPreview.swift`,
  `ChatProgressCard.swift`

**Approach.**

1. Animate composer height with `.animation(.snappy(duration: 0.22), value:)`
   on attachment count, reply target, recording, and talk-enabled.
2. Prefer `safeAreaInset(edge: .bottom)` for the composer on iOS so the
   transcript resizes instead of the whole column reflowing. Only do this if
   a prototype shows no conflict with `scrollEdgeEffectStyle(.soft)` in
   `ChatProTab`.
3. Keep Liquid Glass / material on the composer surface. Do not put glass
   on bubbles (`DESIGN.md`).
4. Leave `ChatComposerTextViewIOS` sizing as the single height owner; wrap
   it, do not replace UITextView with `TextField`.

**Acceptance.**

- Adding a photo: strip appears with a height animation; field stays
  first-responder.
- Reply preview and talk strip do not uncover the last bubble without a
  live-edge follow when the user was already at the bottom.
- Empty → one-line → multiline draft does not flicker the footer controls.
- iOS 18 material fallback and iOS 26 `glassEffect` remain equivalent in
  label, hit target, and contrast.

### P5. Gateway connecting chrome without toolbar reflow

**Impact:** medium. **Risk:** low.

**What.** Unhealthy states always expand a "Connecting" / "Offline" /
"Attention" capsule beside the agent name (`gatewayStatusShouldExpand`).
That is a good product choice. The expansion currently swaps the agent
title for the capsule (`headerAgentIdentityLabel`) and retouches toolbar
width.

**Why.** A connecting flash that shoves the session title is the toolbar
version of the canvas flash.

**Files.**

- `apps/ios/Sources/Design/ChatProTab.swift`
- `apps/ios/Tests/GatewayStatusBuilderTests.swift`

**Approach.**

1. Keep auto-expand on non-success. Do not hide connecting.
2. Reserve a stable trailing slot or overlay the capsule so the agent
   name does not disappear on a 300ms connect.
3. Pulse the existing 13pt avatar dot with
   `phaseAnimator` / `symbolEffect(.pulse)` only while `.connecting`,
   disabled under Reduce Motion.
4. Do not add a second connection banner above the transcript unless the
   canvas is empty (then P1's `emptyUnavailable` is enough).

**Acceptance.**

- Healthy → connecting → healthy: title width does not jump; dot color
  follows `OpenClawBrand.status*`.
- Tap-to-expand on a healthy gateway still uses the existing 0.24s snappy
  animation.
- Voice waveform on the avatar is unchanged.

### P6. Native-ize jump-to-latest and in-thread error banner

**Impact:** medium. **Risk:** low.

**What.** Jump-to-latest is a 36pt custom circle + shadow.
`ChatNoticeBanner` is a one-off card; iOS already has
`OpenClawNoticeBanner` in `OpenClawProComponents.swift`. DESIGN.md tells
hosts to reuse that banner.

**Files.**

- `ChatView.swift` (`jumpToLatestButton`, `ChatNoticeBanner`)
- `apps/ios/Sources/Design/OpenClawProComponents.swift`
  (`openClawGlassButton`, `OpenClawNoticeBanner`)

**Approach.**

- Jump button: `openClawGlassButton` / `OpenClawGlassControlGroup` on iOS
  26, keep the current material circle on iOS 18. Keep the ~44pt tap
  padding. Existing `.move(edge: .bottom).combined(with: .opacity)`
  transition is fine.
- Banner: either reuse `OpenClawNoticeBanner` with a chat-sized
  configuration, or restyle `ChatNoticeBanner` to the same radius/type
  tokens. Do not invent a third notice.
- Animate banner insert with `.move(edge: .top).combined(with: .opacity)`.

**Acceptance.**

- Jump control looks like other chat chrome (actions, sidebar reveal), not
  a floating game HUD.
- Banner uses semantic orange/red from the existing error presentation
  helper, not a decorative tint.
- Light/dark and Dynamic Type still fit a compact phone width.

### P7. Soft consecutive-message grouping (no tails)

**Impact:** medium. **Risk:** medium.

**What.** iOS transcript spacing is a flat 12pt (`Layout.messageSpacing`).
Same-role consecutive messages read as isolated cards.

**Why.** Messages tightens same-sender runs. Doing that with spacing only
(not tails, not hide-timestamp grouping) stays OpenClaw.

**Files.**

- `ChatTranscriptRows.swift` (add `isGroupedWithPrevious` or equivalent)
- `ChatView.swift` (`messageListRows` / `messageRow`)
- `ChatMessageViews.swift` only if grouped rows need slightly less
  vertical padding (10/12 → 8)

**Approach.**

- Same role, no divider/notice between, within a short time window if
  timestamps are already on the model: 4–6pt gap.
- Role change or system row: keep 12pt.
- Still no tails on standard Chat.

**Acceptance.**

- Two user messages in a row look related; an assistant reply after them
  still has the larger gap.
- Search highlight and context-menu targets stay on the full bubble.
- VoiceOver order unchanged.

### P8. Session list: leave Command Center as Command Center

**Impact:** low for "chat feels native." **Risk:** medium if restyled.

**What.** Session picking is a Control-tab card list
(`CommandSessionRow` in `ProCard`s) plus the black sidebar. It is not a
Messages conversation list, and that matches `DESIGN.md` (sidebar +
grouped cards).

**Suggested approach.** Out of the first two PRs. If a later pass happens,
limit it to: stable empty/offline row (already exists), no layout jump
when the 200-session fetch arrives, and `List`/`Form` only if Command
Center is already moving that way. Do not skin it as Messages.

---

## 3. What not to change

These look like product decisions, not leftovers.

| Leave it | Why |
|---|---|
| OpenClaw red / coral accent, carapace palette | Brand. `OpenClawBrand` + `OpenClawChatTheme.userBubble` |
| `OpenClawType` / `OpenClawChatTypography` | iOS AGENTS.md: no bare system fonts on user-visible text |
| Working claw + mascot | Brand motion; already Reduce Motion aware |
| Streaming word-fade | Already implemented and tested |
| Assistant avatars off in Chat tab | Identity is the toolbar waveform badge |
| Bubble tails only in onboarding | Standard tails would clone Messages |
| Grey assistant bubbles in clean chrome | Already the iOS-native choice |
| Canvas gradient (`OpenClawChatTheme.background`) | Deliberate vs `systemGroupedBackground`. Do not "fix" to grouped grey in these PRs |
| Liquid Glass only on chrome / composer | `DESIGN.md`. No glass on bubbles, cards, or markdown |
| Session list in Command Center / sidebar | Not an in-chat switcher. Do not add `showsSessionSwitcher: true` |
| Settings, Talk tab, onboarding wizard, Watch, Agent Pro, Control UI WebView | Out of chat polish |
| `SessionDashboardScreen` | Gateway Control UI, not transcript chrome |
| Chat actions information architecture | Custom popover is awkward (fixed 560pt height) but that is a later chrome PR |
| SQLite / config / protocol | Not a visual change; needs separate approval |
| macOS full composer chrome | Shared-kit changes must no-op or stay behind `composerChrome == .clean` / `os(iOS)` |
| Jump-to-latest show/hide rules | Already fixed a flash (`#108693`). Reuse `chatReaderShowsJumpToLatest` |

Optional later, not these slices: move the canvas to a system grouped
background; replace the ellipsis popover with `Menu`; Messages-style
timestamp gutters.

`DESIGN.md` lists `drawerRadius` on `OpenClawProMetric`, but the enum
does not define it. Treat that as docs drift in a separate chore, not
part of chat polish.

---

## 4. Upstream PR slices

Land in this fork first, then open the same scoped diffs against
`openclaw/openclaw`. Keep macOS behavior identical unless a change is a
pure bugfix.

### Slice A — Stabilize Chat surfaces (mergeable alone)

**Title (suggested):** `fix(ios): stop Chat empty/loading flashes`

**Includes:** P1, P2, P5.

**Why this slice first.** Motion on top of flashing states just animates
the jank. Upstream reviewers can accept a state-machine fix without buying
a motion language.

**Diff bound.**

- `apps/ios/Sources/Design/ChatProTab.swift`
- `apps/shared/OpenClawKit/Sources/OpenClawChatUI/ChatView.swift`
- Presentation helper (same files or a small new `ChatSurfacePresentation.swift`
  next to `ChatView.swift`)
- Tests: `GatewayStatusBuilderTests.swift`, new
  `ChatSurfacePresentationTests.swift` in OpenClawKit

**Proof.** Before/after screen recordings: cold launch, session switch,
disconnect, empty new session, error with and without history. Reduce
Motion on. Light and dark.

**Non-goals.** No new bubble animation, no composer restyle, no session
list work.

### Slice B — Subtle iOS Chat motion (depends on A)

**Title (suggested):** `improve(ios): native Chat bubble and composer motion`

**Includes:** P3, P4, P6. P7 only if Slice A is landed and grouping stays
a few-line spacing change.

**Diff bound.**

- `ChatView.swift`, `ChatComposer.swift`, `CleanChatComposerControls.swift`
- Small helpers in `ChatMessageViews.swift` if needed
- `apps/ios/DESIGN.md`: add a short **Motion** subsection (durations,
  Reduce Motion, no glass on content) so the next reviewer has a rule

**Proof.** Device or simulator recordings: send, stream start, attachment
add, keyboard show, jump-to-latest, banner. iOS 18 and iOS 26. Dynamic
Type XXXL. Reduce Motion.

**Non-goals.** No Messages tails, no system-blue bubbles, no claw removal,
no Command Center restyle.

### Why not one PR

Slice A is correctness. Slice B is taste. Upstream is more likely to merge
a flash fix than a motion pass bundled with it. Slice B also needs visual
evidence that Slice A removed the states being animated.

---

## 5. Implementation notes for whoever codes this

- Read `apps/ios/DESIGN.md` and `apps/ios/AGENTS.md` before editing.
- Keep `#available(iOS 26.0, *)` fallbacks with the same label, action,
  tint, accessibility, and hit target.
- Prefer `#if os(iOS)` / `composerChrome == .clean` over behavior that
  surprises macOS.
- New user-visible strings go through `String(localized:)`.
- Typography tests: if a host `Text` / `Button` is touched, update
  `apps/ios/Tests/OpenClawTypographyTests.swift`.
- Do not add screenshot-only tests that mirror implementation. Prefer
  presentation-enum tests that fail on the old flash (P1) and scroll
  identity tests that fail on remount (P2).
- Visual proof is required (`DESIGN.md` review checklist): matched
  before/after, sanitized, light and dark.

### Suggested first implementation order inside Slice A

1. Write `ChatSurfacePresentation` tests against the current boolean soup
   (they should describe today's flashes).
2. Implement the enum and switch overlays to it with no animation.
3. Remove or narrow `Preparing Chat`.
4. Revisit `.id(ObjectIdentifier(viewModel))`.
5. Reserve toolbar width for connecting.

Only then open Slice B.

---

## 6. Success for the overall effort

Chat still reads as OpenClaw: red send, branded type, claw while thinking,
grey assistant bubbles, glass composer.

It feels iOS-native in the ways people actually notice: one stable thread
surface, no preparing/loading/empty shuffle, composer and keyboard that
grow in place, a short bubble insert, a quiet connecting dot.

Nothing in Settings, Talk, onboarding, or Command Center had to change to
get there.
