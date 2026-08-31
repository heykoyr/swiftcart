# SwiftCart — Design System

How four apps, built by one person, stay recognisably the same product.

---

## The rule it enforces

`constants/theme.ts` is shipped into all four apps and opens with the constraint it exists to impose:

```
// SwiftCart Design System — shared across all SwiftCart apps.
// NEVER use hardcoded colours, sizes, or shadows anywhere.
```

Everything below follows from taking that seriously.

---

## 1. Tokens

**Type** — five Satoshi weights (`Light`, `Regular`, `Medium`, `Bold`, `Black`) over an 11-step size scale from `xs: 12` to `6xl: 40`.

**Spacing** — a 4px-based scale keyed by multiplier (`1: 4` … `16: 64`), so `spacing[4]` reads as "four units," not "sixteen pixels."

**Layout constants** — the measurements that must never drift between screens or apps:

```
screenPaddingH: 16   cardPadding: 16   sectionGap: 24
inputHeight:    52   buttonHeight: 52  tabBarHeight: 60
```

A 52px input and a 52px button mean a form and its submit button always align.

**Radii** — seven steps, `xs: 4` through `full: 100`.

**Elevation** — five levels (`xs` → `xl`), each pairing an iOS shadow with an Android `elevation` so depth reads the same on both platforms. Opacity climbs gently (`0.03` → `0.12`) rather than jumping.

**Colour — ~60 tokens named by role, never by hue.**

This is the decision that makes the rest possible. Tokens are `textSecondary`, `dangerLight`, `inputBorderFocused`, `walletSubtext` — not `grey500` or `red100`. Semantic families each carry a `base` / `Light` / `Dark` triple:

| Family | Used for |
|---|---|
| `primary` | brand actions — the one token white-label builds override |
| `success` `warning` `danger` `info` | state and feedback |
| `bonus` | rewards, loyalty, referral surfaces |

Because names describe purpose, a company's brand colour swaps `primary` and nothing else has to move.

---

## 2. Type scale

One `<Text>` component owns typography. It exposes **18 named variants** and no free-floating font sizes:

| | |
|---|---|
| `h1` `h2` `h3` `h4` `h5` | 28 / 24 / 20 / 17 / 16 — Bold |
| `bodyLg` `body` `bodySm` | 16 / 15 / 14 — Regular |
| `label` `labelBold` | 14 — form labels |
| `caption` `micro` `overline` | 13 / 12 — helper text, timestamps, section labels |
| `display` `displayBold` | 34 Black / 28 Bold — hero numbers and stats |
| `button` `buttonLg` | 15 / 16 Bold |

Every variant sets `fontFamily`, `fontSize` and `lineHeight` together, so vertical rhythm can't be half-applied.

**Migration is designed in.** Several variants exist purely as backward-compatible aliases — `bodyMd` resolves to `bodySm`, `h5` sits in the `h4` band — because the customer app's older scale had to keep rendering while the four apps converged on one system. A design system that can't absorb its own history gets abandoned; this one carries the aliases and moves on.

**One typographic detail worth recording.** Pill and badge text deliberately leaves `lineHeight` **unset**, inheriting the loose default. Satoshi's metrics need a generous line box inside a small capsule — pinning a tight explicit `lineHeight` clips descenders on Android. The fix is to specify less, not more.

---

## 3. Motion as tokens

`utils/motion.ts` treats interaction feel as a system decision rather than a per-screen improvisation:

```
durationFast: 150   durationMedium: 250   durationSlow: 350
scalePress:   0.96
springPress: { damping: 18, stiffness: 380 }
springPop:   { damping: 14, stiffness: 280 }
```

Two hooks consume them — `useFadeInUp` (entrance: 10px rise + fade, cubic ease-out) and `useScalePress` (press: timed compression to `0.96`, spring release). Presses compress on a timing curve and release on a spring, because release should feel physical and press should feel immediate. Everything runs on the native driver.

