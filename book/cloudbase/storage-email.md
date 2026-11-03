---
title: Storage and Email
summary: "Preparing a COS bucket for browser transfers and setting up Tencent SES: sender domain, templates and review."
references:
  corsconcept: "Tencent Cloud, *跨域访问* (cross-origin access in COS), [cloud.tencent.com/document/product/436/11488](https://cloud.tencent.com/document/product/436/11488)."
  corssetup: "Tencent Cloud, *设置跨域访问* (setting a CORS rule), [cloud.tencent.com/document/product/436/13318](https://cloud.tencent.com/document/product/436/13318)."
  sesstart: "Tencent Cloud, *邮件推送快速入门* (SES quick start), [cloud.tencent.com/document/product/1288/47454](https://cloud.tencent.com/document/product/1288/47454)."
  sesdomain: "Tencent Cloud, *发信域名* (sender domains), [cloud.tencent.com/document/product/1288/55191](https://cloud.tencent.com/document/product/1288/55191)."
  sesaddress: "Tencent Cloud, *发信地址* (sender addresses), [cloud.tencent.com/document/product/1288/55192](https://cloud.tencent.com/document/product/1288/55192)."
  sestemplate: "Tencent Cloud, *发信模板* (email templates), [cloud.tencent.com/document/product/1288/55193](https://cloud.tencent.com/document/product/1288/55193)."
  sesapi: "Tencent Cloud, *邮件推送 API 文档* (SES API reference), [cloud.tencent.com/document/product/1288/51062](https://cloud.tencent.com/document/product/1288/51062)."
---

## Choosing a bucket

Attachments live in COS (Cloud Object Storage), and the application reaches
them through signed URLs (Chapter 6). Two kinds of bucket are available to a
CloudBase deployment, and the choice between them decides what happens when
the environment ends.

| | Environment bucket | Standalone bucket |
|---|---|---|
| Name | `<prefix>-<env-id>-<appid>`, tagged 云开发桶 | Chosen freely, `<name>-<appid>` |
| Created | With the environment | In the COS console |
| Billed | Within the environment's package | Separately, pay-as-you-go COS |
| Lifetime | Destroyed with the environment | Until deleted |
| Bucket policy | Pre-set grants to CloudBase and its CDN | Empty |

Table: The two kinds of bucket a CloudBase deployment can use.

The environment bucket costs nothing extra and is the natural choice for a
deployment that stays in one environment. Its lifetime is the catch: when an
application moves to another environment (Chapter 15), its objects must be
copied to the new environment's bucket before the old environment is
destroyed. A standalone bucket outlives every environment and makes a move
cheaper, at the price of a separate bill and its own setup. An application
whose database stores object keys rather than bucket names (Chapter 6) can
switch between the two later by copying objects under the same keys.

The environment bucket's pre-set bucket policy grants CloudBase's own service
and its CDN access to the bucket; it is how CloudBase's storage features work,
and it does not change the bucket's ACL. An application that signs its own
URLs neither needs nor uses those grants, and should leave them alone.

## Recipe: preparing a bucket

