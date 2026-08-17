# Migration: `logs` output shape change

## What changed

The module's `logs` output no longer exposes the entire `module.logs` object.
It now returns a structured object containing only the log bucket's specific,
non-deprecated attributes.

### Before

```hcl
output "logs" {
  value = module.logs
}
```

Consumers accessed it as the whole submodule object, e.g.:

```hcl
module.cloudfront_s3_cdn.logs.bucket_id
module.cloudfront_s3_cdn.logs.bucket_arn
module.cloudfront_s3_cdn.logs.bucket_domain_name
```

### After

```hcl
output "logs" {
  value = {
    bucket_id                          = module.logs.bucket_id
    bucket_arn                         = module.logs.bucket_arn
    bucket_domain_name                 = module.logs.bucket_domain_name
    prefix                             = module.logs.prefix
    bucket_notifications_sqs_queue_arn = module.logs.bucket_notifications_sqs_queue_arn
    enabled                            = module.logs.enabled
  }
}
```

## Why

On AWS provider `>= 6.x`, the legacy inline attributes/blocks on the
`aws_s3_bucket` resource (`cors_rule`, `grant`, `website_endpoint`,
`versioning`, `logging`, `server_side_encryption_configuration`, `website`,
etc.) are marked **deprecated** *and* **Computed**. Because they are Computed,
the provider populates them in state on every read/refresh even when they are
never configured.

Referencing the **whole `module.logs` object** caused Terraform to evaluate the
entire object graph, reach those populated-but-deprecated attributes, and emit
warnings such as:

```
Warning: Deprecated value used

  The deprecation originates from module.cdn.aws_s3_bucket.origin[0].cors_rule

  cors_rule is deprecated. Use the aws_s3_bucket_cors_configuration resource instead.

(and 180 more similar warnings elsewhere)
```

These warnings were spurious — the module already uses the separate
`aws_s3_bucket_cors_configuration`, `aws_s3_bucket_website_configuration`,
etc. resources and never sets the deprecated inline blocks. Narrowing the
output to specific attributes stops Terraform from walking the deprecated
attributes, eliminating the warnings.

## Migration steps

If you referenced the `logs` output, the following access patterns are
**unchanged** and continue to work:

| Attribute                            | Still available |
| ------------------------------------ | --------------- |
| `logs.bucket_id`                     | Yes             |
| `logs.bucket_arn`                    | Yes             |
| `logs.bucket_domain_name`            | Yes             |
| `logs.prefix`                        | Yes             |
| `logs.bucket_notifications_sqs_queue_arn` | Yes        |
| `logs.enabled`                       | Yes             |

Any **other** attribute previously reachable via the whole `module.logs` object
(for example the log bucket's IAM user outputs, region, or website endpoints)
is no longer exposed through this output. If you depend on one of those, open an
issue so it can be added explicitly, or source the log bucket data directly in
your root configuration.
