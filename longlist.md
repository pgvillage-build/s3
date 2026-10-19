# MinIO replacement – longlist

_Compiled 2026-10-19._

## Context

MinIO is no longer a viable upstream for us:

- 2021: relicensed Apache-2.0 → AGPLv3. The license itself is acceptable to us.
- 2025: the admin console was removed from the community edition, and binary and container releases stopped.
- Dec 2025: the repo went into maintenance mode. In Feb 2026 it was marked "no longer maintained", and `minio/minio` was archived in Apr 2026. `minio/mc` followed in Jul 2026.

## Requirements (what we are selecting on)

1. **OSI-approved open-source license**. Apache-2.0, MIT, and AGPLv3 are all acceptable. The license was not the problem with MinIO. The problem was a single vendor withdrawing the open-source project in 2025.
2. **Healthy, independent community that is open to contributions**. Several active maintainers, not one person or one vendor that can pull the plug. Outside PRs are accepted, and the governance and roadmap are public.
3. **Simple**: preferably a single binary with few runtime dependencies and no external DB.
4. **We can build it ourselves and package it as RPM and DEB**. Go and Rust static binaries are easiest. JVM and Node.js need more work.
5. **Expose a local filesystem as an S3 endpoint**. Ideally objects are stored as **plain files** that stay readable without the tool ("passthrough"), not as an opaque blob/erasure format.
6. Reasonable S3 API coverage: multipart upload, ListObjectsV2, SigV4, presigned URLs, ideally versioning, object lock, and multiple users/keys. pgBackRest, WAL-G, and Barman need multipart and SigV4.
7. **Written in Go or Rust (strong preference)**. This matches our in-house skills, so we can patch, build, and audit the code ourselves. It also gives a single static binary without a runtime (no JVM, Node.js, or Python), which keeps rpm/deb packaging simple.

## Longlist

