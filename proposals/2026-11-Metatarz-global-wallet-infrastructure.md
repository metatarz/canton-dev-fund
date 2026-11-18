## Global Wallet Infrastructure

| Field | Value |
| --- | --- |
| **Organization** | Metatarz |
| **Author / Primary Contact** | Meta Tarz, meta@metatarz.xyz |
| **Status** | Draft |
| **Created** | 2026-11-05 |
| **Proposal Type** | RFP-aligned |
| **RFP / Roadmap Area** | RFP 3, Automated Application Management; RFP 14, Wallet and dApp Integration tooling |
| **Aligned SIG** | Wallet Apps; DAR Package Management & App Lifecycle |
| **Champion** | `Needs Champion` |
| **Related CIPs** | [CIP-0103](https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md) (dApp API); [WalletConnect Canton namespace](https://docs.walletconnect.com/wallets/chains/canton) |
| **Related proposals** | [dApp SDK](2026-03-DA-Canton-Network-dapp-sdk-and-tooling.md), [Wallet Gateway](wallet-gateway-reference-implementation.md), [Daml Package Registry](2026-07-Obsidian-Package-Distribution.md), [Daml Package Analyzer](2026-03-Certora-daml_package_analyzer_proposal%20.md) |
| **Total Funding Request** | 4,500,000 CC (2,900,000 CC delivery, 400,000 CC maintenance, 1,200,000 CC adoption) |
| **Project Duration** | 9 months to MainNet, then 12 months of maintenance and 6 months for adoption |
| **Label** | `dar-app-management` |

---

## Abstract

CIP-0103 lets any wallet connect to any app. In practice, the connection often fails at the first transaction. The reason is that the Validator hosting the user's party has not vetted the app's Daml packages. Canton only accepts a transaction if every Validator hosting the parties involved has vetted the packages it uses. Vetting is done by hand by the Validator operator.

So apps work around it. Many build an in-wallet app inside one or two wallets whose operators already vetted their packages. They then recommend that path over any other wallet connection. Users end up tied to specific wallets, or with a separate party for each app. That works against the idea of an open wallet standard and results in a high number of inactive parties in the system which negatively affects party scalability. 

We propose to fix this with a simple flow. Each app publishes the list of packages it needs. When the user approves a wallet connection, the wallet checks whether the user's Validator has those packages. If it doesn't, the approval screen also shows what the app wants to install. A small service next to the Validator, run by the operator, then uploads and vets the packages according to rules the operator sets.

The work delivers:

- package checks that happen when a CIP-0103 or WalletConnect session is approved, one new method (`canton_requestPackages`) for use during a session, and a fallback when a transaction fails;
- an open registry of app package manifests;
- an open-source Validator Sidecar;
- a wallet library.

Everything is built on existing Ledger API endpoints. No protocol changes are needed.

---

## Specification

### 1. Objective

A user should be able to connect any CIP-0103 wallet to any listed app and transact. This should not need the wallet operator to vet the app's packages by hand, and it should not need the app to ship a wallet-specific integration.

### 2. Implementation Mechanics

**App manifest.** Each app publishes a manifest at `https://<app-origin>/.well-known/canton-app.json`, and a copy is listed in a public Git registry. The manifest has:
- the app name and its web origins;
- the packages the app needs, each with its package name, package ID and DAR SHA-256 hash;
- where to download each DAR;
- optionally, links to audits in the Daml Package Registry.

Because the manifest is served from the app's own domain, a phishing site cannot claim another app's packages. The Git registry gives wallets and operators a single place to browse and review apps. Each wallet always checks the registry copy against the one on the app's domain.

**Packages in the wallet session.** We don't add a new way to connect. Apps keep using the existing CIP-0103 connect, or WalletConnect with the `canton` namespace. The package check happens in three places:

1. **When the session is approved.** WalletConnect tells the wallet which site is asking (`metadata.url`), and its Verify API checks that the domain is genuine. The wallet loads the manifest from that domain. If packages are missing, the approval screen also says "this app needs to install packages X, Y". An app can also send its package list directly in the session proposal's `sessionProperties`. For CIP-0103 connections outside WalletConnect, the wallet uses the origin the dApp SDK reports.
2. **During a session.** An app that has upgraded may need a new package version partway through a session. For this case we add one method, modelled on EVM's `wallet_watchAsset` and `wallet_addEthereumChain`, where an app asks the wallet to add something and the wallet asks the user:
   - `canton_requestPackages({ packages })` returns `ready`, `pending` (waiting for operator approval) or `rejected`;
   - `canton_getPackageStatus()` returns the current state;
   - a `packagesChanged` event fires when the state changes.
3. **When a transaction fails.** If `canton_prepareSignExecute` fails because a package isn't vetted, the wallet looks up the app's manifest and offers the same approval. Apps that change nothing still get the fix.

The method and event are specified once at the CIP-0103 level, with a mapping to WalletConnect. The exact names will be agreed with the CIP-0103 and dApp SDK maintainers.

**Flow.**
1. The user connects a wallet to an app, or the app calls `canton_requestPackages`.
2. The wallet loads the app's manifest. It then asks the Ledger API's preferred-packages endpoint whether every Validator hosting the user's party has vetted those packages.
3. If they all have, the result is `ready` and nothing else happens.
4. If they haven't, the wallet shows the user:
   - which packages the app wants to install;
   - whether those packages have a published audit;
   - the results of the Daml Package Analyzer.
5. If the user approves, the wallet sends a request to the Sidecar on each Validator hosting the party. The request is signed with the user's party key.
6. Each Sidecar downloads the DAR and checks its hash against the manifest. It then applies the operator's policy: approve it automatically, put it in a queue for the operator, or reject it.
7. When a request is approved, the Sidecar uploads the DAR and runs the vetting update as a dry run. If the dry run passes, it vets the packages for real. The app gets `ready`.

**Validator Sidecar.** This is a small open-source service shipped as a Docker image and Helm chart. The operator runs it next to an existing Splice Validator, which needs no changes. It uses the Ledger API's package upload and `package-vetting/update` endpoints. It does the following:

- **Checks who is asking.** It only accepts requests signed by the key of a party the Validator hosts.
- **Applies the operator's policy.** Examples are auto-approving apps from an allowlist, auto-approving packages that have a published audit, sending everything else to manual review, and setting rate limits.
- **Blocks silent upgrades.** Vetting is per Validator, so vetting a newer version of a package changes which version every party on that node uses. The Sidecar never vets a newer version of an already-vetted package without explicit operator approval.
- **Keeps a request log.** It records which party asked for which packages and what the policy decided, so the operator can review it and spot abuse.
- **Leaves unvetting to the operator.** Unvetting a package that any hosted party still uses in active contracts would leave those contracts stuck. The Sidecar never unvets on its own. It can show the operator which vetted packages no hosted party currently uses, based on the ledger itself.
- **Keeps multi-host parties consistent.** When a party is hosted on more than one Validator, the wallet sends the request to every host. If any host fails, the wallet reports it, so a party is never left with packages vetted on some of its hosts and not others.
- **Is observable.** It provides an audit log, metrics and an operator UI for the approval queue.

**Wallet library.** This is a TypeScript package for wallet developers, built to work with the Splice Wallet SDK and Wallet Gateway. It covers manifest loading and checks, the readiness check, the approval screen data and the request signing, together with session-approval hooks for the Wallet Gateway and WalletConnect, the `canton_requestPackages` handler and the fallback when a transaction fails. We will offer it upstream to the Wallet SDK.

### 3. Architectural Alignment

RFP 3 asks for Validator tooling covering *"application discovery, review & security analysis; approval, installation and upgrading,"* where *"individual parties… may choose to vet and/or unvet Daml packages, across all Validators with hosting rights for a given party."* This proposal delivers that at the tooling level. The party asks for packages to be vetted, and the operator's policy decides. Vetting in Canton is per Validator, so the operator remains in control. True per-party vetting would need a protocol change, which is out of scope here. We will share what we learn with the DAR Package Management SIG as input for that work.

RFP 14 asks for better *"signing flows, account/party management, and application-to-wallet interactions."* This proposal removes the most common reason a CIP-0103 connection fails after it has been made.

The work uses only existing parts of Canton: CIP-0103, the Ledger API endpoints for package upload, vetting and preferred packages, and external party signatures. It also connects two funded projects to the place where vetting decisions are made. The Daml Package Registry (Obsidian) provides audit status. The Daml Package Analyzer (Certora) provides safety analysis. The Obsidian proposal lists per-network vetting state as out of its scope, and this work fills that gap.

This work also makes multi-hosting and party replication easier. Before a party can be copied to a new Validator, that Validator must vet every package the party's contracts use. Today this is a manual check. The Decentralization Manager docs warn that if a package is missing, *"the import fails after the joining host has left the synchronizer."* With this work in place, each party's approved packages are recorded, so a wallet or operator can get them vetted on the new host before the copy starts. The online party replication being built in [Party Mobility Phase 1](../rfps/protocol-infrastructure-scalability-resilience/2026-09-DA-party-mobility-phase-1-online-party-replication.md), and multi-host setups in general, can then treat package vetting as an automatic step instead of a manual blocker.

### 4. Backward Compatibility

*No backward compatibility impact.* Everything in this proposal is opt-in:

- The CIP-0103 methods are new, so wallets that don't support them return the standard "method not supported" error.
- Validators without the Sidecar keep vetting by hand.
- In-wallet apps keep working.

---

## Milestones and Deliverables

This proposal has four milestones:

- **Milestone 1** releases the flow for apps that operators have chosen in advance.
- **Milestone 2** opens it to any app, protected by checks on the packages themselves rather than an allowlist.
- **Milestone 3** covers 12 months of maintenance.
- **Milestone 4** is paid only for adoption by teams other than Metatarz.

Metatarz runs its own wallet and Validator with the flow throughout. Committed dates assume the grant is approved by 2026-12-31. If approval comes later, each date moves by the same amount.

### Milestone 1: Package Requests for Allowlisted Apps

| Field | Details |
| :---- | :---- |
| **Estimated Delivery** | 4 months after grant approval |
| **Committed Date** | 2027-04-30 |
| **Focus** | `canton_requestPackages` and the check at session approval. These let an allowlisted app get its DARs vetted for its users on a Validator whose operator has allowlisted that app. |
| **Team** | Metatarz |

**Deliverables:**

- Specification v0.9:
  - the app manifest format;
  - the package check when a CIP-0103 or WalletConnect session is approved;
  - `canton_requestPackages`, `canton_getPackageStatus` and the `packagesChanged` event;
  - the signed requests sent to the Sidecar.

  The specification is reviewed with the CIP-0103 and dApp SDK maintainers and submitted as a draft CIP.
- Validator Sidecar v1:
  - checks that each request is signed by a party the Validator hosts;
  - checks each DAR's hash against the app's manifest;
  - allowlist policy;
  - dry-run vetting before anything is applied;
  - a request log and an operator UI.

  It is shipped as a Docker image and Helm chart, with an operator runbook.
- Wallet library v1: manifest loading, the readiness check, approval screen data, session-approval hooks for the Wallet Gateway and WalletConnect, and the `canton_requestPackages` handler
- Curated app registry: listings are reviewed before they are added
- Integration in the Metatarz wallet
- Open-source release under Apache 2.0, with developer documentation

**Acceptance Criteria:**

- **Apps work without manual vetting.** Three allowlisted apps outside Metatarz are used from wallets whose Validators had not vetted their packages. In each case the user approves and completes a transaction, and no operator steps in by hand.
- **External operators.** At least two Validator operators outside Metatarz run the Sidecar with those apps on their allowlist.
- **Policy is enforced.** A request from an app that isn't on the allowlist is refused, and a DAR whose hash doesn't match its manifest is rejected.
- **Specification reviewed.** Written review comments from at least one CIP-0103 or dApp SDK maintainer and one wallet operator are resolved, or recorded in the changelog with a reason.

**Payment breakdown (1,300,000 CC):**

| Part | Amount | Paid when |
| :---- | :---- | :---- |
| Base delivery | 700,000 CC | The specification, Sidecar, wallet library and registry are released, and the flow runs on Metatarz's own Validator |
| External operator test | 150,000 CC per operator, 2 operators (300,000 CC) | The operator runs the Sidecar with an allowlist, and an allowlisted app's packages get vetted on their Validator through the flow |
| External app test | 100,000 CC per app, 3 apps (300,000 CC) | The app publishes a manifest, is allowlisted, and a user completes a transaction from a wallet whose Validator had not vetted its packages |


### Milestone 2: Permissionless Package Requests

| Field | Details |
| :---- | :---- |
| **Estimated Delivery** | 5 months after delivery of Milestone 1 |
| **Committed Date** | 2027-09-30 |
| **Focus** | Any app can get its DARs vetted for its users, without being allowlisted in advance. Operators set policies based on the packages themselves rather than on a list of apps. |
| **Team** | Metatarz, plus an independent security firm |

**Deliverables:**

- Permissionless manifests. The wallet reads the manifest from the app's domain, verified through WalletConnect's Verify API or the origin reported by the dApp SDK. Listing in the registry becomes optional, and the registry accepts listings from anyone, with CI checks.
- DAR safety checks in the Sidecar:
  - every package in a DAR is checked, not just the main one;
  - a DAR that reuses the name of an already-vetted package from a different publisher is rejected;
  - a higher version of a vetted package is never approved automatically;
  - the DAR is validated before upload;
  - limits on DAR size and on the number of packages per DAR;
  - rate limits per party.
- Operator policies based on the packages: automatic approval for packages with a published audit in the Daml Package Registry, or that pass the Daml Package Analyzer, with a manual queue for everything else. Audit and analyzer results are also shown on the user's approval screen.
- Support for parties hosted on more than one Validator. The request goes to every host, and failures on any host are reported.
- Fallback when `canton_prepareSignExecute` fails because a package isn't vetted
- External security audit of the Sidecar and the wallet library. It covers request authentication, the hash and safety checks, policy enforcement and manifest spoofing. Findings are fixed or recorded with a reason, and the summary is shared with the Foundation's Security group.
- Specification v1.0 and the CIP moved to its next stage, plus a pull request offering the wallet library to the Splice Wallet SDK

**Acceptance Criteria:**

- **Any app.** An app that is not on any allowlist and not listed in the registry gets its packages vetted through an operator's audit-based or analyzer-based policy. Its user then completes a transaction.
- **Upgrade guard.** A test DAR that reuses a vetted package's name with a higher version goes to the operator's queue and is not vetted automatically.
- **Multi-host consistency.** For a party hosted on two Validators, the packages end up vetted on both, or the wallet reports which host failed.
- **Audit complete.** The audit report is published, and every finding is fixed or recorded with a reason.

**Payment breakdown (1,600,000 CC):**

| Part | Amount | Paid when |
| :---- | :---- | :---- |
| Base delivery | 700,000 CC | Permissionless manifests, DAR safety checks, policies based on the packages, the fallback and specification v1.0 are released |
| Security audit | 300,000 CC | The audit report is published, and every finding is fixed or recorded with a reason |
| External operator test | 150,000 CC per operator, 2 operators (300,000 CC) | The operator runs an audit-based or analyzer-based policy with no app allowlist, and a package from an app it never allowlisted gets vetted through the flow. The upgrade guard is shown working on their Validator |
| External app test | 100,000 CC per app, 2 apps (200,000 CC) | An app not on any allowlist gets its packages vetted through an operator's policy, and its user completes a transaction |
| Multi-host test | 100,000 CC, 1 test | A party hosted on two Validators run by operators other than Metatarz gets the packages vetted on both through one request |

### Milestone 3: Ongoing Maintenance

| Field | Details |
| :---- | :---- |
| **Estimated Delivery** | Begins after delivery of Milestone 2 and covers 12 months |
| **Focus** | Keeping the Sidecar, wallet library and registry healthy as Canton, Splice, CIP-0103 and WalletConnect change |
| **Team** | Metatarz |

Maintenance covers:

- Security patches: critical issues fixed within 7 days of disclosure, high-severity issues within 30 days
- Dependency updates for new Canton, Splice, Daml SDK and Wallet SDK releases, including changes to the Ledger API's package endpoints
- Keeping up with changes to CIP-0103 and the WalletConnect Canton namespace
- Registry upkeep: new listings reviewed within 5 business days
- External pull requests get a first response within 5 business days
- No new features

**Acceptance Criteria.** Across the 12 months:
- CI stays green;
- there are no open critical vulnerabilities;
- the registry and pull-request response targets are met;
- the code stays compatible with the current Canton and Splice releases on MainNet.

This is reported quarterly.

### Milestone 4: Ecosystem Adoption

| Field | Details |
| :---- | :---- |
| **Estimated Delivery** | Within 6 months of delivery of Milestone 2 |
| **Focus** | Adoption on MainNet by teams other than Metatarz |
| **Team** | Metatarz (integration support, developer relations) |

**Deliverables:**

- Integration support and office hours for wallet, app and operator teams
- Integration guides for each of the three roles
- Eight adoptions on MainNet by teams other than Metatarz:
  - two **wallets** that support the flow for their users;
  - four **Validator operators** running the Sidecar, or the same checks built into their own backend, with automatic approval turned on;
  - two **apps** that use the flow and previously offered only in-wallet integrations.

**Acceptance Criteria.** Each adoption counts once it meets all of the following:

- It is run by an organization other than Metatarz.
- It is live on MainNet. Evidence is any of: the app's manifest or registry listing, the Validator's vetting state in public topology, or the wallet's public release notes.
- It is confirmed in writing by the adopting team.

**Payment breakdown (1,200,000 CC):**

| Adoption | Amount | Target | Total |
| :---- | :---- | :---- | :---- |
| Wallet live on MainNet | 200,000 CC each | 2 | 400,000 CC |
| Validator operator live on MainNet | 150,000 CC each | 4 | 600,000 CC |
| App live on MainNet | 100,000 CC each | 2 | 200,000 CC |


---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone
- Demonstrated functionality or operational readiness
- Documentation and knowledge transfer provided
- Alignment with stated value metrics

Project-specific conditions, in addition to each milestone's own acceptance criteria:

- **Any wallet, any app.** A user can connect a wallet from one organization to an app from another and transact, even though the wallet's Validator had not vetted the app's packages before. No manual step is needed beyond the operator's policy.
- **Operator stays in control.** The Sidecar only vets packages that match the app's manifest and pass the operator's policy. It never vets a newer version of an already-vetted package without operator approval.
- **No custody.** No component built under this grant handles user private keys.

---

## Funding

**Total Funding Request:** 4,500,000 CC

### Milestone Allocation

| Milestone | CC Amount | Payment Trigger |
| :---- | :---- | :---- |
| M1 - Package Requests for Allowlisted Apps | 1,300,000 CC | 700,000 CC base on delivery; 150,000 CC per external operator (2) and 100,000 CC per external app (3) as each completes its test |
| M2 - Permissionless Package Requests | 1,600,000 CC | 700,000 CC base on delivery; 300,000 CC on the published audit; 150,000 CC per external operator (2), 100,000 CC per external app (2) and 100,000 CC for the multi-host test, as each is completed |
| M3 - Ongoing Maintenance (12 months) | 400,000 CC | Paid against quarterly maintenance reports (100,000 CC per quarter) |
| M4 - Ecosystem Adoption | 1,200,000 CC | Paid against each verified adoption: 200,000 CC per wallet (2), 150,000 CC per operator (4), 100,000 CC per app (2) |
| **Total** | **4,500,000 CC** | |

In total, 2,400,000 CC (53%) is paid only when a team other than Metatarz completes a test or goes live: 1,200,000 CC in the tests in Milestones 1 and 2, and 1,200,000 CC in Milestone 4. The other 2,100,000 CC pays for Metatarz's own delivery (1,400,000 CC), the security audit (300,000 CC) and maintenance (400,000 CC).

Milestones 1 and 2 pay for three engineers and one developer relations lead for nine months, plus the audit.

### Timeline Incentives

**SLA penalty.** The payment for a milestone is reduced by 10% for every full month it is delivered after its committed date. If any milestone is more than three full months late for reasons within Metatarz's control, the terms will be revisited with the Tech & Ops Committee. Delays caused only by changes to CIP-0103 or to the Ledger API package endpoints, which are outside Metatarz's control, are excluded.

**Acceleration bonus.** If all eight adoptions in Milestone 4 are met more than one month early, the Milestone 4 payout increases by 10%.

### Volatility Stipulation

The grant is denominated in Canton Coin. The project runs for more than 6 months, so the grant will be re-evaluated at the 6-month mark. We ask that this single re-evaluation cover all remaining milestones, including maintenance and adoption.

---

## Co-Marketing

Upon release, Metatarz will collaborate with the Foundation on:

- Announcement coordination at the release of Milestone 1 (allowlisted apps) and Milestone 2 (any app)
- A technical blog post on how apps can work in any wallet without per-wallet integrations
- A short guide for operators on running the Sidecar and setting approval policies
- A workshop for app and wallet developers
- Adoption numbers in the quarterly Development Fund report

---

## Motivation

Canton has an open wallet standard in CIP-0103 and funded work around it, including the dApp SDK, Wallet Gateway and Wallet SDK. The connection itself works. What breaks is the first transaction, when the user's Validator hasn't vetted the app's packages. Today there is no way for a wallet or app to fix that in the moment. Someone has to contact the wallet operator and wait.

Apps have responded by building in-wallet apps for the wallets that already vetted their packages, and by recommending those over a normal wallet connection. This is a reasonable choice for each app, but it has a cost for the ecosystem:

- Users are pushed toward a few wallets.
- New wallets struggle to support existing apps.
- App teams maintain a separate integration for each wallet.

The roadmap goal that key holders have *"full control over which applications they participate in"* is hard to meet while installing an app means talking to a wallet operator.

This matters more for developers coming from EVM. The Foundation's [2026 Developer Experience and Tooling Survey](https://forum.canton.network/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412) found that 71% of developers come from an EVM background. It found that the move to Canton's model, including party-based identity, is one of the things that takes them longest to get right. On EVM, "connect wallet" is all an app needs. This proposal brings Canton closer to that experience while keeping the operator's control over what runs on their Validator.

Metatarz runs an EVM-compatible Canton wallet that helps EVM teams move to Canton. We've seen this problem in production. Apps we onboarded worked in our wallet, but users of other wallets hit vetting failures. The apps were then told to build integrations for specific wallets instead.
<!-- TODO(Metatarz): add concrete numbers, e.g. how many onboarded apps hit vetting failures from external wallets, or how many built in-wallet apps instead. -->

**Who benefits:**

- Every CIP-0103 app gets one integration that works across wallets.
- Every wallet can support listed apps without handling them one by one.
- Every Validator operator gets a policy-driven, auditable replacement for manual vetting.

**Sustainability.** Milestone 3 funds 12 months of maintenance after MainNet release. After that, Metatarz will keep maintaining the Sidecar, wallet library and registry for at least another 12 months, because our own wallet depends on them. The specification goes through the CIP process so that it does not depend on us. If the wallet library is accepted into the Splice Wallet SDK, its maintenance moves to the existing Wallet Kernel maintenance arrangement.

**Public good.** All code and specifications are released under Apache 2.0. Any wallet, app or operator can use them without using Metatarz services.

---

## Rationale

**Use the session that already exists.** Connecting is the moment the user decides to use an app. CIP-0103 and WalletConnect already show an approval screen at that point, so the package request belongs there instead of in a second connect step. The one new method follows a pattern EVM developers know from `wallet_watchAsset`, and its name says what it does. The fallback when a transaction fails means apps that change nothing still benefit.

**Reuse WalletConnect's domain check.** The manifest is only as trustworthy as the domain it comes from. WalletConnect's Verify API already checks that domain, so we use it instead of building our own.

**The operator decides, the user requests.** Vetting is a setting for the whole Validator. It affects every party on the node, so the decision has to stay with the operator. The Sidecar turns that decision into a policy set once, instead of a support ticket for every app.

**Build on what exists.** The Ledger API already supports package upload, vetting with a dry run, and preferred-package lookup. CIP-0103 already handles connections. Obsidian's registry and Certora's analyzer already answer whether a package can be trusted. This proposal adds the missing piece that connects those to the moment a user connects.

**Alternatives considered:**

| Approach | Why not |
| :---- | :---- |
| Keep vetting by hand | This is the current situation, and it is what pushes apps toward in-wallet integrations. |
| Wallet operators vet all known apps in advance | This doesn't scale with the number of apps, and new apps still wait. Operators also take on code they never reviewed. |
| Ask the app to submit its DARs to each wallet operator directly | This is the same as manual vetting with an API in front. The user and the connection are not part of it. |
| Vetting controlled by each party at the protocol level | This is the right long-term answer, but it needs protocol changes. It is outside a tooling grant, and we'll share our findings with the DAR Package Management SIG. |
