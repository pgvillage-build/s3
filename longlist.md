# MinIO replacement – longlist

_Compiled 2026-10-19._

## Context

MinIO is no longer a viable upstream for us:

- 2021: relicensed Apache-2.0 → AGPLv3.
- 2025: the admin console was removed from the community edition, and binary and container releases stopped.
- Dec 2025: the repo went into maintenance mode. In Feb 2026 it was marked "no longer maintained", and `minio/minio` was archived in Apr 2026. `minio/mc` followed in Jul 2026.

## Requirements (what we are selecting on)

1. **Permissive / unproblematic open-source license**. Prefer Apache-2.0 / MIT / BSD; AGPL is a flag.
2. **Simple**: preferably a single binary with few runtime dependencies and no external DB.
3. **We can build it ourselves and package it as RPM and DEB**. Go and Rust static binaries are easiest. JVM and Node.js need more work.
4. **Expose a local filesystem as an S3 endpoint**. Ideally objects are stored as **plain files** that stay readable without the tool ("passthrough"), not as an opaque blob/erasure format.
5. Reasonable S3 API coverage: multipart upload, ListObjectsV2, SigV4, presigned URLs, ideally versioning, object lock, and multiple users/keys. pgBackRest, WAL-G, and Barman need multipart and SigV4.
6. **Written in Go or Rust (strong preference)**. This matches our in-house skills, so we can patch, build, and audit the code ourselves. It also gives a single static binary without a runtime (no JVM, Node.js, or Python), which keeps rpm/deb packaging simple.

## Longlist

| # | Project | GitHub | Language | License | Local FS model | Packaging effort | Maturity / status | Notes |
|---|---------|--------|----------|---------|----------------|------------------|-------------------|-------|
| 1 | **Versity S3 Gateway (versitygw)** | https://github.com/versity/versitygw | Go | Apache-2.0 | **Plain files** (POSIX backend; metadata in xattrs) | Low: single static binary; upstream already ships `.rpm`/`.deb` | Active, commercially backed (Versity) | Built for exactly this: an S3 front-end for an existing POSIX FS. Also has ScoutFS, Azure and S3-proxy backends. Supports multipart, versioning, object lock, and an IAM-lite with multiple accounts. Needs xattr support on the FS. **Strong candidate.** |
| 2 | **rclone `serve s3`** | https://github.com/rclone/rclone | Go | MIT | **Plain files** (any rclone remote, incl. local dir) | Low: single binary; upstream ships `.rpm`/`.deb`; already in most distros | rclone is very mature; `serve s3` (since v1.65) is younger and labelled experimental-ish | Very simple. Uses gofakes3 underneath. Limited auth (one or a few static keys), and no versioning or object lock. Multipart is buffered. Fine for dev/test and light use. |
| 3 | **RustFS** | https://github.com/rustfs/rustfs | Rust | Apache-2.0 | Opaque (MinIO-style `xl.meta`, erasure coding) | Low–medium: single binary, cargo build; upstream has packages | 1.0 GA in Sep 2026; young. Distributed mode is less proven | Most direct "drop-in MinIO" replacement, with a console, IAM, versioning, and object lock. Data on disk is not plain files. |
| 4 | **SeaweedFS** | https://github.com/seaweedfs/seaweedfs | Go | Apache-2.0 | Opaque (volume files); filer metadata in embedded leveldb or an external DB | Low: single binary (`weed`) | Mature, widely used | Can run all-in-one (`weed server -s3`). Scales to billions of small files. More moving parts (master, volume, filer) than needed for "simple". Some features are enterprise-only. |
| 5 | **Garage** (Deuxfleurs) | https://github.com/deuxfleurs-org/garage (mirror; primary: https://git.deuxfleurs.fr/Deuxfleurs/garage) | Rust | **AGPLv3** ⚠ | Opaque (content-addressed blocks + LMDB/SQLite metadata) | Low: single static binary | Mature for its niche (geo-distributed small clusters) | Light and designed for self-hosting. Narrower S3 API (limited ACL/IAM; versioning/object lock missing or limited). AGPL has the same license concern as MinIO. |
| 6 | **pgsty/minio** (community MinIO fork) | https://github.com/pgsty/minio | Go | **AGPLv3** ⚠ | Opaque (`xl.meta`); single-drive mode | Low: same as MinIO; fork ships binaries/rpm/deb | Maintained by one person (Pigsty), so the bus factor is low. Scope is security fixes only | Least migration effort, since it is functionally MinIO with the console restored. Does not fix the license concern. Long-term risk. |
| 7 | **S3Proxy** (gaul/s3proxy) | https://github.com/gaul/s3proxy | Java | Apache-2.0 | **Plain files** (jclouds `filesystem` / `filesystem-nio2` backend), or other blob stores | Medium: needs a JRE, so package a fat jar plus a systemd unit | Mature, small project | Simple. Also useful as a translation proxy (S3 → Azure/GCS/etc.). Single identity. Metadata stored in xattrs. |
| 8 | **Zenko CloudServer** (Scality) | https://github.com/scality/cloudserver | Node.js | Apache-2.0 | Opaque (`file` backend with its own data/metadata layout) | High: Node.js app with native deps | Maintained by Scality, but focus is their commercial product | Broad S3 API coverage. Heavy-ish for "simple". |
| 9 | **Ceph RGW** | https://github.com/ceph/ceph | C++ | LGPL-2.1 | Opaque (RADOS); no plain-FS mode | Distro packages exist; building yourself is heavy | Very mature | Gold standard for S3 compatibility, but massive overkill for "expose a local FS". Listed for completeness. |
| 10 | **OpenStack Swift + s3api** | https://github.com/openstack/swift (mirror; primary: https://opendev.org/openstack/swift) | Python | Apache-2.0 | Opaque | High | Mature, declining | Overkill. |
| 11 | **Apache Ozone** | https://github.com/apache/ozone | Java | Apache-2.0 | Opaque | High (JVM, multiple services) | Mature (Hadoop ecosystem) | Overkill unless you run Hadoop. |
| 12 | **CubeFS** (CNCF) | https://github.com/cubefs/cubefs | Go | Apache-2.0 | Opaque | High (multi-component) | CNCF graduated | Distributed FS with an S3 gateway (ObjectNode). Overkill. |
| 13 | **JuiceFS S3 gateway** | https://github.com/juicedata/juicefs | Go | Apache-2.0 core; gateway is based on a MinIO fork (**AGPLv3** ⚠) | Opaque (chunks in an object store + metadata DB) | Medium | Mature | Wrong direction: it puts S3 in front of JuiceFS, which itself needs an object store. Not a fit. |
| 14 | **gofakes3** | https://github.com/johannesboyne/gofakes3 | Go | MIT | In-memory / bolt / plain-FS backends | Low (library; tiny wrapper needed) | Test-grade | The library behind rclone `serve s3`. Only for testing. |
| 15 | **Moto (server mode)** | https://github.com/getmoto/moto | Python | Apache-2.0 | In-memory | Medium | Test-grade | AWS mock for CI only. |
| 16 | **LocalStack** | https://github.com/localstack/localstack | Python | Community edition being folded into an account-gated product | In-memory / persisted | Medium | Test-grade | CI only. License and terms are drifting. Not recommended. |
| 17 | **s3gw** (SUSE/Aquarist, RGW + SFS) | https://github.com/aquarist-labs/s3gw | C++ | Apache-2.0 / LGPL | SQLite + files | – | **Discontinued** | Listed only to rule it out. |

