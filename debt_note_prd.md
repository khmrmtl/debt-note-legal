# Utang Tracker — Product Reference Document
> This file is the single source of truth for the Utang Tracker app. Reference this before making any architectural, UI, or feature decision. Do not deviate from the rules defined here without explicit instruction.

---

## 1. Product Overview

**App Name:** Debt Note (on device), "Debt Note: Debt Tracker & IOU App" (App Store / Play Store listing)
**Platform:** iOS and Android via Flutter
**Monetization:** One-time purchase (target price varies by region; PH: ₱199, US: $1.99–$2.99)
**Target Market:** International users who lend money informally to friends, family, and acquaintances. Filipino-first branding with global reach.

**Core Problem Solved:**
Tracking informal debt between people you know is socially awkward. The real friction is not recording the amount — it is collecting it. This app removes that social friction by automating the reminder process in a way that feels official and app-generated, not personal.

---

## 2. Absolute Rules (Never Violate)

- **Offline first, always.** No backend, no server, no cloud sync, no user accounts. All data lives on-device using local storage (Hive or Isar).
- **No subscriptions, no recurring payments.** One-time purchase only. This is a core marketing promise.
- **No internet required** for any core feature. Internet is only used if the user's own messaging app requires it (e.g. Messenger) — the app itself never calls any API.
- **Default language is English.** Filipino (Taglish) is a supported localization. Language is selectable in Settings. No language selection on first launch — English is the default.
  - **English tone — professional and clean.** Plain, clear, trustworthy. Reads like a polished fintech product. This is the default user-facing voice and the one the App Store / Play Store listing is written in.
  - **Filipino tone — informal, barkada-style Taglish.** Casual chat register ("boii", "pre", playful jokes, light self-deprecation). This locale is the "friend helping you collect" voice, not a formal translation of the English strings. The two tones are intentionally different, not parallel.
- **Privacy by design.** No analytics, no crash reporting that sends personal data, no third-party SDKs that phone home. User data never leaves the device.
- **No feature creep.** This is not a full budget app. It does not track expenses, income, or savings. It tracks who owes whom, how much, and helps collect it. That is the entire scope.

---

## 3. Core Features (v1 Scope)

### 3.1 People-Based Debt Tracking
- Each debt is attached to a **person (contact)**, not just an amount
- Adding a person supports two methods: **manual name entry** or **imported from phone contacts** (via `flutter_contacts`, read-only). Both methods result in the same local Person entity — no live sync with the phone's contact book after import.
- Each person can have both pautang and utang simultaneously (e.g. you lent them ₱500 last month and you owe them ₱200 this month). These are tracked as separate debt entries under the same person, not combined.
- Each person has a profile card showing:
  - Name + optional photo or initials avatar
  - Net balance summary: total they owe you vs total you owe them, shown separately
  - Number of active debt entries
  - Last activity date
  - Days overdue on oldest unpaid entry (if applicable)
  - Date of last reminder sent
- The app is **primarily pautang-focused**. Home screen and dashboard prioritize receivables (what others owe you). User's own utang (what they owe) is fully supported but presented as secondary.

### 3.2 Transaction Logging
- Log a new debt entry with:
  - Amount (peso)
  - Person
  - Date incurred
  - Optional note/label (e.g. "para sa grocery", "emergency")
  - Optional proof/credentials attachments — one or more photos (GCash screenshot, signed papel, borrower ID, etc.)
- Log **partial payments** against any debt
  - Show payment timeline per person
  - Show remaining balance
  - Show visual progress bar (% paid)
  - **Payments can be undone** — user can delete a logged payment if entered by mistake. Deleting a payment restores the remaining balance accordingly.
- **Auto-settle on full payment** — when the sum of all logged payments equals the total owed (original amount + accrued interest, if applicable), the app automatically marks the debt as settled and triggers archive flow with a confirmation animation. No separate manual "mark as settled" button needed — settlement is a mathematical result, not a manual action. This is reliable offline since it is pure local arithmetic.
- Fully settled debts move to **Archive** with a satisfying animation (confetti or checkmark)

