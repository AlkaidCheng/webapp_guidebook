---
title: Object Storage
summary: "Buckets, keys and signed URLs, browser transfers and CORS, and the policies that let an API use a bucket and nothing more."
references:
  s3-prefixes: "Amazon Web Services, *Organizing objects using prefixes*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-prefixes.html>."
  s3-policy-keys: "Amazon Web Services, *Bucket policy examples using condition keys*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/amazon-s3-policy-keys.html>."
  s3-public: "Amazon Web Services, *Blocking public access to your Amazon S3 storage*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html>."
  owasp-upload: "OWASP Cheat Sheet Series, *File Upload Cheat Sheet*, <https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html>."
  s3-presigned: "Amazon Web Services, *Download and upload objects with presigned URLs*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html>."
  gcs-signed: "Google Cloud, *Signed URLs*, Cloud Storage documentation, <https://docs.cloud.google.com/storage/docs/access-control/signed-urls>."
  azure-sas: "Microsoft, *Grant limited access to Azure Storage resources using shared access signatures (SAS)*, Microsoft Learn, <https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview>."
  s3-conditional: "Amazon Web Services, *How to prevent object overwrites with conditional writes*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html>."
  cos-put: "Tencent Cloud, *PUT Object*, Cloud Object Storage API documentation, <https://www.tencentcloud.com/document/product/436/7749>."
  cos-versioning: "Tencent Cloud, *Versioning: Overview*, Cloud Object Storage documentation, <https://www.tencentcloud.com/document/product/436/19883>."
  s3-versioning: "Amazon Web Services, *Retaining multiple versions of objects with S3 Versioning*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html>."
  mdn-preflight: "MDN Web Docs, *Preflight request*, <https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request>."
  s3-cors-elements: "Amazon Web Services, *Elements of a CORS configuration*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/ManageCorsUsing.html>."
  s3-multipart: "Amazon Web Services, *Uploading and copying objects using multipart upload in Amazon S3*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html>."
  mdn-expose: "MDN Web Docs, *Access-Control-Expose-Headers header*, <https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Expose-Headers>."
  s3-cors-trouble: "Amazon Web Services, *Troubleshooting CORS*, Amazon S3 User Guide, <https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors-troubleshooting.html>."
---

## Buckets and keys

Object storage keeps files as *objects* in *buckets*. A bucket is a named
container in one region with its own settings: who may read and write it,
whether it keeps old versions, which browser origins may reach it. An object is
a sequence of bytes with some metadata, stored under a *key*.

!!! definition "Object key and prefix"
    An object's key is its full name within the bucket, an arbitrary string
    such as `documents/7f3a/2c91e0`. The namespace is flat: there are no
    directories, and a *prefix* is simply the beginning of a key. The slashes
    that make keys look like paths matter only to list operations, which can
    group keys by a delimiter, and to consoles that draw them as folders
    [@s3-prefixes; @s3-policy-keys].

A bucket for application data is private. Nobody reads or writes it without a
credential or a signed URL, and its access control list grants nothing to
anonymous users. Providers make this the default for new buckets and offer
account-wide switches that block public access whatever a later policy says
[@s3-public].

Three design decisions about keys pay off for the whole life of the
application. Keys are chosen by the application, never taken from the name of
the uploaded file, which may collide, contain path tricks, or reveal personal
information [@owasp-upload]; the file name is metadata, kept in the database. A
key starts with a fixed prefix that identifies the kind of object, such as
`documents/`, so that permissions and list operations can be restricted to it.
And the database stores the key alone, not the bucket name or a full URL. A
bucket can then be replaced by copying its objects under the same keys into a
new bucket and changing one configuration variable, with no rewrite of stored
data.

!!! remark "Platform buckets belong to their environment"
    Some platforms create a bucket as part of each environment and delete it
    with the environment. A standalone bucket, created in the provider's
    storage service directly, outlives any environment. Data that must
    survive a move between environments ([Chapter 8](releases.md)) is safer in
    a standalone bucket; a platform bucket suits data that the environment
    alone needs.

## Signed URLs

The API service never carries file contents. It authorizes a transfer and
issues a URL with which the browser performs it directly
(Figure 1.1).

!!! definition "Signed URL"
    A URL that carries, in its query string, a signature over one storage
    request made with the signer's credential. It grants whoever holds it
    permission to perform that request, and nothing else, until it expires.
    The signature covers the HTTP method, the bucket and key, the expiry, and
    any headers named as signed, so a request that differs in any of them
    fails [@s3-presigned]. Google Cloud Storage calls the same mechanism a
    signed URL [@gcs-signed], Azure a shared access signature [@azure-sas].

