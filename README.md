# Configuring S3 Storage for Archive Fallback

This guide explains how to configure the archive platform so that documents written to the **fallback archive** are stored in an S3 bucket (AWS S3, MinIO, or any S3-compatible object store) instead of a local directory.

## How it fits together

Two services are involved, and each is configured separately:

| Service | Default port | Role | What you configure |
|---|---|---|---|
| **Archive service** (`dmss-archive-services`) | 8090 | Routes documents to the configured archives | The connection entry that points at the fallback service, and how documents are routed to it |
| **Fallback service** (`dmss-archive-services-fallback`) | 8095 | Receives documents from the archive service and stores them | The S3 credentials, bucket, and endpoint |

Despite its name, the fallback service is **not limited to failure scenarios**. A document can reach it in three ways:

```
                        Archive service (routing)
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
  selected directly          primary archive            pipeline mode
  as its archive             write fails                stores here first,
  (by category or            -> failover                 syncs to the target
   priority)                                             archive later
        |                          |                          |
        +--------------------------+--------------------------+
                                   v
                          Fallback service
                                   |
                                   v
                              S3 bucket
```

In many deployments the fallback service is simply the primary archive — it is the highest-priority connection, and no `failover` is configured at all.

**Important:** S3 is configured **only in the fallback service**. The archive service does not need to know that storage is S3-backed — it always talks to the fallback service over HTTP, and the fallback service decides where the bytes go. No archive-service change is required to switch it from local disk to S3, regardless of which of the three routes above is in use.

---

## What S3 mode does and does not do

S3 mode changes **where the service stores the documents it manages**. It does not turn a bucket into a drop folder that the service monitors. Read this section before planning an integration.

### The service owns the bucket layout

Objects are written under keys derived from the document's identifier:

```
<document-id>/<version>/<filename>
```

`<document-id>` is assigned by the service when the document is created, `<version>` is a counter starting at 1, and `<filename>` comes from the supplied object name. The prefix is not configurable. Each version also stores a small companion `request` object next to the document:

```
6f1c2d3e-6b7a-4f0e-9c31-2a5d8e4b71f0/1/contract.pdf
6f1c2d3e-6b7a-4f0e-9c31-2a5d8e4b71f0/1/request
6f1c2d3e-6b7a-4f0e-9c31-2a5d8e4b71f0/2/contract.pdf     <- signed version
6f1c2d3e-6b7a-4f0e-9c31-2a5d8e4b71f0/2/request
```

Do not place unrelated objects inside these folders: retrieval expects exactly the document and its `request` companion per version.

### Documents must be submitted through the API

The service does **not** scan or watch the bucket. An object written into the bucket by another system is invisible to it — there is no import-by-key endpoint and no bucket watcher. A document enters only by being uploaded to the archive service, which returns a document identifier.

### New versions are written to new keys

Storing a new version — a signed document, for example — writes to the next version key (`<document-id>/2/…`). The original object is never overwritten.

### Integrating an external application

A common requirement is that another application produces documents — contracts to be signed, for example — which DMSS must then handle.

**The bucket cannot be used as an exchange folder.** A document written into the bucket by that application is never seen:

```
   External application  ---- writes contract ---->  S3 bucket
                                                          |
                                                          X   not visible to DMSS:
                                                          |   the bucket is not scanned
                                                        DMSS
```

**Documents are exchanged through the archive service API instead.** The identifier it returns is what ties the two systems together:

```
   1. upload          External application
                                |
                                v
                        Archive service  ---- stores ---->  S3 bucket
                                |                           <document-id>/1/contract.pdf
   2. document id  <------------+


   3. sign            User signs, by document id
                                |
                                v
                        Archive service  ---- stores ---->  S3 bucket
                                                            <document-id>/2/contract.pdf


   4. download        External application fetches the signed file by document id
                      and writes it to its own storage if a copy is needed there
```

The application never addresses the bucket itself. It uploads the document, keeps the identifier it receives, and later downloads the signed result using that identifier.

---

## Part 1 — Archive service (`dmss-archive-services`)

File: `application.yml`

The archive service needs a connection entry pointing at the fallback service. How documents are routed to it depends on your deployment.

### The fallback service as the archive (most common)

Give it the lowest `priority` number and it becomes the default archive — documents are stored there directly:

```yaml
archive-connections:
  connections:
    - name: "FS-MAIN"
      url: http://dmss-archive-services-fallback:8095/api
      type: EXTERNAL_FILE_SYSTEM
      priority: 1
```

### The fallback service as a failover target

If documents should normally go to another archive and only reach S3 when that archive is unavailable, name the entry in the primary's `failover` field:

```yaml
archive-connections:
  connections:
    - name: "OPENTEXT-MAIN"
      url: https://<primary-archive-host>/api
      type: OPENTEXT
      priority: 1
      failover: "FS-MAIN"
      metadataStorage: NATIVE

    - name: "FS-MAIN"
      url: http://dmss-archive-services-fallback:8095/api
      type: EXTERNAL_FILE_SYSTEM
      priority: 2
```