### 3.3 Optional Interest Calculation
- When adding a debt, the user can **optionally enable interest** with:
  - **Interest rate** (e.g. 5%)
  - **Period**: Daily, Weekly, Biweekly, Monthly, Quarterly, Semi-annual, or Annual
  - **Interest type**: Simple or Compound
  - **Calculation basis**: Original Principal or Remaining Balance
- **Simple**: Interest is linear — the rate applies to the base each period without growing on itself. Formula: `base × rate × periods`.
- **Compound**: Interest accrues on previously-accrued interest, growing faster over time. Formula: `base × ((1 + rate)^periods − 1)`.
- **Original Principal basis**: Interest accrues on the original debt amount regardless of payments made. Common in Filipino informal lending ("5/6" system).
- **Remaining Balance basis**: Interest accrues on the unpaid balance. Decreases as payments are made. For simple interest, calculated by segmenting time periods at each payment date. For compound interest, walked period-by-period with payments applied before each compounding step.
- The four combinations (Simple × Original, Simple × Remaining, Compound × Original, Compound × Remaining) cover the range of informal and formal lending arrangements.
- Default when enabled: Simple × Original Principal (Filipino "5/6" style).
- Interest is **computed at render time** (pure local arithmetic) — not stored as a separate amount. Always current.
- Interest is reflected in:
  - **Debt entry cards**: Shows "+ ₱X interest (Y% monthly)" below the amount. Progress bar denominator includes interest.
  - **Person totals**: Total owed/owing includes accrued interest.
  - **Home dashboard**: Receivable/payable totals include interest.
  - **Receipt reminder**: Itemized breakdown below the amount (Principal, Interest with period × count, Total Due). Only shown when interest > 0.
  - **Auto-settle threshold**: Settlement triggers when `totalPaid >= principal + accruedInterest`.
  - **Interest-accrual notifications**: Fire only when a new interest period ticks over (matching the debt's own period — monthly interest notifies monthly, weekly weekly, etc.).
- Interest applies to **both pautang and utang** debts equally.
- Debts without interest behave exactly as before — no interest section on cards or receipts.
- Each type and basis option shows a short localized explanation in the Add Debt form.

### 3.4 Soft Reminder System (Key Differentiator)
- User taps "Send Reminder" on any person's debt
- App generates a **receipt image** on-device (no internet needed for generation)
- Receipt design mimics GCash/Maya receipt format:
  - Green header with bell icon and "Payment Reminder" title
  - Large peso amount
  - Creditor name, date incurred, days overdue
  - Auto-generated reference number (format: `UTG-YYYY-XXXXX`)
  - **Reminder cooldown** — a reminder can only be sent once every **24 hours per debt entry**. The send button is disabled and shows "Reminder sent X hours ago" until the cooldown expires. This prevents spam and protects the sender's relationship with the receiver.
- **Reminder log** — every sent reminder is recorded locally (date + time + debt ID). The Person Detail screen shows "Last reminded: X days ago" per debt entry.
- Receipt disclaimer: *"Ito ay isang automated reminder. Para sa katanungan, makipag-ugnayan sa nagpadala nito. Salamat po!"*
  - Dashed footer: "Sent via [App Name]" + timestamp
- Receipt is rendered as a PNG using `RepaintBoundary` + `toImage()` — fully offline
- User shares via **share_plus** (native share sheet)
  - Works with Messenger, Instagram DMs, Viber, Telegram, SMS — any app the user has
  - User chooses the platform; app does not force a specific one
- The message feels **app-generated and official**, not personal, to remove social awkwardness for both sender and receiver
- The "Sent via [App Name]" footer on every receipt is passive organic marketing

### 3.5 Home Screen Widget
- Show top 3 largest outstanding receivables at a glance
- No need to open app
- Implemented via `home_widget` Flutter package

### 3.6 Data Backup / Export
- Manual local backup to device storage as **CSV format**
- Manual restore from CSV backup file — supported as a first-class flow (uninstall + reinstall safe)
- Backup should include all People, Debts, Payments, and ReminderLog entries in separate CSV sheets or files

---

## 4. Features Explicitly Out of Scope (v1)

- No expense tracking
- No income tracking
- No savings goals
- No cloud sync
- No user login or accounts
- No push notifications via server (local notifications only via `flutter_local_notifications`)
- ~~No interest calculation~~ — **Added in v1** (see Section 3.3)
- No group debt splitting
- No GCash/Maya API integration

---

## 5. UI/UX Principles

- **Warm, approachable design.** Not a bank app. Friendly colors, not cold corporate blue and white.
- **Speed of entry is critical.** Adding a new debt must take no more than 3 taps + amount input.
- **Micro-animations matter.** Satisfying checkmark/confetti on full settlement. Smooth transitions on payment progress bar.
- **Fully settled debts go to archive** — keeping the main list clean and action-focused.
- **Dark mode support** — required from day one.
- **English language default** — all UI labels, buttons, navigation, empty states, and error messages are in clean, professional English by default. Filipino (Taglish) is available as a localization in Settings.
  - When Filipino is selected, the tone flips to informal barkada-style Taglish across **all** UI surfaces — screen titles, buttons, empty states, confirmations, notifications, receipt header copy. The casual voice is the whole point of the locale.
  - Receipt disclaimer and "Sent via Debt Note" footer are always in English regardless of locale.
- Never mix languages mid-label. A button says either "Send Reminder" or "Magpadala ng Reminder" — not "Send ng Reminder."
- **Multi-currency support** — currency is auto-detected from device locale on first launch. Displayed using locale-aware formatting (symbol, decimals, separators). Editable in Settings.

### Color Direction
- Primary accent: Teal/green family (trust, money, financial)
- Avoid: Cold corporate blue, aggressive red for primary actions
- Overdue indicators: Warm amber (mild) → coral/red (severe) — escalating urgency

### Typography
- Clean, readable sans-serif
- Amount figures should be large and prominent — the number is the most important element on any screen

---

## 6. Tech Stack

| Concern | Solution |
|---|---|
| Framework | Flutter (latest stable) |
| Local database | Hive or Isar |
| State management | flutter_bloc (BLoC pattern) |
| Receipt image generation | `RepaintBoundary` + `toImage()` |
| Sharing | `share_plus` |
| Home screen widget | `home_widget` |
| Local notifications | `flutter_local_notifications` |
| Contact name/photo lookup | `flutter_contacts` (read-only, optional) |
| Font | `plus_jakarta_sans` or `dm_sans` via Google Fonts |
| Localization | `flutter_localizations` + ARB files |
| Number/currency formatting | `intl` package |
| Glass effect (iOS only) | Manual implementation — custom fragment shader + `BackdropFilter` fallback. No external package. |

### Minimum Targets
| Platform | Minimum Version | Rationale |
|---|---|---|
| iOS | 14.0 | Covers 98%+ of active iPhones globally. Required for `home_widget` support. |
| Android | SDK 21 (Android 5.0) | Maximum market coverage for budget Android devices globally. |
| Flutter | Latest stable | No pinned version. Always `flutter upgrade` before starting a session. |

---

## 7. App Screens (Minimum)

1. **Home / Dashboard** — summary cards (total receivable, total payable), recent activity, quick-add button
2. **People List** — all contacts with outstanding debt, sortable by amount / overdue days
3. **Person Detail** — full debt history for one person, partial payments, send reminder button
4. **Add Debt** — form: amount, person, date, note, photo
5. **Log Payment** — partial or full payment against a specific debt
6. **Receipt Preview** — shows generated receipt before sharing, with share button
7. **Archive** — fully settled debts
8. **Settings** — your name, appearance/theme, language selector (English / Filipino), currency selector (auto-detected, editable), notification toggle + time picker, backup/restore, app info

---

## 8. Monetization & Marketing Notes

- **Price point:** Region-dependent. PH: ₱199, US/global: $1.99–$2.99 one-time
- **Platform:** iOS App Store and Google Play simultaneous
- **Main marketing channel:** Short-form video content (TikTok, Instagram Reels, Facebook Reels)
- **Content angle:** Show the awkwardness the app solves. Universal scenarios — friends who don't pay back, family IOUs, split bills gone wrong. Filipino-specific content for PH market.
- **Organic marketing loop:** Every receipt shared contains "Sent via [App Name]" — receivers become aware of the app passively
- **No paid ads planned for v1**

---

## 9. Competitor Landscape

| App | Problem |
|---|---|
| Generic debt tracker apps | Western-made, no Filipino context, ugly UI |
| "Utang" app on Play Store | Barebones v1, no receipt feature, weak UI |
| Tarsi | Budget app, not debt-specific — different use case |
| Peddlr | Free but targets sari-sari store owners, fintech model |

**Gap we fill:** A beautiful, Filipino-first, offline debt tracker with a socially intelligent reminder system. Nothing in the current market does this.

---

## 10. Success Metrics (Personal Targets)

- v1 release within [developer sets own timeline]
- Target: modest consistent downloads — not viral, just sustainable
- Every receipt shared = free impression
- App should feel complete and polished at launch — no "beta" quality

---

---

## 11. Visual Design System

### App Name Rationale
**Debt Note** — plain, descriptive, and globally legible. Pairs naturally with the receipt-style reminder (the app "sends a note about the debt"). "Sent via Debt Note" on receipts reads as a legitimate fintech product.

### Color Palette
| Role | Hex | Usage |
|---|---|---|
| Primary | `#1D9E75` | Main CTAs, active states, receipt header |
| Primary dark | `#0F6E56` | Pressed states, dark mode primary |
| Primary light | `#E1F5EE` | Background tints, chips, badges |
| Overdue amber | `#EF9F27` | Mild overdue warning (7–30 days) |
| Overdue red | `#E24B4A` | Severe overdue (30+ days) |
| Settled green | `#639922` | Fully paid indicator |
| Neutral surface | System | Let Flutter `ColorScheme.fromSeed` handle light/dark backgrounds natively |

### Typography
- **Font:** `Plus Jakarta Sans` or `DM Sans` via Google Fonts
- Avoid Roboto (too Android-default) and decorative fonts
- Amount figures: large and prominent — the number is always the hero element on any screen

### Shape & Spacing
- Card border radius: `12px`
- Button and input border radius: `8px`
- Generous padding throughout — financial data needs breathing room
- Flat cards with subtle `0.5px` borders — no heavy shadows

### Design Personality
Three words: **clean, warm, trustworthy.** Not playful enough to look like a game, not cold enough to look like a bank. The Taglish copy carries the personality — the visuals stay polished and neutral.

### Receipt as a Moment
- The receipt is the **only element** in the app that uses a full-color teal header
- All other screens use neutral surfaces — no colored headers, no strong background fills
- This contrast makes receipt generation feel like a distinct, premium moment
- The receipt's visual weight signals to the user: *this is official, this means business*

### iOS Liquid Glass Theming (iOS 26 Design Language)
- On iOS, apply **Liquid Glass** theming using a manually implemented custom widget
- No external package dependency — implementation uses Flutter's own `BackdropFilter`, `ImageFilter`, and custom fragment shaders via `FragmentProgram`
- Liquid Glass is Apple's iOS 26 design material — shader-based dynamic blur that refracts and tints content behind it
- Apply Liquid Glass to: navigation bar, bottom tab bar, modal sheets, card overlays
- Tint color: `#E1F5EE` (Primary light) at `0.15–0.20` opacity over the glass surface
- If custom shader implementation is too complex or fails on device: fall back gracefully to `BackdropFilter` + `ImageFilter.blur(sigmaX: 10, sigmaY: 10)` with same tint — this is an acceptable production fallback and still looks premium
- On Android: flat Material surfaces only — no glass, no blur, no shaders
- Detect platform via `Platform.isIOS`
- Use `kIsWeb` guard to prevent any glass code running on web builds

### Liquid Glass Implementation Rules
- Wrap all glass elements in `ClipRRect` with matching border radius — blur bleeds outside bounds without clipping
- Use `RepaintBoundary` around expensive glass surfaces to isolate repaints
- Wrap shader code in `try/catch` — if shader compilation fails silently fall back to `BackdropFilter`
- **Never apply any glass or blur effect to the receipt widget** — the receipt must render as a clean flat PNG for sharing. No exceptions.
- Glass border: `1px` white at `0.25` opacity on iOS
- Test on physical iOS device — simulator shader rendering is unreliable
- Do not add any pub.dev package for this effect — own the code

---

## 12. Localization & Internationalization

### 12.1 Supported Languages (v1)
| Language | Code | Role |
|---|---|---|
| English | `en` | Default. All UI text ships in English. |
| Filipino (Taglish) | `fil` | Localization. Informal, barkada-style Taglish — casual chat register with jokes, "boii/pre" address, and playful self-deprecation. Not a literal translation of the English strings; it is a distinct voice. |

### 12.2 Localization Method
- **Flutter's built-in localization** via `flutter_localizations` + ARB files
- **`intl` package** for locale-aware number and currency formatting
- All user-facing strings extracted to ARB files (`app_en.arb`, `app_fil.arb`)
- Language setting stored in Hive preferences box as `locale` (e.g. `en`, `fil`)
- Editable in Settings screen at any time
- No language selection on first launch — English is the default

### 12.3 Currency System
- **Auto-detect** currency from device locale on first launch via `Platform.localeName` → country code → currency mapping (e.g. `en_PH` → PHP, `en_US` → USD, `ja_JP` → JPY)
- Store selected currency code in Hive preferences box as `currencyCode` (e.g. `PHP`, `USD`, `EUR`, `JPY`)
- Currency shown on first-launch welcome sheet as a **pre-filled dropdown** — user can change it before continuing
- Editable in Settings screen at any time
- Use `intl` package `NumberFormat.currency()` for locale-aware formatting — handles symbol, decimal places, thousand separators automatically
- Replace all `formatPeso()` calls with a locale-aware `formatCurrency()` function
- The app does not convert between currencies — it simply displays amounts in the user's chosen currency symbol and format

### 12.4 First-Launch Welcome Sheet
- Current: name field only
- Updated: name field + currency dropdown (pre-filled from device locale)
- No language dropdown on first launch (defaults to English, changeable in Settings)

### 12.5 Receipt Localization
- Receipt body labels (From, To, Date Incurred, Days Overdue, Reference No., Note): **user's selected language**
- Receipt disclaimer: **always English** — "This is an automated reminder. For questions, please contact the sender. Thank you!"
- "Sent via Debt Note" footer: **always English**
- Currency on receipt: user's selected currency and formatting

### 12.6 Notification Localization
- Notification title and body: user's selected language
- English: "You have [N] overdue debt(s). Time to collect!"
- Filipino: "May [N] overdue debt(s) ka. Magtanggal na ng utang today!"

### 12.7 Strings to Extract (~110 total)
- ~45 Taglish strings → Filipino ARB file (`app_fil.arb`)
- ~65 English strings → English ARB file (`app_en.arb`) — the default
- `formatPeso()` → `formatCurrency()` across 20+ call sites
- Receipt widget: 10+ labels
- All screen titles, button labels, empty states, error messages, snackbar messages, dialog copy, notification copy

### 12.8 Settings Screen Additions
- **Language** — selector with two options: English, Filipino. Stored as `locale` in preferences.
- **Currency** — searchable dropdown with common currencies (PHP, USD, EUR, GBP, JPY, KRW, AUD, CAD, SGD, etc.). Auto-detected on first launch, stored as `currencyCode` in preferences.
