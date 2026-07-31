# Vyomanaut — UX Finding

*A positioning and user-experience foundation document. Stands alongside `architecture.md`, `requirements.md`, and the other core system-design docs. This is not a spec and not an ADR — it is the thinking those documents and future ADRs should be built on.*

*Out of scope, on purpose: the desktop shell framework (Electron/Wails/Tauri/etc.) and any milestone plan. Both come after this document is agreed.*

---

## 1. The problem

Storing data safely, cheaply, and without trusting one company is still hard to get right. The two existing answers both have a real cost:

- **Big cloud storage (Google Drive, Dropbox, AWS, etc.)** is easy to use but means trusting one company with your data, and paying its price.
- **Decentralized storage (Storj, Filecoin, etc.)** removes the single company, but pays you or charges you in a crypto token, and expects you to already know what a "node" is. It has stayed a hobbyist product for a decade.

Meanwhile, millions of ordinary desktop PCs — at home, in offices, in labs — sit mostly idle, disks half-empty, doing nothing outside working hours.

## 2. The solution — problem/solution fit

**Vyomanaut turns idle space on ordinary desktop computers into a storage network, paid in rupees, with no crypto and no command line for the people using it day to day.**

Two apps, one network:

- **A Data Owner app** — upload files, pay a predictable monthly amount, get them back when needed. As simple to use as any consumer storage app.
- **A Provider app** — turn on sharing, allocate some disk space, earn money. As simple to use as any "install and forget" income app.

The fit: data owners get lower prices and no single company holding their data hostage. Providers get income from a resource (idle disk space) they already own and aren't using. Vyomanaut earns the difference. Nobody involved needs to understand encryption, peer-to-peer networking, or blockchain to use it.

This is the whole pitch. Everything below exists to make sure we build it for the right people, and don't repeat mistakes the rest of the storage industry has already made in public.

---

## 3. Where Vyomanaut stands in the storage industry

Storage is a 25-year-old industry with many well-defended corners. We looked at all of them. Vyomanaut competes in exactly one.

| Category | Who's there | Why we are not competing here |
|---|---|---|
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

### 5.2 Providers — this is the persona that needs real thought

The instruction driving this document is clear: don't compete for the NAS crowd, go after the much bigger population of plain desktop and PC owners — starting with universities, offices, hospitals, and especially engineering students. Here is what the research says about that plan, including the parts that need to be handled carefully rather than assumed.

**It's a real, proven idea — just not a new one.** Turning idle desktops across an institution into a shared resource is exactly what university computing has done for over 30 years:

- **HTCondor**, built at the University of Wisconsin in the 1980s, has been harvesting idle lab and office desktops across "thousands of campuses, labs, and organizations" ever since — precisely the model we're proposing, just for compute instead of storage.
- **BOINC** (SETI@home and similar projects) proved individuals will donate idle machine resources at huge scale — millions of hosts — when the ask is simple and the cause feels worthwhile.

So the instinct is sound. Two things from that same history need to be carried over honestly, not skipped:

**First: "students" and "institutions" are two different sales motions, not one.** HTCondor's campus deployments are installed by the university's own IT or research-computing staff, machine-wide, with permission. BOINC's volunteers install it themselves, one machine at a time, with no one's permission needed. Vyomanaut needs to pick which motion it's running for universities/offices/hospitals — a slower, IT-approved rollout across many machines at once, or a faster, one-student-at-a-time install with no institutional backing. These lead to different first versions of the app and different first conversations we have. This should be a deliberate choice, not something we back into.

**Second: institutional IT policy is a real wall, not a formality.** We looked at actual university IT policies. They commonly and explicitly ban exactly this category of software. NYU's policy bans "peer-to-peer file sharing software" by name. Other universities require every piece of lab software to be pre-approved and security-reviewed before it's installed. This doesn't mean the plan is wrong — it means the institutional route needs an actual conversation with an IT department, not just a download link, and the individual/student route needs to default to personally-owned machines, not the university's own lab PCs, unless we have that conversation first.

**Third: engineering students mostly carry laptops, not desktops.** This matters more than it sounds like it should. A laptop is unplugged, moved between hostel, home, and classroom, and usually has far less spare disk space than a desktop. Vyomanaut needs a machine that's on, plugged in, and connected for long, predictable stretches — a desktop's whole reason for being. Engineering students are still exactly the right *audience* — young, tech-comfortable, exactly the "forward-thinking mindset" the brief calls for — but the best *device* target within that audience is the desktop lined up in their college's own computer lab, not the laptop in their bag. That again points back to needing an institutional conversation, not just an app.