A signed URL is a bearer token: anyone who holds it can use it, as often as
they like, until it expires [@s3-presigned]. Its lifetime is therefore kept to
minutes, enough to start the transfer; a download that has started may finish
after expiry. The URL also cannot outlive the credential that signed it, so a
URL signed with temporary credentials expires with them, and deleting a key
invalidates every URL it signed [@s3-presigned].

An upload proceeds in four steps:

1. The browser asks the API for permission to upload, stating the file's
   name, size and type.
2. The API checks that the user may upload there, chooses a new key, records
   a pending object in the database, and signs a `PUT` for that key, binding
   the content type and, where the store supports it, the size or checksum.
3. The browser sends the file to the signed URL, with exactly the signed
   headers.
4. The browser tells the API the upload is complete; the API confirms that the
   object exists with the expected size and marks it available.

A download is one step: the API checks access and returns a short-lived signed
`GET`. The browser follows it, or the API answers with a redirect to it.

### Preventing overwrites

A signed `PUT` is bound to a key that the API chose to be new, but nothing in
the signature forbids the key from already existing: a replayed URL, or a bug
that reuses a key, would replace an existing object silently. Stores offer a
request header that makes a write conditional on the key being free. S3 refuses
a `PUT` carrying `If-None-Match: *` with `412 Precondition Failed` when an
object exists under the key [@s3-conditional]; Tencent Cloud COS refuses one
carrying `x-cos-forbid-overwrite: true` [@cos-put]. Binding the header into
the signature makes it mandatory: a request without it does not match the
signature and fails.

The guard's interaction with versioning differs between stores. On S3 it
applies to the current version in a versioned bucket as well
[@s3-conditional]. COS documents its header for buckets without versioning, and
states that on a bucket with versioning enabled the header has no effect and
overwrites are allowed [@cos-put]. Because a bucket that has once had
versioning enabled can never return to the unversioned state, only be
suspended [@cos-versioning; @s3-versioning], a design that relies on the COS
guard needs a bucket that has never been versioned.

!!! warning "Check the versioning state at startup"
    Versioning can be switched on in a console by anyone with the permission,
    and on COS that silently disables the overwrite guard. An API that relies
    on the guard reads the bucket's versioning state at startup and refuses
    to start if versioning has ever been enabled, as part of the startup
    check of [Chapter 5](services.md). The same check reads the bucket's
    access control list and refuses to start if the bucket is public.

## Browser transfers and CORS

A `PUT` from a page on `https://<web-origin>` to the storage endpoint is a
cross-origin request, and because its method is `PUT` and it sends a
`Content-Type` that a form could not, the browser preflights it
[@mdn-preflight]. The storage service answers the preflight from the bucket's
CORS configuration, a list of rules each naming allowed origins, methods and
request headers, the response headers to expose, and how long the browser may
cache the answer [@s3-cors-elements].

!!! example "A bucket CORS rule"
    The following S3 configuration allows uploads and downloads from one
    origin. Other stores use the same fields under similar names
    ([Chapter 14](../cloudbase/storage-email.md) gives the COS form).

    ```json
    [
      {
        "AllowedOrigins": ["https://www.example.com"],
        "AllowedMethods": ["GET", "PUT", "HEAD"],
        "AllowedHeaders": ["content-type", "if-none-match"],
        "ExposeHeaders": ["ETag"],
        "MaxAgeSeconds": 600
      }
    ]
    ```

Each field has a reason. The origin is exact: the scheme, host and port of the
web service, with no wildcard, so that no other site's scripts can use a URL
that leaks to them. The methods are those the application uses. The allowed
headers are those the browser sends that a plain request could not, which
includes every header bound into the signature; a signed header missing here
makes the preflight fail before the signature is ever checked. `ETag` is
exposed because the browser cannot read response headers outside a small safe
set unless the response lists them [@mdn-expose], and a browser-driven
multipart upload must collect each part's `ETag` to complete the upload
[@s3-multipart]. The maximum age saves one preflight per upload for that many
seconds.

