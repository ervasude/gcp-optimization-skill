---
name: gcp-optimization-skills
description: Cost and performance optimization for Google Cloud Platform — BigQuery query/storage tuning, Cloud Storage lifecycle, compute right-sizing, and data pipeline optimization. Use when GCP resources are slow, expensive, or being newly designed. For complex queries (100+ lines, multi-JOIN, multi-CTE), use Section 8 (How to Review a Query) as a systematic walkthrough before applying individual optimizations. 
---

# GCP Optimization

This skill is a guide for cost and performance optimization on Google Cloud Platform. All information is based on official Google Cloud documentation.

## When will be used

- BigQuery queries are running slowly or generating high costs
- Cloud storage costs need to be optimized
- Compute resources (Cloud Run, GKE, Compute Engine) need to be right-sized
- Data pipelines (Dataflow, Composer) need to be tuned
- A new table/resource is being designed and the right decisions need to be made

## Content

Details are in the relevant files in the `references/` folder:

- **BigQuery optimization** → `references/bigquery.md`
- **Cloud Storage optimization** → `references/cloud-storage.md`
- **Compute optimization** → `references/compute.md` *(coming soon)*
- **Data pipelines optimization** → `references/data-pipelines.md` *(coming soon)*


## Reference

All content has been compiled from the following sources:
- https://cloud.google.com/bigquery/docs
- https://cloud.google.com/storage/docs
- https://cloud.google.com/architecture/framework/cost-optimization

## Testing

This skill has eval cases under `evals/evals.json`. See `evals/README.md` for how to run and interpret them.