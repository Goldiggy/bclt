# bclt: high-level plan (v0)

This document sets the overall vision, the core architectural bets, and the
phases. Each phase and each open question gets its own sub-plan in `docs/`
(see the list at the end).

---

## 1. Vision

> Instagram's photos and stories plus Snapchat's ephemeral messaging, with no
> company, server or government able to delete an account or silence a user.

The product rules:

1. **Users own their identity.** An account is a cryptographic key pair, not a
   row in someone's database. No operator can delete it.
2. **Users own their data.** Posts are signed by their author and can be
   copied to any server. If one host drops them, others still have them.
3. **No single point of failure.** No mandatory central server, DNS name,
   app store or company. Any of these can disappear and the network keeps
   working.
4. **Users control what they see.** Moderation is something users choose and
   stack, not something imposed at the network level.
5. **Feels like a mainstream app.** If it is harder to use than Instagram,
   only activists will use it. Decentralisation must be invisible by default.
6. **French first.** French UI and onboarding, French community, hosted under
   French/EU law, legal structure in France.

### What "censorship-resistant at every level" means

| Level | Censorship threat | Our defence |
|---|---|---|
| Account | Ban or deletion by the platform | Self-sovereign keys; no server can revoke them |
| Content | Server deletes posts | Signed, content-addressed data mirrored on several independent servers |
| Server / relay | Host seized, shut down or pressured | Many interchangeable relays run by different operators in different countries; clients publish to several |
| Network (ISP / state) | IP or DNS blocking, deep packet inspection | Multiple endpoints, Tor / onion services, pluggable transports, nearby peer-to-peer sync as a last resort |
| Distribution | App removed from the App Store / Play Store | F-Droid, direct APK, PWA, EU alternative marketplaces (DMA), reproducible builds |
| Organisation | Company pressured, bought or shut down | Open protocol, open source (AGPL), non-profit association, several independent client apps |
| Discovery | Shadow-banning, algorithm manipulation | Open, choosable feed algorithms; chronological feed always available |

### An honest limit: resistance ≠ no moderation

A network that is literally impossible to moderate attracts CSAM, spam and
illegal content first. That kills adoption and exposes operators in France
(hosts under LCEN / DSA) to prosecution. The model we adopt (similar to
Nostr and Bluesky) is:

- **Protocol level: neutral.** Nobody can erase an identity or stop a user
  publishing somewhere.
- **Operator level: each relay or media server applies its own rules and
  local law**, including mandatory removals (CSAM hash matching,
  court orders). Because there are many operators and users publish to
  several, no single operator's decision censors anyone network-wide.
- **User level: opt-in, composable moderation.** Block lists, labelling
  services, web-of-trust filters and feed choices.

That keeps the network resistant to censorship while letting individual
operators stay legal, which is what lets the network survive long-term.

---

## 2. Core architectural bets

These are proposals to validate in sub-plan **SP-1 (Protocol choice)**.

### 2.1 Build on an existing protocol

Inventing a new decentralised protocol is a multi-year project. The options:

| Protocol | Censorship resistance | Portability | Maturity / ecosystem | Fit for us |
|---|---|---|---|---|
| **Nostr** | Very high: dumb relays, keys only, trivial to run relays | Total (keys + multi-relay) | Good, many clients, photo apps exist (Olas) | **Recommended base** |
| AT Protocol (Bluesky) | Medium: portable, but relay/AppView are heavy and few | High | Good, large user base | Interesting for interoperability, later |
| ActivityPub (Mastodon, Pixelfed) | Low-medium: admins own accounts, defederation | Weak | Mature, popular in France | Bridge later, not core |
| Pure P2P (libp2p, IPFS, Holepunch) | Very high | Total | Weaker UX on mobile (battery, NAT) | Useful as a fallback transport |

**Proposal:** use Nostr as the data and identity layer, add our own NIPs
(extensions) only where needed, and add P2P transports as a fallback. Users
can then use other Nostr apps, and we get existing relays and libraries for
free.

### 2.2 Building blocks

- **Identity:** secp256k1 keys (Nostr), stored in the device secure enclave;
  human-friendly handles (`@lea@bclt.fr` via NIP-05, plus handles that don't
  depend on DNS); account recovery via seed phrase **and** social recovery
  (trusted friends), because "lost key = lost account" doesn't work for
  mainstream users.
- **Public content** (posts, comments, likes, follows): signed Nostr events
  published to the user's own set of relays ("outbox model").
- **Media** (photos, videos): content-addressed by hash, stored on several
  independent Blossom media servers, optional IPFS pinning; resized and
  EXIF-stripped on device (location privacy).
- **Private messaging and snaps:** end-to-end encrypted with **MLS
  (RFC 9420)** for 1:1 and groups, metadata minimised (gift-wrapped events /
  sealed sender).
- **Ephemerality (stories, snaps):** expiry timestamps (NIP-40) honoured by
  clients and relays, plus encryption with keys deleted after viewing.
  *Limit to state openly:* nothing stops a screenshot or a modified client;
  ephemerality is a social contract plus best effort, not a guarantee.
- **Feeds and discovery:** chronological by default; pluggable "feed
  algorithms" that users pick (local or third-party), French-language
  trending, hashtags, search relays.
- **Transport:** WebSocket over TLS to relays, onion endpoints for every
  official relay, Tor built into the app (optional), later local sync over
  Bluetooth / Wi-Fi Direct when the internet is cut.

### 2.3 Proposed stack (to confirm in SP-2)