---

## 4. A shared component vocabulary

The same parts appear in all four apps, so a rider screen and a vendor screen are visibly the same product:

**Structure** — `ScreenContainer`, `ScreenShell`, `ScreenHeader`, `SectionHeader`, `PinnedButton` (the action that stays docked above the keyboard and safe area).

**Feedback** — `SkeletonLoader`, `EmptyState`, `NetworkErrorCard`, `StatusBadge`, `ConfirmSheet`, `BottomSheet`.

**Input** — `OTPInput`, `PhoneInput`, `CustomDropdown`, `ImageUploadField`, `SegmentedControl`.

**Data** — `StatCard`, `KPICard`, `DataRow`, `TransactionRow`, `RatingStars`, `SignedImage`.

Each app then adds only what is genuinely its own: `SwipeableCartItem` and `ProductPreviewRail` (customer), `StepBar` (vendor), `FaceVerificationRing` and `DeliveryCard` (rider), `CompanySwitcher`, `CashFlowChart` and `MfaGate` (admin).

---

## 5. States are part of the design, not an afterthought

Loading, empty, error and refresh handling appears **662 times across 163 files**. Concretely:

- **Loading** — `SkeletonLoader` pulses between `0.4` and `1.0` opacity on an 800ms loop, sized to the content it replaces, so lists don't reflow when data lands.
- **Offline** — a dedicated `NetworkErrorCard` in every app, distinguishing "no connection" from "nothing here."
- **Empty** — a stated reason and a way forward, not a blank screen.
- **Refresh** — pull-to-refresh throughout.

**`StatusBadge` is the clearest example of state design.** All 13 order states are mapped to a semantic token pair — light background, dark text from the same family — and, more importantly, to **human labels**: `navigating_to_store` reads as *"Heading to Store"*, `navigating_to_customer` as *"On the Way"*. Colour carries the lifecycle (warning → info → bonus → primary → success), so a customer can read progress at a glance without reading a word. Unknown states fall back to a neutral chip with the raw status de-underscored, so a new backend state degrades to something legible instead of crashing.

---

## 6. White-labelling, and the contrast problem it creates

The rider and admin apps present as the delivery company the user works for; the customer app is always SwiftCart. That means arbitrary brand colours entering a system designed around one.

**Two problems, both solved properly.**

**Readability.** A company supplying a near-black brand colour is fine with white text. One supplying yellow or mint is not. `utils/contrast.ts` implements WCAG 2.1 relative luminance and the `(L1+0.05)/(L2+0.05)` ratio from scratch, then picks the foreground by **measured contrast** rather than assumption:

```ts
export function onColor(background: Rgb): { hex: string; isLight: boolean; ratio: number }
```

`surfaceFromBrand()` builds a whole readable surface from one brand colour — background, primary text, muted text, hairline, chip fill — and returns the achieved ratio. Muted text sits at `0.72` alpha, moved up from a hardcoded `0.50` that measured **below AA on mid-tone brand colours**. An unparseable or absent brand colour falls back to the default surface, so an unbranded company looks exactly as before.

**No flash.** Build-time brand colours are applied **synchronously at module load**, before any lazily-loaded screen calls `StyleSheet.create()` — because React Native captures colour values at stylesheet creation. Applying the brand after first render would paint SwiftCart red for a frame and then correct itself. Runtime values from the database cover anything that re-renders. The generic build sets neither and is pixel-identical to an unbranded app.

---

## 7. Where the system is honest about its gaps

- **Screen-reader support is not implemented.** `accessibilityLabel` and `accessibilityRole` appear in only a handful of places across 234 screens. The contrast work above is real and the marketing site meets WCAG 2.1 AA, but the apps are not instrumented for assistive technology. This is the clearest outstanding design debt.
- **Light mode only.** No dark theme; the token structure would support one, but it hasn't been built.
- **No visual regression testing.** Consistency is maintained by the token discipline and code review, not by tooling.