### Legend

- **Plain files**: an object `bucket/key` lands as a normal file at `<root>/bucket/key`. Existing data can be served, and the data stays readable without the tool.
- **Opaque**: the tool owns the on-disk format. You can only get data out via S3 or the tool.

## First-cut assessment

Against the requirements (simple, permissive license, self-packaged rpm/deb, local FS as S3, Go/Rust):

| Project | GitHub | Go/Rust | License | Simple | Easy rpm/deb | Plain FS | S3 coverage |
|---|---|---|---|---|---|---|---|
| versitygw | https://github.com/versity/versitygw | ✅ Go | ✅ Apache-2.0 | ✅ | ✅ | ✅ | ✅ good |
| rclone serve s3 | https://github.com/rclone/rclone | ✅ Go | ✅ MIT | ✅ | ✅ | ✅ | ⚠ basic |
| S3Proxy | https://github.com/gaul/s3proxy | ❌ Java | ✅ Apache-2.0 | ✅ | ⚠ JVM | ✅ | ⚠ basic |
| RustFS | https://github.com/rustfs/rustfs | ✅ Rust | ✅ Apache-2.0 | ✅ | ✅ | ❌ | ✅ good |
| SeaweedFS | https://github.com/seaweedfs/seaweedfs | ✅ Go | ✅ Apache-2.0 | ⚠ | ✅ | ❌ | ✅ good |
| Garage | https://github.com/deuxfleurs-org/garage | ✅ Rust | ❌ AGPLv3 | ✅ | ✅ | ❌ | ⚠ limited |
| pgsty/minio | https://github.com/pgsty/minio | ✅ Go | ❌ AGPLv3 | ✅ | ✅ | ❌ | ✅ good |

The Go/Rust preference rules out S3Proxy (Java), Zenko CloudServer (Node.js), Swift and Moto (Python), and Ozone (Java). It leaves Ceph RGW (C++) further behind. All three shortlisted options are Go or Rust.

**Proposed shortlist:**

1. **versitygw**: best match for "local FS as S3" with a permissive license and an upstream that already does rpm/deb.
2. **RustFS**: if we want a MinIO-like product (console, IAM) and can accept an opaque on-disk format.
3. **rclone serve s3**: lightweight fallback, or for dev/test environments.

Next step: validate the shortlist against our actual clients (pgBackRest / WAL-G / Barman S3 repos: multipart, SigV4, path-style vs vhost, TLS) and do a trial build and package for EL9 and Debian/Ubuntu.

## Sources

- [MinIO alternatives (vonng.com)](https://blog.vonng.com/en/db/minio-alternative/)
- [MinIO Is Dead, Long Live MinIO (pgsty/minio fork)](https://vonng.com/en/db/minio-resurrect/)
- [MinIO archived: Garage, SeaweedFS, Ceph RGW (youngju.dev)](https://www.youngju.dev/transcribe/2026-07-17-minio-archived-garage-seaweedfs-ceph-rgw.en)
- [MinIO moves away from open source (It's FOSS)](https://itsfoss.com/news/minio-moves-away-from-open-source/)
- [Self-hosted S3 alternatives in 2026 (glukhov.org)](https://www.glukhov.org/pl/data-infrastructure/object-storage/self-hosted-s3-alternatives/)
- [RustFS 1.0 GA (Linuxiac)](https://linuxiac.com/rustfs-1-0-s3-compatible-object-storage-reaches-ga/)
- [What is RustFS (Sealos)](https://sealos.io/blog/what-is-rustfs/)
- [RustFS docs](https://docs.rustfs.com/concepts/introduction.html)