| # | Project | GitHub | ★ Stars | 80% of commits by | Language | License | Local FS model | Packaging effort | Maturity / status | Notes |
|---|---------|--------|-------|------|----------|---------|----------------|------------------|-------------------|-------|
| 1 | **RustFS** | https://github.com/rustfs/rustfs | 35.1k | 6 of 194 (3.1%) | Rust | Apache-2.0 | Opaque (MinIO-style `xl.meta`, erasure coding) | Low–medium: single binary, cargo build; upstream has packages | 1.0 GA in Sep 2026; young. Distributed mode is less proven | Most direct "drop-in MinIO" replacement, with a console, IAM, versioning, and object lock. Data on disk is not plain files. |
| 2 | **Garage** (Deuxfleurs) | https://github.com/deuxfleurs-org/garage (mirror; primary: https://git.deuxfleurs.fr/Deuxfleurs/garage) | 4.7k ² | 6 of 156 (3.8%) | Rust | AGPLv3 | Opaque (content-addressed blocks + LMDB/SQLite metadata) | Low: single static binary | Mature for its niche (geo-distributed small clusters) | Light and designed for self-hosting. Narrower S3 API (limited ACL/IAM; versioning/object lock missing or limited). |
| 3 | **rclone `serve s3`** | https://github.com/rclone/rclone | 60.4k ¹ | 24 of 1,256 (1.9%) | Go | MIT | **Plain files** (any rclone remote, incl. local dir) | Low: single binary; upstream ships `.rpm`/`.deb`; already in most distros | rclone is very mature; `serve s3` (since v1.65) is younger and labelled experimental-ish | Very simple. Uses gofakes3 underneath. Limited auth (one or a few static keys), and no versioning or object lock. Multipart is buffered. Fine for dev/test and light use. |
| 4 | **SeaweedFS** | https://github.com/seaweedfs/seaweedfs | 35.6k | 3 of 634 (0.5%) | Go | Apache-2.0 | Opaque (volume files); filer metadata in embedded leveldb or an external DB | Low: single binary (`weed`) | Mature, widely used | Can run all-in-one (`weed server -s3`). Scales to billions of small files. More moving parts (master, volume, filer) than needed for "simple". Some features are enterprise-only. |
| 5 | **JuiceFS S3 gateway** | https://github.com/juicedata/juicefs | 14.6k | 13 of 200 (6.5%) | Go | Apache-2.0 core; gateway is based on a MinIO fork (AGPLv3) | Opaque (chunks in an object store + metadata DB) | Medium | Mature | Wrong direction: it puts S3 in front of JuiceFS, which itself needs an object store. Not a fit. |
| 6 | **CubeFS** (CNCF) | https://github.com/cubefs/cubefs | 5.7k | 25 of 179 (14.0%) | Go | Apache-2.0 | Opaque | High (multi-component) | CNCF graduated | Distributed FS with an S3 gateway (ObjectNode). Overkill. |
| 7 | **Silo** (pgsty, formerly `pgsty/minio`; community MinIO fork) | https://github.com/pgsty/silo | 4.2k | 12 of 550 (2.2%) ³ | Go | AGPLv3 | Opaque (`xl.meta`); single-drive mode | Low: same as MinIO; fork ships binaries/rpm/deb | Maintained by one person (Pigsty), so the bus factor is low. Scope is security fixes only | Least migration effort, since it is functionally MinIO with the console restored. Long-term risk: one maintainer, no new features. |
| 8 | **Versity S3 Gateway (versitygw)** | https://github.com/versity/versitygw | 3.0k | 3 of 83 (3.6%) | Go | Apache-2.0 | **Plain files** (POSIX backend; metadata in xattrs) | Low: single static binary; upstream already ships `.rpm`/`.deb` | Active, commercially backed (Versity) | Built for exactly this: an S3 front-end for an existing POSIX FS. Also has ScoutFS, Azure and S3-proxy backends. Supports multipart, versioning, object lock, and an IAM-lite with multiple accounts. Needs xattr support on the FS. **Strong candidate.** |
| 9 | **gofakes3** | https://github.com/johannesboyne/gofakes3 | 0.9k | 5 of 52 (9.6%) | Go | MIT | In-memory / bolt / plain-FS backends | Low (library; tiny wrapper needed) | Test-grade | The library behind rclone `serve s3`. Only for testing. |
| 10 | **LocalStack** | https://github.com/localstack/localstack | 65.1k | 22 of 655 (3.4%) | Python | Community edition being folded into an account-gated product | In-memory / persisted | Medium | **Repo archived** (last push Mar 2026) | CI only. Not recommended. |
| 11 | **Ceph RGW** | https://github.com/ceph/ceph | 17.1k | 68 of 2,302 (3.0%) | C++ | LGPL-2.1 | Opaque (RADOS); no plain-FS mode | Distro packages exist; building yourself is heavy | Very mature | Gold standard for S3 compatibility, but massive overkill for "expose a local FS". Listed for completeness. |
| 12 | **Moto (server mode)** | https://github.com/getmoto/moto | 8.7k | 182 of 1,455 (12.5%) | Python | Apache-2.0 | In-memory | Medium | Test-grade | AWS mock for CI only. |
| 13 | **OpenStack Swift + s3api** | https://github.com/openstack/swift (mirror; primary: https://opendev.org/openstack/swift) | 2.8k | 32 of 470 (6.8%) | Python | Apache-2.0 | Opaque | High | Mature, declining | Overkill. |
| 14 | **S3Proxy** (gaul/s3proxy) | https://github.com/gaul/s3proxy | 2.4k | 1 of 83 (1.2%) | Java | Apache-2.0 | **Plain files** (jclouds `filesystem` / `filesystem-nio2` backend), or other blob stores | Medium: needs a JRE, so package a fat jar plus a systemd unit | Mature, small project | Simple. Also useful as a translation proxy (S3 → Azure/GCS/etc.). Single identity. Metadata stored in xattrs. |
| 15 | **Zenko CloudServer** (Scality) | https://github.com/scality/cloudserver | 2.0k | 21 of 97 (21.6%) | Node.js | Apache-2.0 | Opaque (`file` backend with its own data/metadata layout) | High: Node.js app with native deps | Maintained by Scality, but focus is their commercial product | Broad S3 API coverage. Heavy-ish for "simple". |
| 16 | **Apache Ozone** | https://github.com/apache/ozone | 1.3k | 62 of 342 (18.1%) | Java | Apache-2.0 | Opaque | High (JVM, multiple services) | Mature (Hadoop ecosystem) | Overkill unless you run Hadoop. |
| 17 | **s3gw** (SUSE/Aquarist, RGW + SFS) | https://github.com/s3gw-tech/s3gw | 0.2k | 6 of 20 (30.0%) | C++ | Apache-2.0 / LGPL | SQLite + files | – | **Discontinued** | Listed only to rule it out. |

