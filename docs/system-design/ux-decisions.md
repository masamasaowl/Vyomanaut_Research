# Vyomanaut — UX & Application Decisions

*The positioning, product, and desktop-application-engineering foundation for Vyomanaut. Stands alongside `architecture.md`, `requirements.md`, `data-model.md`, and `interface-contracts.md`. This is not a spec and not an ADR — it is the thinking those documents and future ADRs are built on. It replaces `ux-finding.md` and `desktop-application-foundations.md`, which are retired as of this document.*

> **Scope: LTS only.** As of the Milestone 16 Demo/LTS split, this entire document belongs to
> the LTS track. Demo ships no GUI — it is the CLI/simulated-daemon lifecycle in `mvp.md` §3.6,
> full stop — so there is no Demo-relevant content to segregate here, unlike the other four
> system-design documents. Every decision below, and every ADR it cites (ADR-038 through
> ADR-047, and ADR-049 through ADR-052), is realized starting at **Milestone 19**, the first
> milestone with any GUI work in it (§9's own note already anchored §9.7's accessibility work
> there; that anchor is now the rule for the whole document, not one subsection's exception).
> Nothing here depends on where the backend build (currently Milestone 16) happens to be —
> these are product and engineering-foundation decisions, not build-sequenced ones, and they
> hold regardless of backend milestone progress. Streamlined for the split: see the changelog
> note at the end of §11.

---

## 1. The problem

Storing data safely, cheaply, and without trusting one company is still hard to get right. The two existing answers both have a real cost:

- **Big cloud storage** (Google Drive, Dropbox, AWS, etc.) is easy to use but means trusting one company with your data, and paying its price.
- **Decentralized storage** (Storj, Filecoin, etc.) removes the single company, but pays you or charges you in a crypto token, and expects you to already know what a "node" is. It has stayed a hobbyist product for a decade.

Meanwhile, millions of ordinary desktop PCs — at home, in offices, in labs — sit mostly idle, disks half-empty, doing nothing outside working hours.

## 2. The solution — problem/solution fit

**Vyomanaut turns idle space on ordinary desktop computers into a storage network, paid in rupees, with no crypto and no command line for the people using it day to day.**

Two apps, one network:

- **A Data Owner app** — upload files, pay a predictable monthly amount, get them back when needed. As simple to use as any consumer storage app.
- **A Provider app** — turn on sharing, allocate some disk space, earn money. As simple to use as any "install and forget" income app.

The fit: data owners get lower prices and no single company holding their data hostage. Providers get income from a resource (idle disk space) they already own and aren't using. Vyomanaut earns the difference. Nobody involved needs to understand encryption, peer-to-peer networking, or blockchain to use it.

This is the whole pitch. Everything below exists to make sure we build it for the right people, and don't repeat mistakes the rest of the storage industry has already made in public — and, from §9 onward, to make sure the two apps that deliver it actually build and ship on the platform we're targeting first.

---

## 3. Where Vyomanaut stands in the storage industry

Storage is a 25-year-old industry with many well-defended corners. We looked at all of them. Vyomanaut competes in exactly one.

| Category | Who's there | Why we are not competing here |
| --- | --- | --- |
| **Consumer cloud storage** (Google Drive, Dropbox, iCloud, OneDrive) | ~$180–200B market. Dominated by ecosystem lock-in — people stay because of Gmail, Photos, Office, or their iPhone, not because of the storage itself. | We can't out-integrate an operating system or an office suite. This market isn't won on storage alone. |
| **Enterprise object storage** (S3, Azure Blob, Google Cloud Storage, Cloudflare R2) | The default for anything a company builds on. Competes on uptime guarantees, compliance certificates, and being next to the compute that uses the data. | Needs guaranteed uptime and formal compliance. A network of home PCs that switch off at night cannot promise that, and shouldn't pretend to. |
| **Block storage** (AWS EBS, Azure Managed Disks) | Acts as a raw, low-latency hard disk for a single server or database. | Wrong shape of problem entirely — this is millisecond-level, single-machine storage. We are file storage, not disk storage. |
| **File storage** (Amazon EFS, Azure Files) | Shared network drives for a cluster of machines in one data center. | Same issue — needs low, predictable latency inside one facility. Not what a home-PC network can offer. |
| **Distributed file systems** (HDFS, Ceph, GlusterFS, Lustre) | Software that turns a rack of *trusted, company-owned* machines into one storage pool. Used for big-data analytics and supercomputing. | Built for trusted hardware in one building, administered by one IT team. We assume every machine is untrusted and may vanish — a different engineering problem, not a smaller version of the same one. |
| **CDN / edge storage** (Cloudflare, Akamai, Fastly) | Delivers content fast to end users worldwide, backed by paid, professionally-run points of presence. | Requires guaranteed low latency and uptime from every node. A home PC that's asleep at 2 a.m. cannot be part of a CDN. |
| **Enterprise backup** (Backblaze, Wasabi, Veeam) | Competes purely on cost-per-terabyte and guaranteed retrieval speed, backed by real data centers. | It's a price war against companies with their own hardware and no per-node uncertainty. We'd lose that fight on their terms. |
| **Hybrid / enterprise storage hardware** (NetApp, Dell EMC, Pure Storage) | Sells physical storage appliances to large companies, with support contracts and multi-year refresh cycles. | We don't sell hardware, don't have an enterprise sales force, and this market doesn't want a software-only newcomer. |
| **Decentralized storage** (IPFS, Filecoin, Storj, Sia, Arweave) | Software-defined storage across independent, untrusted machines. **This is our actual category.** | This is where we compete — see below. |

**The one-line summary: everything above either needs guaranteed performance we can't promise from home PCs, or needs an enterprise sales motion we don't have. Decentralized storage is the only category built for untrusted, everyday hardware — which is exactly what we're using.**

## 4. Our real competition, and how we're different

Filecoin, Storj, Sia, and Arweave are real, growing businesses — decentralized storage is now a genuine, multi-billion-dollar sector, not a crypto experiment. But every one of them shares the same two problems:

1. **You get paid, and often pay, in a crypto token.** Every serious review of this sector says the same thing: normal people find the token economics confusing, and it's a real barrier to anyone outside the crypto-native audience.
2. **Their actual node operators are hobbyists, not mainstream users.** Storj's own setup guide recommends a NAS or Raspberry Pi and talks about port-forwarding and dynamic DNS. Its community forums read like a home-server enthusiast hobby, not a mainstream product — despite years of broader marketing.

**Vyomanaut's difference, in one sentence: real money (rupees, via UPI), not a token — and a target user who has never run a home server in their life.**

We are not trying to out-Storj Storj. We are not chasing the NAS/home-server crowd at all — that market is small and already served. We are going after the far larger population of people who simply own a desktop and have never thought about "running a node."

---

## 5. Who will actually use Vyomanaut

### 5.1 Data owners

Individuals, small businesses, and anyone who wants affordable, private file storage without trusting one big company with it. This side of the product is well understood and not controversial — consumer storage apps already prove this audience is reachable with the right app.

### 5.2 Providers — the audience is real, and the product should stay agnostic about how it reaches them

The instruction driving this document is clear: don't compete for the NAS crowd, go after the much bigger population of plain desktop and PC owners — starting with universities, offices, hospitals, and especially engineering students. The product itself should not bet on any one of these being the way in. It should work equally well for one student installing it on their own PC, and for an IT department that decides to roll it out across a lab — and be good enough that either path is viable whenever the business decides to pursue it. Building well doesn't require picking a lane first; business development happens after, and a strong enough product earns its way into whichever market suits it best.

That said, two facts from researching this space are worth knowing now, because they shape what "works for anyone" actually requires of the app itself — not because they require choosing a market up front:

**Turning idle desktops into a shared resource is a real, 30-year-proven idea.** HTCondor, built at the University of Wisconsin in the 1980s, has been harvesting idle lab and office desktops across "thousands of campuses, labs, and organizations" ever since. BOINC (SETI@home and similar) proved individuals will donate idle machine resources at huge scale — millions of hosts — when the ask is simple. Both models exist today, side by side, for the same underlying idea; the app should be equally at home being installed one machine at a time or handed to an IT administrator to roll out at scale.

**Institutional IT policy can be a real wall, and the app should be built to survive scrutiny either way.** Actual university IT policies commonly and explicitly ban peer-to-peer software by name (NYU's does, in exactly those words). This doesn't change what we build — it means the app should default to asking for exactly the permissions it needs and nothing more, explain itself clearly if a security team ever looks at it, and never assume it's welcome on a machine it didn't ask permission to run on. That's just good behavior for any desktop app, and it happens to also be what keeps the institutional door open for later. §10.2 and ADR-042 are the concrete engineering consequence of this paragraph — not a separate concern from it.

One genuine device-level fact worth carrying into design: engineering students mostly carry laptops, not desktops, and a laptop that moves between hostel, home, and classroom on battery power makes a worse fit for a service that wants to be on, plugged in, and connected for long stretches — a desktop's whole reason for being. This isn't a reason to avoid students as an audience; it's a reason the app should read the machine it's on (plugged in vs. on battery, for instance) and be honest with the person about what that means for them, rather than pretending every device is equally suited.

---

## 6. What great precedent apps teach us

We looked closely at four products that already do some version of "a background service, fronted by an app, for people who don't want to think about the background service."

- **Syncthing** proves the daemon-plus-app pattern works technically, but it never built a real native app — just a web page. It stayed a tool for technical users because of that, not despite it.
- **Tailscale** is the proof that a genuinely hard technical problem (secure networking) can be made to feel effortless with enough design investment — and it's the bar we should hold ourselves to *for design quality*. It is not, however, a precedent for *unprivileged Windows autostart*: Tailscale's Windows client runs as a genuine, admin-installed Windows Service, because driving a virtual network adapter requires kernel-level access no standard account has. Vyomanaut's provider daemon has no equivalent requirement, has no equivalent reason to pay that cost, and does not (§10.2, ADR-047). Worth also knowing: even Tailscale hasn't gotten a proper native app onto Linux yet, years in — excellent design takes real, sequenced effort, and we shouldn't assume simultaneous, equal polish on Windows, macOS, and Linux on day one.
- **Docker Desktop and MongoDB Compass** are the right model for what we're building: a real, polished native app that sits in front of something technical (a container engine, a database) and never makes the everyday user drop into a command line to get things done. The command line still exists underneath for the people who want it — it's just never the front door.
- **Honeygain-style "earn money from your idle device" apps** prove a mainstream, non-technical audience for this kind of product is real and large. But they succeed by asking almost nothing of the user and by keeping the user almost entirely uninvolved. That's explicitly not our model (§8) — we want people to feel part of what they're doing, not just glance at a number once a month.

---

## 7. The GUI mandate and platform order

**There is no command-line-only version of Vyomanaut for mainstream users.** A command line will exist underneath for engineers and power users who want it, exactly like Docker Desktop still has a `docker` CLI — but it is never the primary way anyone is expected to use this product, on any platform, including Linux. Both apps (Data Owner, Provider) are real, installed, native-feeling applications — not web pages opened in a browser tab.

**Windows is the first platform we build for and ship on**, full stop — not a "build once, adjust for other platforms" plan. Most Indian desktops — in homes, offices, and university labs — run Windows, and the product needs to be installable and testable the moment it launches, on the machines people actually have. This also happens to be the easiest platform for the shell choice in §9.1: Windows ships its own modern, auto-updating browser engine (WebView2), the same rendering-engine family Electron bundles, so the "smooth, premium" bar below is fully achievable on Windows without any of Electron's extra weight. On most Windows 11 machines it's already installed; where it isn't, the installer offers it automatically (§9.4).

macOS and Linux follow, matching the same design bar as closely as each platform's own browser engine allows — that's real, separate work each time, not an afterthought, and we're not pretending otherwise.

Concretely, both apps:

- Look and feel like clean, modern, premium software: minimal, uncluttered, confident use of type and whitespace, no dense settings screens dressed up as a "dashboard."
- Assume the person using them has never heard of erasure coding, peer-to-peer networks, or zero-knowledge encryption, and never needs to.

## 8. The experience: "Netflix," not "Honeygain"

Netflix hides an enormous, genuinely complicated streaming and recommendation system behind a simple, pleasant screen — but it doesn't hide *everything*. You still see your watch history, your list, what's trending. Vyomanaut should work the same way: hide the encryption, the network, the audits, the economics — and surface the parts a person actually wants to see and feel good about.

Concretely, the provider app's home screen should always show, clearly and without digging:

- **What you're earning**, and why it changed if it changed. (ADR-016 addendum and ADR-045 govern exactly what "why" discloses — the amount and a plain-language reason category, not the underlying scoring formula.)
- **How much storage you're sharing**, and its status (healthy, being repaired, etc.) in plain words.
- **Your impact** — a small, honest set of numbers, not a marketing slogan. The genuine, defensible version of this argument is: using disk space that already exists, on a machine that already exists, avoids building new dedicated data-center hardware and the industrial cooling that comes with it (cooling alone is roughly a third of a data center's total power use). That's a real, quantifiable claim we can stand behind — "you helped avoid building new server hardware," or an estimated-emissions-avoided figure — not a vague "you're saving the planet" line we can't back up.

This is also, deliberately, the opposite of the Honeygain model: those apps work precisely because the user is barely involved. We want the opposite — a user who checks in, feels informed, and feels like a participant in something real, because they are one.

---

## 9. Desktop application engineering foundations

*These are decisions, not options — made so engineering has a fixed foundation to build on when Milestone 19 starts. Each one has, or will get, its own ADR, all realized starting M19 per the scope note at the top of this document (ADR-038 through ADR-042 already cover §9.1–§9.4; §9.5 remains a design pass, not an ADR; ADR-049 covers §9.6; ADR-050 covers §9.7; ADR-051 covers §9.8; ADR-052 covers §9.9).*

### 9.1 Shell technology: Wails

**Decision: the desktop app is built with Wails, not Electron.**

Docker Desktop and MongoDB Compass — the two reference points for §7's GUI mandate — are both built on Electron, and it's worth being direct about why we're not following them there. Electron is the safe, proven choice when your backend is a separate thing entirely (Docker's is a virtual machine; MongoDB's is a database server you connect to over the network) and your team already thinks in JavaScript. Neither is true here. Vyomanaut's entire backend — the cryptography, the peer-to-peer network, the audit system, the payment logic, the API — is already written, and it's already Go. Wails takes that Go code and gives it a window, directly, with no second language and no second runtime sitting next to it. Electron would mean shipping a bundled Chromium and Node.js runtime (150–300MB, and a growing list of security patches to track and re-ship over the app's lifetime) purely to display a UI in front of a backend we've already built in a language that doesn't need any of that.

The one real gap Wails has — no built-in system tray — turns out not to be a blocker (§9.3): it's a known, documented gap in the current stable release, with a proven workaround confirmed working on Windows, our first platform. The next major version of Wails is already being built with a proper tray built in, and it's in active weekly development, not stalled — we'll move to it once it's stable; the migration only removes a workaround, it doesn't restructure anything.

**What this decides, concretely:**

- Both apps (Data Owner, Provider) are Wails apps: a Go backend we already have, a web-based frontend (HTML/CSS/JS, framework choice below) rendered in the operating system's own built-in browser engine, packaged as one native application.
- No Node.js, no Electron, anywhere in the shipped product.
- The existing `cmd/client` and `cmd/provider` logic is what the app runs — the app is a new front door on the same house, not a rebuild of it.

### 9.2 How the app talks to the daemon

**Decision: direct, in-process calls — no local web server, no open port.**

Storj and Syncthing both run a small web server on the user's machine and expect them to open it in a browser tab. We're deliberately not doing that: a web server on a loopback port is one more thing to secure (both of those projects have had to add authentication specifically because a bare local server isn't automatically safe), and it doesn't match §7's "real app, not a browser tab" mandate. Wails avoids this entirely — the frontend calls Go functions directly, in the same process, the way a normal function call works. Nothing is listening on a port for the UI's sake.

The REST API already built for Vyomanaut (used by `cmd/client`/`cmd/provider` today) doesn't go away — it stays as the interface for scripting, automation, and any future mobile app. It's just not how the desktop app itself talks to its own backend.

### 9.3 System tray, for real this time

**Decision: yes, on all platforms, starting with Windows — built with a small dedicated tray library running alongside the app, not inside Wails itself (Wails doesn't have its own yet).** This is a documented, working pattern, already proven in production apps, not an experiment.

The tray is the Provider app's main interface for routine use — status at a glance, pause, quit — matching the tray-first navigation model this implies. The full window opens for anything deeper. §10.2 (ADR-047) is what makes this possible on Windows without an elevated, Session-0 service standing in the way.

### 9.4 Packaging and signing, for Windows

- **Installer:** NSIS, built directly by Wails's own tooling (`wails build -nsis`). This also handles the WebView2 dependency automatically, so a user without it gets it installed as part of setup, not as a separate confusing step.
- **Code signing:** required from the first build we let anyone outside the team install, not added later. An unsigned installer gets flagged by Windows and antivirus tools by default — this is normal and expected for any new desktop app, not a Vyomanaut-specific problem, but it's a real trust barrier for exactly the non-technical end of our audience, and it compounds with the Electron-specific version of the same problem we already avoided in §9.1. Azure's newer code-signing service (Trusted Signing) is the practical way to do this without the cost and paperwork of a traditional multi-year certificate, and fits a small team's budget.
- **Automated builds:** a straightforward CI pipeline (Wails already has ready-made GitHub Actions tooling for this) should produce a signed, installable Windows build on every release from day one — "ready to test and integrate immediately after launch" is a pipeline decision as much as a product one, and it's solved, not research.

### 9.5 Design system — making "premium and minimal" concrete

A design mandate only helps engineering if it turns into specific, checkable choices. These are the starting primitives; a designer should refine the exact values, but the *kind* of choice is decided:

- **Typography:** one confident, modern system typeface (Windows' own Segoe UI Variable is a legitimate, free starting point — it was built for exactly this kind of clean, native-feeling look on Windows), a small number of sizes and weights used consistently, generous line height. No more than two typefaces in the entire app, ever.
- **Color:** a small, restrained palette — a neutral base (whites, greys, one dark-mode equivalent) plus a single accent color used sparingly for actions and status. Status colors (healthy, degraded, error) are consistent everywhere they appear, tied directly to the error/status wording already defined in `interface-contracts.md` §14 so the same event never looks or reads differently in two places.
- **Spacing and layout:** one consistent spacing scale (e.g., multiples of 4px) used everywhere, generous whitespace over dense information, no screen that tries to show everything at once. If a screen feels crowded, it's split into two screens, not shrunk to fit.
- **Motion:** short, purposeful transitions (opening a window, switching a tab, a status changing) — never decorative animation for its own sake. Motion should make state changes easier to follow, not slower to get through.
- **One component system, shared by both apps.** The Data Owner and Provider apps are different products with the same visual language — a shared internal component library (buttons, cards, status badges, the toast/error system already defined in the interface contracts) is what keeps them feeling like one company made both, and keeps a design change from having to happen twice.

### 9.6 Frontend framework: Svelte + TypeScript

**Decision: the frontend inside the Wails shell is Svelte 5 with TypeScript, using Wails' official `svelte-ts` template.**

Wails ships six official templates — Svelte, React, Vue, Preact, Lit, and Vanilla, each in JavaScript and TypeScript variants — so this was a real choice, not a default. Svelte compiles reactivity away at build time instead of shipping a virtual-DOM runtime, which wins on the same runtime-footprint grounds §9.1 already used to pick Wails over Electron, and its syntax stays closer to plain HTML/CSS/JS than React's or Vue's — the same "closest to what a Go-first team already half-knows" reasoning applied one layer up the stack. TypeScript is chosen specifically so the type definitions Wails already auto-generates from Go structs and functions get consumed with real compile-time checking, instead of that generated typing going unused behind a JavaScript frontend.

The single shared component system §9.5 requires lives in its own package, `@vyomanaut/ui` (Svelte + Vite, library mode), linked into both app frontends through a pnpm workspace — a build-time dependency, not a runtime one, so each app still embeds only its own compiled output via `embed.FS` and ships independently. Full decision, including the options considered and rejected: ADR-049, Paper 51.

### 9.7 Accessibility baseline

**Decision: WCAG 2.1 Level AA, built into the design system from the start, not added later.**

§5.2's institutional audience — universities, offices, hospitals — includes a real share of government and government-funded organizations, and India's accessibility framework for them (the RPwD Act 2016, IS 17802, GIGW 3.0) is not just a website concern; IS 17802 explicitly covers software. This is the same "survive scrutiny before it happens" posture §5.2 and §10.2 already take toward institutional IT policy, applied to a different kind of scrutiny. Every custom component in `@vyomanaut/ui` is built with correct semantic HTML and ARIA, a minimum 4.5:1 text contrast ratio, full keyboard operability, and a visible focus indicator — cheap now, at the design-system-primitives stage §9.5 is still at, and expensive to retrofit into a shipped component library later.

Correct markup alone isn't sufficient proof this works, though: the bridge between markup and an actual screen reader has had real, documented gaps in exactly this stack — a Microsoft-confirmed WebView2 keyboard-focus bug (fixed in Windows App SDK 1.5, not independently confirmed against Wails' own WebView2 hosting) and a currently open Wails-specific screen-reader focus bug on Linux. Before the M19 shared component library is considered done, the first built screens get a direct Windows Narrator smoke test against the real packaged build — not an inference from the spec. Formal IS 17802 certification isn't pursued at this stage; like the rest of institutional go-to-market timing (§11), that's a decision for when the business actually needs it, not before. Full decision: ADR-050, Paper 52.

### 9.8 Keeping the app up to date

**Decision: a two-phase update mechanism — silent NSIS reinstall now, Wails v3's built-in updater once it's stable and confirmed shipped.**

Neither app has a way to receive updates today, which is a real, standing security gap, not just a missing convenience — and it's worth being honest that Wails v2 has no built-in answer for this (a years-old feature request for one was never resolved) and its `embed.FS` asset model doesn't cleanly support a custom partial-update workaround either. Wails v3 does solve this properly — GitHub Releases as an update source, SHA-256 and Ed25519 verification, an in-place binary swap, smaller downloads via delta patches — but it's still beta, the framework's own release notes recommend keeping v2 apps in place until v3 is actually ready, and the updater feature itself isn't confirmed to have landed in a tagged release yet, only demonstrated against one still on a branch.

So the app updates itself now the way Tailscale eventually learned to: not with a custom binary-swap tool, but by silently re-running the same installer it already ships (§9.4's signed NSIS build, via NSIS's own `/S` flag) against a version it checks for on GitHub Releases. Tailscale's own Windows auto-updater took years to ship, and what it landed on wasn't a from-scratch mechanism — it was reusing the install path that already existed. The moment Wails v3 is stable and its updater is confirmed to actually be in a tagged release, the app moves to it and the interim mechanism retires. Full decision: ADR-051, Paper 53.

### 9.9 Locale-ready copy and number formatting

**Decision: all user-facing text goes through Paraglide JS message functions, not hardcoded strings — and every number and rupee amount uses `Intl.NumberFormat("en-IN", ...)`, unconditionally.**

These are two different questions wearing the same "localization" coat, and only one of them is actually gated on the language-scope business decision §11 defers. Whether the app ever ships in anything other than English is a business call, deliberately not answered here. Whether a rupee amount displays as "₹12,34,567" instead of "₹1,234,567" is not a business call at all — it's a correctness question for the exact Indian audience §5.2 and §8 are already built for, and JavaScript's native `Intl.NumberFormat("en-IN", ...)` gets this right today, at zero dependency cost, independent of what language the surrounding text is in.

The copy side is cheaper to decide now than to retrofit later, the same reasoning §9.7 already used for accessibility: user-facing text is authored as keyed Paraglide JS messages — a compiler-based system that fits ADR-049's own compile-time philosophy and works in a plain Svelte + Vite project without needing SvelteKit — sourced from `interface-contracts.md` §14's copy table once it merges. This doesn't commit to shipping a second language; it just means the English strings V1 ships with are already structured so that adding one later doesn't mean rewriting every component's text. Full decision: ADR-052, Paper 54.

---

## 10. Provider build risk: what "Windows first" actually requires underneath

§7 commits to Windows shipping first. Two things had to be true underneath that commitment for it to actually be buildable, and neither was, until now.

### 10.1 The provider storage engine had no Windows build path

The provider daemon's chunk store (ADR-023) is built on RocksDB via a CGo binding, with a working, CI-tested build on Linux and macOS only. Upstream RocksDB's Windows support is MSVC-only, which conflicts with the MinGW-family compiler Go's CGo toolchain conventionally expects on Windows — a real toolchain gap, not a hypothetical one, confirmed against upstream RocksDB's own issue tracker (Paper 49). **Windows builds of the provider storage engine use BadgerDB instead** — a pure-Go, CGo-free implementation of the same WiscKey design ADR-023 is already grounded in. Linux and macOS are unchanged. Full decision: ADR-046, addendum to ADR-023.

### 10.2 The daemon's auto-start requirement, as written, couldn't coexist with the tray or the privilege model

`requirements.md`'s FR-031 named "Windows Service" as the auto-start mechanism. A genuine Windows Service needs administrator rights to install (conflicting with the Provider app's no-elevation-ever decision, §5.2/ADR-042) and runs in a session architecturally walled off from the interactive desktop, unable to show a tray icon at all (conflicting with §9.3's tray-as-primary-interface). **The provider daemon auto-starts via a per-user Windows Task Scheduler logon trigger** — no elevation, registered by the installer under the installing user's own account, running in the same interactive session the tray needs. Full decision, including why Tailscale (§6) is not the applicable precedent here: ADR-047, Paper 50.

Both were unaddressed gaps between an accepted architecture decision and an accepted product requirement — exactly the kind of thing that should surface before a build milestone starts on it, not partway through.

---

## 11. What still needs deciding (not answered here, on purpose)

- **macOS- and Linux-specific packaging, signing, and autostart mechanisms** — deferred until the Windows version is real and in front of users, per §7. FR-031's "macOS LaunchDaemon" / "Linux systemd unit" wording needs the same system-wide-vs-per-user correction §10.2 made for Windows, once those platforms are actually being built.
- **The exact color palette, type scale, and spacing values** — §9.5 fixes the *kind* of system, not the final numbers; that's a focused design pass, not an open research question.
- **Whether BadgerDB should replace RocksDB on Linux/macOS too**, not just Windows (Q49-1) — Badger's own published comparisons, and independent recent large-value-KV-store research, suggest it may outperform general-purpose RocksDB at Vyomanaut's fixed 256 KB chunk size regardless of platform. Not blocking — the existing RocksDB path on Linux/macOS works and is CI-proven — but worth measuring against real provider hardware post-launch rather than leaving two storage engines in place indefinitely by default.
- **How and when we approach institutions** (universities, offices, hospitals) as a go-to-market motion — a business-development question, deliberately not a product one (§5.2). The product is built to work either way; this is about timing and outreach, not design. Formal IS 17802 accessibility conformance certification (§9.7, ADR-050) is deferred to this same decision — the WCAG 2.1 AA baseline is already built in regardless of when, or whether, that outreach happens.

Everything else in this document — the problem, the solution, the market we're in, the market we're not in, who we're building for, the design bar we're holding ourselves to, and the desktop-application foundations that make "Windows first" actually buildable — is intended to be settled.

---

**Changelog — Milestone 16, Demo/LTS split.** Streamlined for the split: added the LTS-only
scope banner at the top (this document had no Demo-relevant content to begin with — Demo has
no GUI — so streamlining here meant making that explicit and removing backend-milestone framing
that could read as a dependency, not cutting decisions); tied §9's ADR list and this section's
Windows-service finding to Milestone 19 explicitly rather than leaving the connection implicit
in §9.7 alone. No product, market, or engineering decision in §§1–10 changed.