!!! recipe "Checking the bucket's access settings"
    1. Open the COS console (<https://console.cloud.tencent.com/cos>) →
       存储桶列表 (bucket list) → the bucket.
    2. 概览 (overview): 访问权限 (access) must read 私有读写 (private read and
       write), and the region is the one the application will be configured
       with.
    3. 权限管理 (permissions): the user ACL lists only the root account
       (主账号) with 完全控制 (full control).
    4. 容错容灾管理 (fault tolerance) → 版本控制 (versioning): 未开启 (not
       enabled). An application that relies on COS's overwrite guard needs a
       bucket that has never been versioned (Chapter 6).

    **Check:** all three settings read as above; anything else is changed
    before the application uses the bucket.

Browsers upload and download directly to COS with the signed URLs, so the
bucket must allow the web origin by CORS [@corsconcept]. The rule names the
origin exactly, the methods the URLs are signed for, the request headers the
upload sends, and the response headers the page needs to read.

!!! recipe "Adding the CORS rule"
    1. In the bucket, 安全管理 (security) → 跨域访问 CORS 设置 (CORS settings)
       → 添加规则 (add rule) [@corssetup].
    2. 来源 Origin: the web origin, exactly, for example
       `https://<web-service-domain>`. One origin per line; no wildcard.
    3. 操作 Methods: `PUT`, `GET`, `HEAD`.
    4. Allow-Headers: the headers the application's signed uploads carry,
       one per line, for example `content-type`, `content-encoding`,
       `content-length`, `x-cos-acl`, `x-cos-forbid-overwrite`,
       `x-cos-meta-sha256`, `x-cos-server-side-encryption`.
    5. Expose-Headers: `ETag`, `Content-Length`, `Content-Encoding`,
       `x-cos-request-id`, and any `x-cos-meta-*` header the page reads.
    6. 超时 Max-Age: `600`. 返回 Vary: Origin: off.
    7. 保存 (save). The environment bucket's existing rules for CloudBase's
       own consoles stay as they are.

    **Check:** a preflight from the allowed origin succeeds with the
    configured headers, and one from any other origin is refused:

    ```bash
    BUCKET_URL=https://<bucket>.cos.<region>.myqcloud.com
    for origin in https://<web-service-domain> https://example.com; do
      echo "== $origin"
      curl -s -o /dev/null -D - -X OPTIONS "$BUCKET_URL/workspaces/probe/documents/probe" \
        -H "Origin: $origin" -H 'Access-Control-Request-Method: PUT' \
        -H 'Access-Control-Request-Headers: content-type,x-cos-meta-sha256' \
        | grep -i -E '^HTTP|^access-control'
    done
    ```

    The first answers `200` with `Access-Control-Allow-Origin` naming the
    origin; the second answers `403`.

A new web origin, after a move or when a custom domain is added, needs its
own line in the rule; until it has one, every attachment transfer from that
origin fails in the browser while the API reports no error.

The API then needs a CAM user whose policy names this bucket (Chapter 11),
and four variables: the bucket name, its region, and the user's SecretId and
SecretKey. Probe the user from the API container before relying on it
(Recipe 11.3).

## Email with Tencent SES

Tencent SES (邮件推送) sends the application's account email: verification
codes, password resets and invitations (Chapter 7). It is an account-level
service, set up once and shared by the application's environments, in one of
its regions; the API names that region in its configuration [@sesstart].

The account's verification level shapes the integration. In 2026, an
account under individual real-name verification could send only reviewed
templates filled with each message's values, and could not use SES's SMTP
interface. Such an application calls the SES API [@sesapi] with template IDs
rather than composing messages itself.

!!! recipe "Setting up a sender domain and address"
    1. Choose a subdomain for sending, such as `mail.example.com`. A domain
       that already carries business mailboxes must not be used as a sender
       domain, because its SPF and MX records would conflict [@sesdomain].
    2. Open the SES console (<https://console.cloud.tencent.com/ses>) →
       邮件配置 (email settings) → 发信域名 (sender domains) → 新建发信域名
       (new sender domain): enter the subdomain and submit.
    3. The console lists the DNS records to publish: an MX record, an SPF TXT
       record, a DKIM TXT record and a DMARC TXT record (Chapter 7). Add them
       at the domain's DNS provider exactly as listed. On DNSPod they are
       ordinary records on the subdomain's hosts.
    4. Back in 发信域名, run the verification. The domain stays in 待验证
       (awaiting verification) until every record resolves.
    5. 邮件配置 → 发信地址 (sender addresses): create the address the
       application sends from, such as `noreply@mail.example.com`, with the
       display name users will see [@sesaddress].

    **Check:** the domain shows as verified and the address is listed; a
    test message from the console's sending page arrives with SPF, DKIM and
    DMARC all passing in the received headers.

Templates are created per message and language. A template is plain text or
HTML with variables written `{{name}}` [@sestemplate], and each goes through
SES's review, which took about one working day in 2026. Templates are
account-wide, so a template created for one environment can be used by
another; a template whose text contains the web address, such as an
invitation link, is tied to that address, and a move to a new web origin
needs new templates and a new review.

!!! recipe "Creating templates"
    1. SES console → 邮件配置 → 发信模板 (templates) → 新建 (new): name,
       type (纯文本 for plain text or HTML), body with `{{variables}}`;
       预览 (preview), then 提交 (submit) [@sestemplate].
    2. Name templates systematically, for example
       `<app>_<message>_<language>`, since the API is configured with a map
       from message and language to template ID.
    3. When there are many templates, create them through the API instead
       (`CreateEmailTemplate`), from the application's own wording, so the
       console text and the code cannot drift apart. The same tool can list
       them with their review state (`ListEmailTemplates`).
    4. Wait for review. In 2026 the API's template list reported an approved
       template with `TemplateStatus` 0.
    5. Configure the API with the region, the sender address, the template
       ID map and the email user's keys (Chapter 11). An API that checks at
       startup that every message has a template ID fails early when one is
       missing.

    **Check:** trigger a verification email to a test inbox through the
    application, and confirm the code arrives and works.

Keep the previous templates approved until a move has been verified: they are
what a rollback to the old environment sends with.

!!! summary "Key points"
    - The environment bucket is free but destroyed with the environment; a
      standalone bucket outlives it. Store object keys, not bucket names, so
      either choice can be revisited.
    - A usable bucket is private, owner-only in its ACL and never versioned,
      with a CORS rule for the exact web origin (PUT, GET, HEAD, the upload
      headers, the exposed headers, Max-Age 600). Check it with two
      preflights.
    - SES is account-level. Send from a verified subdomain with MX, SPF,
      DKIM and DMARC records, through the API and reviewed templates when the
      account is individually verified.
    - Templates are per message and language, reviewed in about a working
      day; templates that contain the web address are per origin.