¹ Stars are for rclone as a whole, not for `serve s3` specifically.
² Stars on the GitHub mirror only. Garage is developed on git.deuxfleurs.fr, so this undercounts its popularity.

³ Counts the full upstream MinIO history. Since the fork (Dec 2025), one person (Vonng) wrote 486 of 504 commits (96%).

Star counts were fetched from the GitHub API on 2026-10-19. Stars measure interest, not health. Treat them as a rough popularity signal next to the community column. For comparison, the archived `minio/minio` repo has 61.3k stars.

### Legend

- **Plain files**: an object `bucket/key` lands as a normal file at `<root>/bucket/key`. Existing data can be served, and the data stays readable without the tool.
- **Opaque**: the tool owns the on-disk format. You can only get data out via S3 or the tool.

## First-cut assessment

Against the requirements (license, community, simple, self-packaged rpm/deb, local FS as S3, Go/Rust):

| Project | GitHub | ★ Stars | 80% of commits by | Go/Rust | License | Community / governance | Simple | Easy rpm/deb | Plain FS | S3 coverage |
|---|---|---|---|---|---|---|---|---|---|---|
| RustFS | https://github.com/rustfs/rustfs | 35.1k | 6 of 194 (3.1%) | ✅ Rust | ✅ Apache-2.0 | ⚠ Vendor-led (RustFS company), young, fast-growing | ✅ | ✅ | ❌ | ✅ good |
| Garage | https://github.com/deuxfleurs-org/garage | 4.7k ² | 6 of 156 (3.8%) | ✅ Rust | ✅ AGPLv3 | ✅ Non-profit collective (Deuxfleurs), publicly funded (NLnet/NGI). Contributions via their Forgejo, not GitHub | ✅ | ✅ | ❌ | ⚠ limited |
| rclone serve s3 | https://github.com/rclone/rclone | 60.4k ¹ | 24 of 1,256 (1.9%) | ✅ Go | ✅ MIT | ✅ Large, long-lived community with many contributors | ✅ | ✅ | ✅ | ⚠ basic |
| SeaweedFS | https://github.com/seaweedfs/seaweedfs | 35.6k | 3 of 634 (0.5%) | ✅ Go | ✅ Apache-2.0 | ⚠ One lead maintainer, plus a commercial enterprise edition | ⚠ | ✅ | ❌ | ✅ good |
| Silo (pgsty/minio) | https://github.com/pgsty/silo | 4.2k | 12 of 550 (2.2%) ³ | ✅ Go | ✅ AGPLv3 | ❌ Single maintainer, scope is security fixes only | ✅ | ✅ | ❌ | ✅ good |
| versitygw | https://github.com/versity/versitygw | 3.0k | 3 of 83 (3.6%) | ✅ Go | ✅ Apache-2.0 | ⚠ Vendor-led (Versity), accepts outside PRs | ✅ | ✅ | ✅ | ✅ good |
| S3Proxy | https://github.com/gaul/s3proxy | 2.4k | 1 of 83 (1.2%) | ❌ Java | ✅ Apache-2.0 | ⚠ Mostly a single maintainer | ✅ | ⚠ JVM | ✅ | ⚠ basic |