**The honest summary:** the target is right — plain desktop and PC owners, at real scale, via universities/offices/hospitals and the students inside them — but reaching it will need a real relationship with a handful of institutions early on, not just a polished download page. That's a go-to-market decision, not a UX one, but the UX (and the first version of the app) should be built with it in mind.

---

## 6. What great precedent apps teach us

We looked closely at four products that already do some version of "a background service, fronted by an app, for people who don't want to think about the background service."

- **Syncthing** proves the daemon-plus-app pattern works technically, but it never built a real native app — just a web page. It stayed a tool for technical users because of that, not despite it.
- **Tailscale** is the proof that a genuinely hard technical problem (secure networking) can be made to feel effortless with enough design investment — and it's the bar we should hold ourselves to. Worth knowing: even Tailscale hasn't gotten a proper native app onto Linux yet, years in. Excellent design still takes real, sequenced effort — we shouldn't assume simultaneous, equal polish on Windows, macOS, and Linux on day one.
- **Docker Desktop and MongoDB Compass** are the right model for what we're building: a real, polished native app that sits in front of something technical (a container engine, a database) and never makes the everyday user drop into a command line to get things done. The command line still exists underneath for the people who want it — it's just never the front door.
- **Honeygain-style "earn money from your idle device" apps** prove a mainstream, non-technical audience for this kind of product is real and large. But they succeed by asking almost nothing of the user and by keeping the user almost entirely uninvolved. That's explicitly not our model (see below) — we want people to feel part of what they're doing, not just glance at a number once a month.

---

## 7. The GUI mandate, stated plainly

**There is no command-line-only version of Vyomanaut for mainstream users.** A command line will exist underneath for engineers and power users who want it, exactly like Docker Desktop still has a `docker` CLI — but it is never the primary way anyone is expected to use this product, on any platform, including Linux.

Concretely:

- The app is a real, installed, native-feeling application — not a web page opened in a browser tab.
- It looks and feels like clean, modern macOS software: minimal, uncluttered, confident use of type and whitespace, no dense settings screens dressed up as a "dashboard."
- **macOS is the first platform we build for and hold to this bar.** Windows and Linux follow, matching it as closely as each platform allows — a web-based interface can be used to help reach Windows/Linux faster where it doesn't compromise the feel of the app, but macOS is where "done right" is defined.
- Every screen assumes the person using it has never heard of erasure coding, peer-to-peer networks, or zero-knowledge encryption, and never needs to.

## 8. The experience: "Netflix," not "Honeygain"

Netflix hides an enormous, genuinely complicated streaming and recommendation system behind a simple, pleasant screen — but it doesn't hide *everything*. You still see your watch history, your list, what's trending. Vyomanaut should work the same way: hide the encryption, the network, the audits, the economics — and surface the parts a person actually wants to see and feel good about.

Concretely, the provider app's home screen should always show, clearly and without digging:

- **What you're earning**, and why it changed if it changed.
- **How much storage you're sharing**, and its status (healthy, being repaired, etc.) in plain words.
- **Your impact** — a small, honest set of numbers, not a marketing slogan. The genuine, defensible version of this argument is: using disk space that already exists, on a machine that already exists, avoids building new dedicated data-center hardware and the industrial cooling that comes with it (cooling alone is roughly a third of a data center's total power use). That's a real, quantifiable claim we can stand behind — "you helped avoid building new server hardware," or an estimated-emissions-avoided figure — not a vague "you're saving the planet" line we can't back up.

This is also, deliberately, the opposite of the Honeygain model: those apps work precisely because the user is barely involved. We want the opposite — a user who checks in, feels informed, and feels like a participant in something real, because they are one.

---

## 9. What still needs deciding (not answered here, on purpose)

- **The desktop shell technology** (how the app is actually built) — deliberately left open. This choice affects how much creative and engineering freedom we keep, and shouldn't be locked in before the rest of this document is agreed.
- **Which go-to-market motion we run first for providers** — institution-led (universities/offices/hospitals, IT-approved, high machine count per deal) versus individual-led (one student or PC owner at a time, no institutional approval needed). See §5.2 — this changes what the first version of the provider app needs to do.
- **How we start institutional conversations** (universities, offices, hospitals) given the real IT-policy barrier in §5.2 — this is a business development question as much as a product one, and should be looked at before we assume lab machines as a v1 target.

Everything else in this document — the problem, the solution, the market we're in, the market we're not in, who we're building for, and the design bar we're holding ourselves to — is intended to be settled.