!!! recipe "Verifying a bucket's CORS rule with curl"
    Send the preflight that a browser would send, first from the allowed
    origin:

    ```bash
    curl -si -X OPTIONS 'https://<storage-endpoint>/documents/probe' \
      -H 'Origin: https://<web-origin>' \
      -H 'Access-Control-Request-Method: PUT' \
      -H 'Access-Control-Request-Headers: content-type'
    ```

    The expected answer is `200` with `Access-Control-Allow-Origin` naming
    `https://<web-origin>`, `Access-Control-Allow-Methods` including `PUT`,
    `Access-Control-Allow-Headers` including `content-type`,
    `Access-Control-Expose-Headers` including `ETag`, and
    `Access-Control-Max-Age`. Then repeat the request with
    `Origin: https://foreign.example.com`. The expected answer is `403`: S3
    refuses a preflight that matches no rule with `403 Forbidden` and the
    message *This CORS request is not allowed* [@s3-cors-trouble]. A foreign
    origin that gets `200` means the rule is too broad.

The rule names origins, so it changes whenever the application's origin does:
staging and production need their own entries, and a move to a new domain
adds the new origin before the switch ([Chapter 2](domains.md)).

## The minimal policy

The API's storage identity ([Chapter 3](access.md)) needs a handful of
permissions, all of them derived from the flows above (Table 6.1).

| Permission | Scope | Needed for |
|---|---|---|
| Read the bucket's access control list | The bucket | The startup check that the bucket is private |
| Read the bucket's versioning state | The bucket | The startup check that it has never been versioned |
| Put an object | Keys under `documents/` | Signing upload URLs |
| Get an object and its metadata | Keys under `documents/` | Signing download URLs; confirming an upload |
| List objects | Only with the prefix `documents/` | Reconciliation and checks |
| Delete an object | Not granted | |

Table: The storage permissions of an API service.

The absence of delete is deliberate. The application deletes documents
logically, by marking the database row, so the API never needs to remove an
object, and a leaked key or a bug cannot destroy stored files. Removing
objects that are no longer referenced is a separate, occasional task, run under
a separate identity.

!!! example "The minimal policy in AWS syntax"
    `amzn-s3-demo-bucket` is AWS's documented example bucket. Listing is
    allowed only when the request names a prefix under `documents/`, through
    the `s3:prefix` condition key [@s3-policy-keys].

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "InspectBucketSettings",
          "Effect": "Allow",
          "Action": ["s3:GetBucketAcl", "s3:GetBucketVersioning"],
          "Resource": "arn:aws:s3:::amzn-s3-demo-bucket"
        },
        {
          "Sid": "ReadAndWriteDocuments",
          "Effect": "Allow",
          "Action": ["s3:PutObject", "s3:GetObject"],
          "Resource": "arn:aws:s3:::amzn-s3-demo-bucket/documents/*"
        },
        {
          "Sid": "ListDocumentsOnly",
          "Effect": "Allow",
          "Action": "s3:ListBucket",
          "Resource": "arn:aws:s3:::amzn-s3-demo-bucket",
          "Condition": {
            "StringLike": { "s3:prefix": "documents/*" }
          }
        }
      ]
    }
    ```

    [Chapter 11](../cloudbase/cam.md) gives the equivalent CAM policy for a
    COS bucket such as `examplebucket-1250000000`.

## Replacing a bucket

Because the database stores keys and the bucket name lives in configuration,
replacing a bucket is a copy and a configuration change. The new bucket is
created with the same settings (private, never versioned, the CORS rule for
the current origins); the objects are copied under the same keys with the
provider's copy tools; the copy is checked by listing both buckets and
comparing key, size and count; and the API's bucket variable is changed and
released. Uploads made during the copy are caught by freezing writes, or by a
second pass that copies the keys added since the first. The old bucket stays,
untouched, until the new one has served for long enough to be trusted, and is
deleted only after that.

[Chapter 14](../cloudbase/storage-email.md) prepares a COS bucket for browser
transfers in this way, and [Chapter 15](../cloudbase/operations.md) moves a
deployment's objects with the rest of the environment.

!!! summary "Key points"
    - Keys are flat strings chosen by the application under a fixed prefix;
      store keys, not bucket names or URLs, so that a bucket can be replaced
      by copying.
    - Keep application buckets private. Prefer a standalone bucket for data
      that must outlive an environment.
    - Signed URLs are short-lived bearer permissions for one method on one
      key, bound to the signed headers; the API signs, the browser
      transfers.
    - An overwrite guard header stops a signed `PUT` from replacing an
      existing object. On COS it has no effect once versioning has been
      enabled, so such a design needs a never-versioned bucket and a startup
      check.
    - Direct browser transfers need a bucket CORS rule with the exact origin,
      the methods, the signed headers, `ETag` exposed and a max-age; verify it
      with a preflight from the allowed origin (`200`) and a foreign one
      (`403`).
    - Grant the API read access to the bucket's ACL and versioning, put and
      get under the document prefix, listing restricted to that prefix, and
      no delete.