The Go/Rust preference rules out S3Proxy (Java), Zenko CloudServer (Node.js), Swift and Moto (Python), and Ozone (Java). It leaves Ceph RGW (C++) further behind.

**On community risk.** MinIO showed that the real risk is one company controlling the project, not the license. Vendor-led projects (versitygw, RustFS) carry the same kind of risk. Because the code is Apache-2.0, a community fork stays possible, but someone would have to maintain it. The community column is a first impression. Before deciding, check: the number of active maintainers, the share of commits from outside the vendor, whether a CLA allows relicensing, and how fast outside PRs get reviewed.

**Proposed shortlist:**

1. **versitygw**: best match for "local FS as S3", and upstream already does rpm/deb. The main risk is vendor control.
2. **rclone serve s3**: strongest community, and plain files. S3 features are basic, so it suits lighter use or dev/test.
3. **Garage** or **RustFS**: if we accept an opaque on-disk format and want a full object store. Garage has the healthier, independent governance but a narrower S3 API. RustFS is closer to MinIO in features but is vendor-led and young.

Next step: validate the shortlist against our actual clients (pgBackRest / WAL-G / Barman S3 repos: multipart, SigV4, path-style vs vhost, TLS) and do a trial build and package for EL9 and Debian/Ubuntu.

## Contributor concentration

How dependent each project is on a few people. "80% of commits by" is the smallest number of contributors who together made at least 80% of all commits, with that number as a percentage of all contributors. A low number means a high bus-factor risk.

Method: GitHub contributors API (`/contributors?anon=1`), fetched 2026-10-19. It counts **commits, not lines of code**, over the **full history** of the default branch. Bots are excluded (GitHub `Bot` accounts, `*[bot]`, Scality's `bert-e` merge bot, OpenStack's `Jenkins`/`Zuul`). Contributors are counted per GitHub account or, if unlinked, per author name or email, so one person with several identities is counted more than once.