### Field reference

| Field | Description |
|---|---|
| `name` | Unique identifier for the connection. Other entries reference it by this name. |
| `url` | Base URL of the fallback service, including `/api`. |
| `type` | Always `EXTERNAL_FILE_SYSTEM` for the fallback service — this stays the same whether it stores to local disk or to S3. |
| `priority` | Order in which archives are tried. Lower is tried first; the lowest becomes the default archive. |
| `failover` | Optional. On another archive's entry: the `name` of the connection to retry against when a write fails. |

> **Note:** A connection pointing at `dmss-archive-services-fallback` almost certainly already exists in your deployment. If so, no change is needed here — continue to Part 2.

---

## Part 2 — Fallback service (`dmss-archive-services-fallback`)

File: `application.yml`

Add the `aws` block. Setting `aws.enabled: true` switches the fallback service from local directory storage to S3 for all document reads and writes.

### AWS S3

```yaml
server:
  port: 8095

fallback:
  directoryDepth: 3
  tempDirectory: /docs
  documentsDirectory: /docs
  actAsMainService: true
  tempDocumentType: 'temp'

aws:
  enabled: true
  s3:
    accessKey: ${AWS_ACCESS_KEY}
    secretKey: ${AWS_SECRET_KEY}
    region: eu-west-1
    bucket: my-archive-bucket
```

### S3-compatible storage (MinIO, on-premises object storage)

Add `customEndpoint` and `pathStyleAccessEnabled`:

```yaml
aws:
  enabled: true
  s3:
    accessKey: ${AWS_ACCESS_KEY}
    secretKey: ${AWS_SECRET_KEY}
    region: ""
    bucket: my-archive-bucket
    customEndpoint: http://minio:9000/
    pathStyleAccessEnabled: true
```

### Field reference

| Field | Required | Default | Description |
|---|---|---|---|
| `aws.enabled` | yes | `false` | Set to `true` to store documents in S3. When `false`, the service writes to the local directories configured under `fallback`. |
| `aws.s3.accessKey` | yes | — | Access key ID. Inject from an environment variable or secret store; do not commit a literal value. |
| `aws.s3.secretKey` | yes | — | Secret access key. Inject from an environment variable or secret store; do not commit a literal value. |
| `aws.s3.region` | conditional | — | Required for AWS S3 (e.g. `eu-west-1`). May be left empty (`""`) when `customEndpoint` is set and the target store does not use regions. |
| `aws.s3.bucket` | yes | — | Name of the bucket documents are written to. The bucket must already exist. |
| `aws.s3.customEndpoint` | no | *(unset)* | Base URL of an S3-compatible endpoint. Omit entirely for AWS S3. |
| `aws.s3.pathStyleAccessEnabled` | no | `false` | Set to `true` for endpoints requiring path-style URLs (typical for MinIO). Leave `false` for AWS S3. |

> The `fallback.tempDirectory` and `fallback.documentsDirectory` settings must remain present even when S3 is enabled — the service requires them at startup.

---

## Prerequisites

Before enabling S3, confirm the following:

1. **The bucket exists.** The service does not create it.
2. **Credentials are scoped to that bucket** and permit object read/write plus bucket-versioning configuration.
3. **Object versioning can be enabled.** At startup the fallback service enables versioning on the configured bucket — document versions rely on it. If the credentials lack permission to set the versioning configuration, the service will fail to start.
4. **Credentials are supplied as environment variables** (or via your secret manager) rather than literal values in the YAML file.

## Applying the change

1. Provision the bucket and an access key / secret key pair scoped to it.
2. Supply `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` to the fallback service's environment.
3. Add the `aws` block to the fallback service's `application.yml`, using the AWS or S3-compatible example above.
4. Confirm the archive service's fallback connection and `failover` settings (Part 1).
5. Restart the fallback service, then the archive service.

## Verifying

1. Confirm the fallback service starts cleanly — a startup failure here usually indicates an incorrect bucket name, endpoint, or insufficient credentials.
2. Archive a test document so that it is routed to the fallback service. If it is the default archive, archiving any document is sufficient; if it is configured only as a failover target, make the primary archive unavailable first.
3. Confirm the object appears in the bucket, and that the document can be retrieved through the archive service.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Fallback service fails to start | Bucket does not exist, endpoint unreachable, or credentials cannot set the bucket versioning configuration |
| Documents still written to local directories | `aws.enabled` is not `true`, or the service was not restarted after the change |
| Connection or signature errors against non-AWS storage | `customEndpoint` missing, or `pathStyleAccessEnabled` not set to `true` |
| Region errors against AWS S3 | `aws.s3.region` empty while `customEndpoint` is unset |

## Security

- Never commit access keys, secret keys, or environment-specific bucket names to version control.
- Use credentials scoped to the archive bucket only, and rotate them according to your organisation's policy.