- **Monorepo, TypeScript first** for speed: shared `core` library (protocol,
  crypto, storage, sync), **React Native / Expo** mobile app, web/PWA client
  from the same code.
- **Rust** for performance- or security-critical parts later (MLS via
  OpenMLS, media pipeline, our own relay if needed), exposed to the app.
- **Infrastructure:** existing open-source Nostr relays (strfry) and Blossom
  servers, deployed with Docker / Ansible, run on several French and EU hosts
  plus volunteer operators.
- **Licence:** AGPL-3.0 for apps and servers, MIT/Apache for the core
  library (so other clients adopt it).

---

## 3. Phases

Each phase ends with something usable. Durations are rough estimates for a
small team (2–4 developers).

### Phase 0: Foundations (2–4 weeks)
- Name, branding, French positioning.
- Threat model: who are the adversaries (platforms, states, harassers,
  spammers), what do we protect against.
- Protocol and stack decisions recorded as ADRs (`docs/adr/`).
- Legal: association loi 1901, operator responsibilities (LCEN, DSA, RGPD),
  legal counsel.
- Repo setup: monorepo, CI, linting, licences, contribution guide.

### Phase 1: Protocol prototype (4–6 weeks)
- `core` library: key generation, signing, publishing to and reading from
  several relays, outbox model, local cache (SQLite).
- CLI client to create an identity, post, follow and read a feed.
- Deploy 2–3 test relays plus 2 media servers.
- **Exit criterion:** shutting down any one server loses no data and stops
  no user.

### Phase 2: MVP "Instagram core" (2–3 months)
- Mobile app (iOS / Android) plus PWA.
- Onboarding in under a minute with no seed phrase shown up front (key
  generated and backed up in the background).
- Profile, photo/carousel posts, filters, chronological feed, likes,
  comments, follows, notifications.
- Multi-server media upload, on-device compression and EXIF removal.
- Basic moderation: block, mute, report to the relay operator.
- Closed beta with a French community.

### Phase 3: "Snapchat layer": stories and messaging (2–3 months)
- 24h stories, ephemeral snaps, view receipts.
- E2EE DMs and group chats (MLS), encrypted media.
- Camera-first UX, basic AR filters later.
- Social key recovery.

### Phase 4: Resilience and anti-censorship (ongoing, starts in phase 2)
- Built-in Tor, onion relays, automatic failover between relays.
- Relay discovery with no central list (gossip, signed relay lists).
- Distribution: F-Droid, signed APK, PWA, reproducible builds, mirrors.
- Offline / mesh mode: sync between nearby phones.
- Chaos tests: simulate blocking, server seizure and DNS takedown.

### Phase 5: Discovery and community moderation (2–3 months)
- French-language search and trending, hashtags, suggestions.
- Choosable feed algorithms.
- Labelling services and shared block lists, web-of-trust spam filtering.
- Tools for operators: CSAM hash matching, legal takedown workflow,
  transparency reports.

### Phase 6: Ecosystem and sustainability (ongoing)
- Governance (association, transparent decisions, published protocol spec).
- Funding: donations, grants (NLnet, NGI, BPI), optional paid premium
  storage, zaps / micro-payments between users (Lightning) if wanted.
- "Run a relay" kit for French associations, collectives and individuals.
- Bridges to Mastodon / Bluesky for reach.
- Public API and documentation for third-party clients.

---

## 4. Key risks

| Risk | Mitigation |
|---|---|
| Too complex for mainstream users | Hide keys by default, social recovery, UX tests with non-technical users |
| Illegal content and spam flood | Operator-level policies, opt-in moderation, proof-of-work / web of trust against spam |
| Legal exposure of the team and operators | Non-profit structure, clear operator ToS, counsel, distributed operators |
| Network effect: nobody joins | Nostr interoperability (existing users), French niche communities first, bridges |
| Infrastructure cost (media is expensive) | Several independent operators, paid storage tier, compression, grants |
| RGPD vs. immutable data | Signed deletion requests honoured by compliant relays; no promise of erasure from rogue nodes, stated clearly |
| App store removal | Alternative distribution prepared from day one |

---

## 5. Upcoming sub-plans

| ID | Sub-plan | Phase |
|---|---|---|
| SP-1 | Protocol choice: Nostr vs AT Proto vs custom, with prototypes | 0 |
| SP-2 | Tech stack and monorepo architecture | 0 |
| SP-3 | Threat model and security architecture | 0 |
| SP-4 | Legal and organisational framework (France / EU) | 0 |
| SP-5 | Identity, key management and account recovery | 1 |
| SP-6 | Data model: events, media, our NIP extensions | 1 |
| SP-7 | Infrastructure: relays, media servers, deployment | 1 |
| SP-8 | MVP mobile UX and design system | 2 |
| SP-9 | E2EE messaging and ephemeral content | 3 |
| SP-10 | Anti-censorship transport and distribution | 4 |
| SP-11 | Moderation, spam and trust | 5 |
| SP-12 | Governance, funding and community launch | 6 |

---

## 6. Open questions for the founder

1. **Team and skills:** who builds this, and with which languages (TypeScript,
   Rust, Kotlin/Swift…)?
2. **Budget and timeline:** side project, or funded with a launch date?
3. **Interoperability:** OK to build on Nostr (users can see their posts in
   other apps), or must bclt be a closed, separate network?
4. **Which feature defines v1:** the public photo feed (Instagram) or private
   ephemeral messaging (Snapchat)?
5. **Moderation stance:** do you accept the "neutral protocol, moderated
   operators, user-chosen filters" model in section 1?
6. **Name:** is "bclt" the final name, and what does it stand for?