| Project | Contributors | Commits | 80% of commits by | #1 contributor | #2 contributor | #3 contributor |
|---|---|---|---|---|---|---|
| RustFS | 194 | 7,124 | 6 (3.1%) | overtrue: 2,613 (36.7%) | houseme: 1,553 (21.8%) | weisd: 715 (10.0%) |
| Garage | 156 | 2,930 | 6 (3.8%) | Alexis211: 1,493 (51.0%) | Alex Auvolat: 306 (10.4%) ⁴ | superboum: 223 (7.6%) |
| rclone | 1,256 | 10,521 | 24 (1.9%) | ncw: 6,772 (64.4%) | albertony: 569 (5.4%) | nielash: 200 (1.9%) |
| SeaweedFS | 634 | 13,849 | 3 (0.5%) | chrislusf: 10,614 (76.6%) | kmlebedev: 391 (2.8%) | Konstantin Lebedev: 141 (1.0%) ⁴ |
| JuiceFS | 200 | 4,604 | 13 (6.5%) | davies: 787 (17.1%) | zhijian-pro: 772 (16.8%) | SandyXSD: 513 (11.1%) |
| CubeFS | 179 | 6,196 | 25 (14.0%) | leonrayang: 571 (9.2%) | awzhgw: 539 (8.7%) | sejust: 406 (6.6%) |
| Silo (pgsty/minio) ³ | 550 | 13,047 | 12 (2.2%) | harshavardhana: 5,678 (43.5%) | vadmeste: 901 (6.9%) | klauspost: 628 (4.8%) |
| versitygw | 83 | 2,800 | 3 (3.6%) | benmcclelland: 1,801 (64.3%) | niksis02: 343 (12.2%) | mlt180: 284 (10.1%) |
| gofakes3 | 52 | 384 | 5 (9.6%) | johannesboyne: 153 (39.8%) | shabbyrobe: 140 (36.5%) | ChrisSchinnerl: 7 (1.8%) |
| LocalStack | 655 | 7,699 | 22 (3.4%) | whummer: 1,582 (20.5%) | bentsku: 601 (7.8%) | thrau: 492 (6.4%) |
| Ceph | 2,302 | 166,443 | 68 (3.0%) | Sage Weil: 35,285 (21.2%) | tchaikov: 12,388 (7.4%) | yehudasa: 5,440 (3.3%) |
| Moto | 1,455 | 10,070 | 182 (12.5%) | bblommers: 2,088 (20.7%) | spulec: 1,584 (15.7%) | bpandola: 623 (6.2%) |
| OpenStack Swift | 470 | 7,214 | 32 (6.8%) | tipabu: 1,258 (17.4%) | alistairncoles: 635 (8.8%) | clayg: 559 (7.7%) |
| S3Proxy | 83 | 1,783 | 1 (1.2%) | gaul: 1,565 (87.8%) | kahing: 32 (1.8%) | klaudworks: 26 (1.5%) |
| Zenko CloudServer | 97 | 6,170 | 21 (21.6%) | LaurenSpiegel: 865 (14.0%) | rahulreddy: 471 (7.6%) | BourgoisMickael: 382 (6.2%) |
| Apache Ozone | 342 | 11,169 | 62 (18.1%) | adoroszlai: 1,823 (16.3%) | elek: 462 (4.1%) | bharatviswa504: 398 (3.6%) |
| s3gw | 20 | 818 | 6 (30.0%) | m-ildefons: 185 (22.6%) | jecluis: 145 (17.7%) | Giuseppe Baccini: 115 (14.1%) |

³ Full upstream MinIO history; the top 3 are former MinIO Inc. developers. Since the fork (Dec 2025), Vonng wrote 486 of 504 commits (96%).
⁴ Probably the same person under two identities (a GitHub account plus an unlinked author name). Merged, Garage's #1 is about 61% and SeaweedFS's #2 is about 3.8%.

**Takeaways for the shortlist:**

- **versitygw**: 3 people made 80% of the commits, and one (benmcclelland) made 64%. If they are all Versity staff (to verify), that reinforces the vendor-control risk.
- **rclone**: the lead (ncw) made 64% of commits, but there is a long tail. 24 people make up 80%, and there are over 1,200 contributors.
- **RustFS**: more evenly spread over a core team (6 people for 80%), but young and vendor-led.
- **Garage**: one lead developer made about 61%, with 6 people for 80%.
- **SeaweedFS** and **S3Proxy** effectively depend on one person (77% and 88%).

## Sources

- [MinIO alternatives (vonng.com)](https://blog.vonng.com/en/db/minio-alternative/)
- [MinIO Is Dead, Long Live MinIO (pgsty/minio fork)](https://vonng.com/en/db/minio-resurrect/)
- [MinIO archived: Garage, SeaweedFS, Ceph RGW (youngju.dev)](https://www.youngju.dev/transcribe/2026-07-17-minio-archived-garage-seaweedfs-ceph-rgw.en)
- [MinIO moves away from open source (It's FOSS)](https://itsfoss.com/news/minio-moves-away-from-open-source/)
- [Self-hosted S3 alternatives in 2026 (glukhov.org)](https://www.glukhov.org/pl/data-infrastructure/object-storage/self-hosted-s3-alternatives/)
- [RustFS 1.0 GA (Linuxiac)](https://linuxiac.com/rustfs-1-0-s3-compatible-object-storage-reaches-ga/)
- [What is RustFS (Sealos)](https://sealos.io/blog/what-is-rustfs/)
- [RustFS docs](https://docs.rustfs.com/concepts/introduction.html)
