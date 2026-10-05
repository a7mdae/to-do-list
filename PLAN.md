# Implementation Plan — مختبر الثغرات السيبرانية (Cybersecurity Vulnerability Lab)

Interactive, Arabic-first (RTL) web app for a BTEC IT Level 3 presentation in front of
security engineers and assessors. Three real, publicly disclosed incidents, each with a
safe in-browser simulation in a **vulnerable** mode and a **secure** mode.

Source idea: [`docs/ORIGINAL_IDEA_AR.txt`](docs/ORIGINAL_IDEA_AR.txt). This plan keeps that
scope, fixes some facts (§1), and turns it into an architecture (§2–§6) and build order (§7).

---

## 1. Fact-check of the three case studies (read this first)

The audience is security engineers, so the facts have to hold up. I checked each
incident against the original disclosure. **Three details in the original idea need correcting.**

| | Instagram | Shopify | Snapchat |
|---|---|---|---|
| Layer | المصادقة — Authentication | التفويض — Authorization | أمان الخادم — Server-side |
| Bug class | Account takeover via OTP brute force | IDOR | SSRF |
| Researcher | Laxman Muthiyah | (HackerOne report, disclosed) | Ben Sadeghipour (@nahamsec), Cody Brocious (@daeken), @ziot |
| Reported | 2019 (public July 2019) | 12 Oct 2023 (disclosed 8 Feb 2024) | 8 Apr 2019 (disclosed 30 Nov 2020) |
| Bounty | $30,000 (Facebook) | $5,000 · severity Medium | $4,000 |
| Primary source | thezerohack.com/hack-any-instagram | hackerone.com/reports/2207248 | hackerone.com/reports/530974 |

### Corrections to the original idea

