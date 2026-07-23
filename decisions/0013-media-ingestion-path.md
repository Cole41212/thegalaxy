# 0013 — Media ingestion path: two doors, one dataset
**Status:** Accepted (Phase 4, 2026-07-22)
**Context:** The media library must be writable from falcon (TRUSTED) for ingest and
readable by cantina (SERVERS) for serving. A single rw path serving both would hand the
internet-adjacent media server write access to the whole library — the exact host a
media-stack compromise lands on.
**Decision:** Two doors into one dataset. Write door: authenticated SMB from TRUSTED as
`coalish`, a dedicated non-admin TrueNAS user. Read door: NFS, mounted `ro,soft` on
cantina. Dataset POSIX ACL: named user `coalish` rwx plus the mandatory mask rwx;
`Other` stays r-x — that r-x IS the NFS read path.
**Alternatives:** One rw path consumed by both sides (rejected — grants cantina the write
access this design exists to withhold); ingest under NAS admin credentials (rejected —
ingest needs one dataset, not the NAS).
**Consequences:** A compromised cantina can stream the library but cannot modify or
encrypt it. Ingest credentials scope to a single dataset, not to NAS administration.
Verified end-to-end: a file written over SMB from falcon was read back over NFS on
cantina.
