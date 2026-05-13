# Cloud Storage Optimization

> Practical guide for cost optimization on Google Cloud Storage. All content is based on Google Cloud's official documentation.

## When to use this file

- A Cloud Storage bucket is generating high monthly cost
- Old data (logs, backups, archives) is sitting in expensive storage
- A new bucket is being created and the right storage class needs to be chosen
- Object versioning is producing unexpected cost growth
- A multi-region vs single-region decision needs to be made

---

## 1. Storage Classes

### What is a storage class?

A storage class is a setting on each object that determines its price-per-GB and how it's billed when accessed. 

### The four classes

| Class | Min storage duration | Best for | Relative cost |
|---|---|---|---|
| **Standard** | None | Hot data, frequently accessed | Highest |
| **Nearline** | 30 days | Accessed ~once per month (backups) | Lower |
| **Coldline** | 90 days | Accessed a few times per year | Even lower |
| **Archive** | 365 days | Long-term archive, accessed yearly or less | Lowest |

### Why does it matter?

Choosing the wrong class is one of the most common sources of wasted GCS spend — for example, keeping years-old backups or audit logs in Standard when they're read once a quarter at most. 

**Reference:** https://cloud.google.com/storage/docs/storage-classes

---

## 2. Lifecycle Management

### What is a lifecycle rule?

A lifecycle rule is a policy attached to a bucket that automatically moves objects to a cheaper class — or deletes them — once they meet a condition like age, version count, or storage class. You set it up once, and GCS applies it forever in the background.

### Common lifecycle patterns

- **Logs:** Standard → Nearline at 30 days → Coldline at 90 days → delete at 365 days. Logs are read often when fresh, rarely after a month, almost never after a quarter.

- **Backups:** Standard → Coldline at 30 days → Archive at 180 days. Backups are usually only restored in emergencies, so they can move to cold storage quickly.

- **Old object versions:** delete noncurrent versions after 30 days. Prevents versioning from quietly inflating cost.

### Example: typical log retention

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
```

✅ Logs auto-transition to cheaper classes and are deleted after a year.

**Reference:** https://cloud.google.com/storage/docs/lifecycle

---

## 3. Object Versioning

### What is object versioning?

When versioning is enabled on a bucket, GCS keeps a copy of every object that gets overwritten or deleted instead of removing it. The current version is "live" and the older copies are called noncurrent versions.

### Hidden cost trap

Every noncurrent version is billed exactly like a regular object — same per-GB rate, same storage class. On a bucket with frequent overwrites (e.g. a daily-replaced data file), versioning can quietly grow storage cost 5-10x without anyone noticing, because the live bucket size doesn't change.

### Mitigation

Pair versioning with a lifecycle rule that limits how many old versions to keep (`numNewerVersions`) or how long to keep them (`daysSinceNoncurrentTime`). A common setup is "keep the last 3 versions, delete anything older than 30 days."

**Reference:** https://cloud.google.com/storage/docs/object-versioning

---

## 4. Location: Region vs Multi-region vs Dual-region

### Tradeoffs

- **Single region** (e.g. `europe-west1`): cheapest storage, lowest latency for compute in the same region, but no cross-region redundancy.
- **Dual-region** (e.g. `nam4`): replicated across two regions, higher availability, ~2x cost.
- **Multi-region** (e.g. `EU`, `US`): replicated across many regions, highest availability, also higher cost.

### Choosing a location

Always colocate the bucket with the compute that reads it most often — BigQuery, Dataflow, GKE, etc. Putting a bucket in the US multi-region while BigQuery runs in europe-west1 means every read crosses regions and pays egress fees, which can easily dwarf the storage cost itself.

**Reference:** https://cloud.google.com/storage/docs/locations

---

## 5. Other Quick Wins

- **Compress before upload:** gzip/snappy text-heavy files; storage cost scales with bytes stored.
- **Avoid tiny objects:** thousands of <1KB files have higher per-operation cost than fewer larger files.
- **Audit unused buckets:** old bucket cleanup is the #1 forgotten cost source. Use `gcloud storage buckets list` and check last modified dates.
- **Use Storage Insights / inventory reports** to identify cold data still in Standard class.

---

## Quick Checklist

- [ ] Are buckets using the right storage class for their access pattern?
- [ ] Are lifecycle rules configured to auto-transition old data?
- [ ] Is object versioning enabled? If yes, is there a cleanup rule?
- [ ] Are buckets colocated with the compute that reads them?
- [ ] Are there old/unused buckets that can be deleted?

**General reference:** https://cloud.google.com/storage/docs/best-practices