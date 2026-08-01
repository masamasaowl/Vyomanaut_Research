# Vyomanaut — UX Finding

*A positioning and user-experience foundation document. Stands alongside `architecture.md`, `requirements.md`, and the other core system-design docs. This is not a spec and not an ADR — it is the thinking those documents and future ADRs should be built on.*

*Companion document: `desktop-application-foundations.md` covers the desktop shell technology and other engineering primitives. Milestones are still out of scope for both documents — those come once the foundations are agreed.*

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

**Institutional IT policy can be a real wall, and the app should be built to survive scrutiny either way.** Actual university IT policies commonly and explicitly ban peer-to-peer software by name (NYU's does, in exactly those words). This doesn't change what we build — it means the app should default to asking for exactly the permissions it needs and nothing more, explain itself clearly if a security team ever looks at it, and never assume it's welcome on a machine it didn't ask permission to run on. That's just good behavior for any desktop app, and it happens to also be what keeps the institutional door open for later.

One genuine device-level fact worth carrying into design: engineering students mostly carry laptops, not desktops, and a laptop that moves between hostel, home, and classroom on battery power makes a worse fit for a service that wants to be on, plugged in, and connected for long stretches — a desktop's whole reason for being. This isn't a reason to avoid students as an audience; it's a reason the app should read the machine it's on (plugged in vs. on battery, for instance) and be honest with the person about what that means for them, rather than pretending every device is equally suited.

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
- It looks and feels like clean, modern, premium software: minimal, uncluttered, confident use of type and whitespace, no dense settings screens dressed up as a "dashboard."
- **Windows is the first platform we build for and ship on.** Most Indian desktops — in homes, in offices, in university labs — run Windows, and the product needs to be ready to test and adopt the moment it launches, on the machines people actually have. macOS and Linux follow, matching the same design bar as closely as each platform allows.
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

- **The desktop shell technology and the other engineering primitives it implies** — decided separately, in `desktop-application-foundations.md`, so that decision gets the dedicated attention it needs rather than a paragraph here.
- **How and when we approach institutions** (universities, offices, hospitals) as a go-to-market motion — a business-development question, deliberately not a product one (§5.2). The product is built to work either way; this is about timing and outreach, not design.

Everything else in this document — the problem, the solution, the market we're in, the market we're not in, who we're building for, and the design bar we're holding ourselves to — is intended to be settled.
