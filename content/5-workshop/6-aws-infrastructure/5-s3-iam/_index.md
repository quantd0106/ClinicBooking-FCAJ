---
title: "5.6.5. Private S3 and IAM"
weight: 5
chapter: false
---

# Private S3 and IAM

## Bucket protection

`ProfileFilesBucket` is an `AWS::S3::Bucket` with the following source-defined controls:

| Setting | Current template |
|---|---|
| Public access | All four Block Public Access options enabled. |
| Encryption | `AES256` server-side encryption (SSE-S3). |
| Object ownership | `BucketOwnerEnforced`. |
| Bucket policy | Denies `s3:*` on bucket and objects when `aws:SecureTransport` is false. |
| Multipart lifecycle | Aborts incomplete multipart uploads after 1 day; not a rule deleting completed profile files. |
| Removal / replacement | `DeletionPolicy: Retain`, `UpdateReplacePolicy: Retain`. |

The real public-access excerpt:

```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```

There is no public Allow statement in the bucket policy. Anonymous direct object GET should not expose the private object. The policy also does **not** restrict all requests to a VPC endpoint; private bucket access and VPC-only access are different controls. An authorized client's presigned POST can therefore use HTTPS from outside the VPC.

## Authenticated, constrained upload

The implementation uses **presigned POST**, returning an upload URL and form fields rather than a presigned PUT URL.

1. A doctor sends an authenticated `POST /api/v1/files/presigned-upload` request.
2. JWT and role guards allow **DOCTOR only**. The backend checks that the caller has a doctor profile and validates `fileName` and `contentType`.
3. The server generates the object key and signs a POST policy with content-type and size constraints.
4. The API returns `objectKey`, `uploadUrl`, `formFields`, `expiresInSeconds`, and `maxFileSizeBytes`.
5. The client uploads the file directly to S3 using the returned multipart POST form; S3 enforces the signed policy.

| Restriction | Verified behavior |
|---|---|
| JPEG | `image/jpeg`, extension `.jpg` or `.jpeg`. |
| PNG | `image/png`, extension `.png`. |
| WEBP | `image/webp`, extension `.webp`. |
| Filename | Trimmed plain filename, maximum 255 characters; path separators disallowed. |
| Type matching | Declared MIME type must match the filename extension. |
| Actual upload size | Signed POST policy permits 1 byte through **5 MiB** (`5 * 1024 * 1024` bytes). |
| Expiry | Configurable; template and application default are 300 seconds. |
| Server key | Pattern `doctors/<doctor-id>/avatar/<random-uuid><extension>`; client does not choose the full key. |

The API request has no size field and carries no file binary. The backend sets the size limit in the signed policy; **S3 checks the actual uploaded size**. MIME/extension checks do not amount to inspection of the file's binary signature.

A short source excerpt from `src/files/s3-presigner.service.ts`:

```typescript
return createPresignedPost(this.client, {
  Bucket: bucket,
  Key: objectKey,
  Expires: expiresInSeconds,
  Fields: { 'Content-Type': contentType },
  Conditions: [
    ['eq', '$Content-Type', contentType],
    ['content-length-range', 1, maximumBytes],
  ],
});
```

Direct upload keeps the binary out of the API/Lambda request path for this use case. Authorization still begins at the authenticated API, and a temporary signed form is constrained to the generated object key.

## Actual Lambda execution-role permissions

`LambdaExecutionRole` trusts `lambda.amazonaws.com`. Its inline policies are:

| Policy | Verified permissions | Scope / limitation |
|---|---|---|
| `RestrictedCloudWatchLogs` | `logs:CreateLogStream`, `logs:PutLogEvents`. | Function log-group scope; the template creates log groups separately. |
| `LambdaVpcNetworking` | EC2 create/describe/delete network interface and assign/unassign private IP operations. | `Resource: '*'`; broader networking permissions, not perfect least privilege. |
| `RuntimeSecrets` | `secretsmanager:GetSecretValue`. | Managed RDS secret and `JwtSecret` references only. |
| `DoctorProfileUploads` | `s3:PutObject`. | Bucket's `doctors/*` objects only; no GetObject, ListBucket, DeleteObject, or S3 full-access grant here. |

`S3GatewayEndpoint` permits GetObject and PutObject for that prefix, but the endpoint policy does not grant the Lambda role GetObject. IAM and endpoint permissions must both allow a request. The two declared CloudWatch log groups use 14-day retention.

The role already limits secrets, uploads, and logging. Possible future hardening includes reviewing supported conditions for the broad EC2 networking permissions and reviewing the broader SG egress described in 5.6.2. These are separate design considerations; no IAM or SG change is made in this documentation step.

{{< notice warning >}}
Keep the bucket private and treat presigned form values as temporary authorization. Bucket retention can leave objects after stack removal, so later cleanup planning must account for retained storage. No cleanup or live access test is performed here.
{{< /notice >}}

<!-- TODO_SCREENSHOT: S3 Block Public Access and encryption settings, with bucket/private identifiers redacted. -->