1. **Instagram did have rate limiting. It was bypassed, not missing.** The limit was
   counted **per IP** (about 200 requests per IP). The researcher used a **race condition**
   (concurrent requests) plus **IP rotation** (1,000 machines, 200k requests = 20% of the
   1,000,000 possible 6‑digit codes, within the code's 10‑minute validity). His estimate
   for a full attack was 5,000 IPs and about $150.
   → In the vulnerable mode, the limiter is **per IP**. The attacker gets around it by rotating IPs.
   The secure mode counts attempts **per account / per issued code** and invalidates the code
   after N failures. Engineers will ask about this difference, and the lab should demo it.

2. **Snapchat's SSRF went through a headless browser and DNS rebinding.** The Ads Manager
   "Import" feature (`/api/v1/media/import`) loaded a user-supplied URL in headless Chrome,
   which **ran JavaScript**. The researchers' domain first pointed to their own server.
   They then rebound it to `169.254.169.254` (Google Cloud metadata) and minted
   service-account tokens.
   → A plain domain allowlist or a one-time IP check is **not enough** in the secure mode.
   It must also resolve DNS once, pin the IP, check that pinned IP, and refuse redirects.
   Showing a DNS-rebinding bypass against a naive filter is the strongest demo in the project.

3. **There are no CVEs for these three.** CVEs cover distributable software. Bugs in a
   company's own hosted service that get fixed server-side normally get none.
   → Use as sources: HackerOne reports, the researcher's write-up, and reputable coverage
   (PortSwigger Daily Swig, SecurityWeek, WeLiveSecurity). Do not show "CVE" in the UI.

**Shopify scenario mapping:** report #2207248 is an IDOR on `BillingInvoice` IDs in the
`BillingDocumentDownload` / `BillDetails` GraphQL operations. It leaked other merchants'
invoices, email, address, and card type + last 4 digits. The original idea's "report-A.pdf /
report-B.pdf" maps directly onto **invoice download**, so the lab uses invoices
(`invoice-1001.pdf`, `invoice-1002.pdf`) and can then be called a faithful simplification.

**Recommended presentation order:** Instagram → Shopify → Snapchat.
This follows the story "من أنت؟ ← ماذا يحق لك؟ ← إلى أين يصل الخادم؟", and Snapchat, the
hardest one (متقدم), lands last. Difficulty badges: Shopify سهل, Instagram متوسط, Snapchat متقدم.

---

## 2. Tech stack (decisions)

| Concern | Choice | Why |
|---|---|---|
| Build | **Vite + React 19 + TypeScript** | Fast, static output. TS catches data and content typos in the case files. |
| Styling | **Tailwind CSS v4** (`@tailwindcss/vite`) | Design tokens in CSS `@theme`. Logical utilities (`ms-*`, `pe-*`, `start-*`) make RTL automatic. |
| Routing | **React Router, `HashRouter`** | Works on any static host and in an offline build without server rewrites. |
| Fonts | **`@fontsource/cairo`** (UI) + **`@fontsource/jetbrains-mono`** (code), self-hosted | The venue may have no internet. Nothing loads from Google Fonts at runtime. |
| Icons | `lucide-react` | Per the idea. Tree-shaken. |
| Motion | CSS transitions + `motion` only where needed | Respect `prefers-reduced-motion`. |
| Tests | **Vitest** (lab engines) + **Playwright** (smoke + "no network" test) | §6. |
| Lint | ESLint + `no-restricted-globals` for `fetch` / `XMLHttpRequest` / `WebSocket` in `src/labs/**` | Enforces "100% simulated". |

No backend and no state library. React state plus a small context for mode, glossary, and presentation is enough.

---

## 3. Architecture

### 3.1 Core idea: each lab = a pure "mock server" + a UI

Every lab has an `engine.ts` with **pure functions** that act as the target server:

```ts
type Mode = 'vulnerable' | 'secure';
interface SimRequest  { method: string; path: string; headers?: Record<string,string>; body?: unknown; actor: Actor }
interface SimResponse { status: number; body: unknown; log: LogLine[] }   // log = what the "server" did, step by step
function handle(req: SimRequest, mode: Mode, state: LabState): { res: SimResponse; state: LabState }
```

- The UI never decides whether an attack works. The engine does. That keeps the demo
  honest and lets each engine be unit-tested.
- `log` lines drive the "Server Log" panel and the step-by-step flow diagram. Example:
  `✓ المستخدم مسجّل الدخول (AuthN)` then `✗ لم يتم التحقق من الملكية (AuthZ skipped)`.
- The **same request** can be replayed against both modes and shown side by side.
  This is section 7, "إعادة التجربة بعد الإصلاح".

### 3.2 Safety guarantees (state these to the panel)

1. No network calls in lab code. ESLint blocks it at build time.
2. A CSP meta tag, `connect-src 'none'`, means the browser itself refuses any outbound
   request. You can say: "even if I wanted to, this page cannot reach Instagram."
3. All "secrets" are obviously fake: `DEMO-ONLY-TOKEN-…`, `example.test` domains, made-up
   people. Nothing looks like a real credential format.
4. A permanent safety banner (already in the idea), plus a dismissible-per-session detail line.

### 3.3 Folder structure

```
src/
  main.tsx, App.tsx, routes.tsx
  styles/index.css                 # @theme tokens, fonts, RTL base, projector mode
  components/
    layout/   Navbar, SafetyBanner, Footer, PageShell
    glossary/ Term (inline badge), GlossaryDrawer, GlossaryCard
    lab/      ModeToggle, HttpExchange (request/response viewer), ServerLog,
              FlowDiagram, DefenseChecklist, SideBySide
    case/     CaseSectionNav, SourceList, ImpactCards, Callout
  labs/
    instagram/ engine.ts, engine.test.ts, InstagramLab.tsx
    shopify/   engine.ts, engine.test.ts, ShopifyLab.tsx
    snapchat/  engine.ts, mockNetwork.ts, engine.test.ts, SnapchatLab.tsx
  content/                          # ALL Arabic text lives here, not in components
    glossary.ts
    cases/ instagram.ts, shopify.ts, snapchat.ts
    slides.ts
  pages/ Home, CasePage, Compare, GlossaryIndex, Present, PresenterView
  context/ GlossaryContext, PresentationContext
e2e/ smoke.spec.ts, no-network.spec.ts
```

Keeping all copy in `content/` lets the student proofread or translate it without touching
components, and lets TypeScript enforce the 8-section structure.

### 3.4 Routes

| Route | Page |
|---|---|
| `#/` | Hero + safety notice + 3 case cards |
| `#/case/instagram`, `#/case/shopify`, `#/case/snapchat` | Generic `CasePage` with 8 sections |
| `#/compare` | "ماذا تعلمنا من الحالات الثلاث؟" |
| `#/glossary` | Searchable list of all terms |
| `#/present` | Presentation mode (slides) |
| `#/presenter` | Optional presenter view (notes + timer), synced over `BroadcastChannel` |

---

## 4. Content model

### 4.1 Glossary (3-layer rule)

```ts
interface Term {
  id: string;            // 'idor'
  ar: string;            // 'مرجع مباشر غير آمن للكائن'
  en: string;            // 'IDOR — Insecure Direct Object Reference'
  question?: string;     // optional framing: 'ماذا يُسمح لك أن تفعل؟'
  simple: string;        // layer 2: plain explanation
  analogy: string;       // layer 3: everyday analogy
  related?: string[];
}
```

Usage in content: `<Term id="rate-limiting" />` renders `تحديد معدل الطلبات (Rate Limiting)` with a 💡 badge.
Clicking it opens the drawer, which on mobile is a bottom sheet.

**Initial term list (about 24):** Authentication, Authorization, Server-side security,
Account Takeover, OTP / verification code, Brute force, Rate limiting, Race condition,
IP rotation, Account lockout, CAPTCHA, IDOR, Object ID, HTTP 403 vs 404, Least privilege,
SSRF, Cloud metadata service (169.254.169.254), Private IP ranges (RFC 1918), Allowlist vs
denylist, DNS, DNS rebinding, Redirect, Defense in depth, Bug bounty / Responsible disclosure.

### 4.2 Case file (enforces the 8 sections)

```ts
interface CaseStudy {
  slug: 'instagram' | 'shopify' | 'snapchat';
  platform: string; layer: 'authn' | 'authz' | 'server';
  difficulty: 'easy' | 'medium' | 'advanced';
  facts: { researcher: string; reported: string; disclosed?: string; bounty: string; };
  sections: {
    incident: Rich;      // 1 ماذا حدث؟
    definition: Rich;    // 2 ما هي الثغرة؟
    howItWorks: FlowStep[]; // 3 كيف تعمل الفكرة؟  (rendered by FlowDiagram)
    // 4 = vulnerable lab (component slot)
    impact: ImpactItem[];  // 5 ما الخطر؟
    mitigations: Mitigation[]; // 6 كيف نحمي النظام؟
    // 7 = secure lab / side-by-side (component slot)
    sources: Source[];     // 8 المصادر
  };
  speakerNotes: Record<string, string>; // 🎤 ماذا أقول؟ per section
}
```

---

## 5. The three labs (detailed spec)

Shared lab UI: a **mode toggle** (🔴 النسخة المصابة / 🟢 النسخة المحمية), an **HTTP exchange
viewer** (request and response, always `dir="ltr"`), a **server log**, and a **"replay in the
other mode"** button.

### 5.1 Instagram — Account Takeover (Authentication)

**Scenario:** password recovery for `alex_demo`. The server "sends" a 6-digit code,
generated with `crypto.getRandomValues` and shown only in a "victim's phone" mock panel. Valid for 10 simulated minutes.

**Attacker panel controls:** number of IPs (1 → 5,000), concurrency, start/stop.
The attack runs on a **simulated clock** (for example 1 sim-second per 16 ms tick). It does not
loop a million real iterations. Each tick, the engine processes `IPs × requests-per-IP` guesses
against the rules of the current mode.

| | Vulnerable | Secure |
|---|---|---|
| Limit key | per **IP**, about 200 req / IP | per **account + code** |
| Concurrency | race: checks then increments (non-atomic), so bursts slip past | atomic counter |
| Failure policy | none | code invalidated after **3** wrong attempts, then lockout + CAPTCHA + alert to the owner |
| Result | with enough IPs, the code is found inside 10 min → "تم الاستيلاء على الحساب" | stops at attempt 3 whatever the IP count |

**Visuals:** a big counter (attempts / 1,000,000), coverage progress bar, a countdown to code
expiry, a grid of rotating fake IPs (`203.0.113.x` from the documentation ranges), and a probability
readout: vulnerable `≈ attempts/1,000,000`, secure `3/1,000,000 = 0.0003%`.
**DefenseChecklist:** toggle each defense on its own (per-account limit, atomic counter,
invalidate-after-N, expiry) to show which ones actually stop the attack. This works well for the "Defense in Depth" message.

### 5.2 Shopify — IDOR (Authorization)

**Data:** أحمد (ID 1001) owns `invoice-1001.pdf`; محمد (ID 1002) owns `invoice-1002.pdf`.
The fake invoice has a name, a city, `بطاقة •••• 4242`, and an amount. This mirrors what the real report leaked.

**UI:** logged in as أحمد. "My invoices" list, plus an **editable request bar**:
`GET /api/invoices/[1001]/download`. The student changes 1001 to 1002 live.

| Check | Vulnerable | Secure |
|---|---|---|
| Session valid? (AuthN) | ✅ | ✅ |
| `invoice.owner_id === currentUser.id`? (AuthZ) | ⏭️ skipped | ❌ → **403** «تم رفض الوصول لعدم امتلاك الصلاحية» |
| Response | Mohammad's invoice opens (red "LEAKED" stamp) | refused |

**Extra panels:**
- An AuthN vs AuthZ matrix showing the 4 combinations of logged-in/out × owner/not-owner.
- A "Fix in code" snippet: the vulnerable vs secure query (`WHERE id = ? AND owner_id = ?`).
- A myth-buster callout: "Random IDs (UUID) make guessing harder but are **not** a fix. Authorization is the fix." Engineers will appreciate this.
- A talking point on 403 vs 404: returning 404 hides whether the object exists.

### 5.3 Snapchat — SSRF (Server-side Security)

**Scenario:** an "Import image from URL" feature for an ad creative. The simulated server
"fetches" whatever the user enters. Nothing is fetched for real: the engine looks the URL
up in a small **fictional network map** (`mockNetwork.ts`) that holds:
- a public image CDN, `images.example.test`, which returns a placeholder image
- a fictional internal admin service, `mock-internal-service`
- a fictional "cloud metadata" entry at `169.254.169.254`, which returns obviously fake
  data (`DEMO-ONLY-TOKEN`, a made-up project name)

The UI offers three preset buttons, «صورة عادية», «خدمة داخلية», and «بيانات السحابة»,
so the demo stays at the conceptual level of the original idea.

| | Vulnerable | Secure |
|---|---|---|
| Behaviour | Server requests any address it is given | Request passes through a visible **guard pipeline** |
| Internal / metadata URL | Fake sensitive data comes back → «تسريب!» | Blocked with a clear reason in the server log |
| Normal image URL | Works | Works |

**Secure-mode guard pipeline** (each step shown as a ✓ / ✗ card in the FlowDiagram):
1. Only `https` is allowed.
2. The domain must be on the **allowlist** of trusted image hosts.
3. The resolved address must not be private, loopback, link-local, or metadata (RFC 1918, cloud metadata).
4. The checked address is the one actually used. The server does not look it up again, and it does not follow redirects.
5. Network isolation: the fetcher service has no route to internal systems. This is defense in depth, so the server stays safe even if a check is missed.

**Teaching point for the panel:** the real 2019 case shows why step 4 matters. The
researchers got past a check that happened *once* by changing where their domain pointed
*after* the check (DNS rebinding). Present this as **narrative plus a diagram in section 3**,
not as an interactive bypass tool. It shows engineers the student understands the real
incident without turning the page into an attack playbook.

---

## 6. Presentation mode, comparison page, design

### 6.1 Presentation Mode (🎤 وضع العرض)
- `#/present` shows full-screen slides from `content/slides.ts`. Slide types: `title`,
  `concept`, `flow`, `lab` (embeds the live lab in a compact layout), and `compare`.
- Navigation: buttons, plus keyboard **← → / PageUp / PageDown / Space**. Clicker remotes
  send PageUp/PageDown. Map the arrows for RTL so "next" feels natural.
- Larger type (root font-size about 125%), and non-essential chrome hidden.
- **Timer bar**, 6 min target (green → amber at 6:00 → red at 7:00). Suggested split: intro 0:45 ·
  Instagram 1:30 · Shopify 1:15 · Snapchat 1:45 · conclusion 0:45.
- **«🎤 ماذا أقول؟»**: collapsible notes per slide, collapsed by default because the
  audience can see the projector. Optional `#/presenter` window on the laptop screen shows the notes plus
  timer, synced with `BroadcastChannel`. Works offline.

### 6.2 Comparison page (ماذا تعلمنا؟)
- Table: platform → layer → question (من أنت؟ / ماذا يحق لك؟ / إلى أين يصل الخادم؟) →
  root cause → fix.
- **Layered SVG diagram (Defense in Depth):** three nested rings or a stacked wall. Hovering or tapping a
  layer highlights its case.
- Closing line: «الأمان لا يعتمد على جدار واحد، بل على تكامل الطبقات معاً».

### 6.3 Visual design
- Tokens in `@theme`: `--color-bg` (obsidian/slate), `--color-surface` (glass), `--color-accent`
  (cyan), `--color-safe` (emerald), `--color-vuln` (rose), `--color-warn` (amber).
- `<html lang="ar" dir="rtl">`. Every URL, IP, code block, and HTTP exchange is wrapped in
  `dir="ltr"` / `<bdi>` so it doesn't render backwards. This is the most common RTL bug.
- **Projector warning:** dark glassmorphism with thin, low-contrast borders washes out on
  projectors. Add a "Projector contrast" toggle (stronger borders, no blur, brighter text) and
  test on a real projector or a TV before the day.
- Accessibility: visible focus rings, AA contrast, `prefers-reduced-motion`, and every
  colour signal (red/green) paired with an icon plus text.

---

## 7. Build order (phases + definition of done)

The main change from the original phase list: **build one case end to end before the
others** (vertical slice). That way the CasePage template, lab components, and glossary are
proven on the simplest case first.

| Phase | Work | Done when |
|---|---|---|
| **0. Scaffold** | Vite + React + TS, Tailwind v4, fonts, RTL base, tokens, HashRouter, ESLint rule, CSP meta, Vitest + Playwright setup | `npm run dev` shows an RTL page in Cairo. Lint, test, and build all pass. |
| **1. Shell** | Navbar, SafetyBanner, Home (hero + 3 cards), Footer | Home matches design on desktop and phone width |
| **2. Glossary** | `glossary.ts` (all terms, 3 layers), `<Term>`, drawer, `#/glossary` | Clicking any term anywhere opens its card. Search works. |
| **3. Vertical slice: Shopify** | `CasePage` template + 8 sections, shared lab components, Shopify engine + tests + UI | Full Shopify page works in both modes. Engine tests are green. |
| **4. Instagram** | Engine (simulated clock, per-IP vs per-account), attacker panel, DefenseChecklist | Vulnerable mode reaches takeover. Secure mode stops at 3 in every configuration. |
| **5. Snapchat** | Mock network, guard pipeline, flow diagram incl. rebinding narrative | Internal and metadata URLs leak only in vulnerable mode. The image URL works in both. |
| **6. Compare + Present** | Compare page, slides, timer, notes, presenter window | Full 6-minute run-through with a clicker works offline |
| **7. Polish + verify** | Content proofread (Arabic + facts + sources), a11y pass, projector contrast, perf | Checklist in §8 is all ticked |
| **8. Ship** | GitHub Pages via Actions + offline single-file build (`vite-plugin-singlefile`) on USB | Both open and run with Wi-Fi off |

---

## 8. Verification checklist

- **Unit (Vitest)**, for each engine: the vulnerable mode leaks, the secure mode blocks, and the legitimate action works in both modes.
- **E2E (Playwright, Chromium):**
  - each case page loads, and the mode toggle plus "replay" work
  - `dir="rtl"` is on `<html>`, and code blocks are `ltr`
  - **no-network test:** record every request the page makes and assert it is same-origin
    static assets only
- **Content:** each case's facts table matches §1, every source link opens, there is no "CVE"
  wording, and every glossary term has all 3 layers.
- **Rehearsal:** time two full runs. Test on a projector or TV, with a clicker, with Wi-Fi off.

---

## 9. Open questions (none block Phase 0)

1. **Presentation date?** Sets how much polish is realistic. Phases 0–6 are the must-have core.
2. **Venue setup:** your own laptop + projector? Internet available? (The plan assumes no.)
3. **TypeScript OK?** Plain JS also works if you'd rather explain simpler code to assessors.
4. **Student name / school branding** for the hero and title slide?
5. This repo is named `to-do-list`. Build here, or create a dedicated repo?

---

## 10. Sources

- Instagram: Laxman Muthiyah, "How I could have hacked any Instagram account", https://thezerohack.com/hack-any-instagram · SecurityWeek, https://www.securityweek.com/instagram-account-takeover-vulnerability-earns-hacker-30000 · WeLiveSecurity, https://www.welivesecurity.com/2019/07/16/instagram-account-could-have-been-hijacked
- Snapchat: HackerOne #530974, https://hackerone.com/reports/530974 · PortSwigger Daily Swig, https://portswigger.net/daily-swig/researchers-nab-4-000-bug-bounty-after-discovering-ssrf-vulnerability-in-snapchats-ad-platform
- Shopify: HackerOne #2207248, https://hackerone.com/reports/2207248
