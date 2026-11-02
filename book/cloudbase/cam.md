---
title: CAM Users and Policies
summary: "Creating least-privilege CAM users for storage and email: custom policies, programmatic keys, rotation and retirement."
references:
  policies: "Tencent Cloud, *CAM 策略* (policies), [cloud.tencent.com/document/product/598/10601](https://cloud.tencent.com/document/product/598/10601)."
  camsyntax: "Tencent Cloud, *CAM 策略语法* (policy syntax), [cloud.tencent.com/document/product/598/10603](https://cloud.tencent.com/document/product/598/10603)."
  subuser: "Tencent Cloud, *创建子账号并授权* (creating and authorising a sub-account), [cloud.tencent.com/document/product/598/54458](https://cloud.tencent.com/document/product/598/54458)."
  accesskeys: "Tencent Cloud, *访问密钥* (access keys), [cloud.tencent.com/document/product/598/40488](https://cloud.tencent.com/document/product/598/40488)."
  updatepolicy: "Tencent Cloud, *UpdatePolicy* (更新策略信息, CAM API), [cloud.tencent.com/document/api/598/34569](https://cloud.tencent.com/document/api/598/34569)."
---

## Which parts need a CAM identity

The CloudBase server API key (Chapter 10) covers everything inside one
environment's gateway: the database, and through it the application's data.
It does not cover the account-level services that a deployment also uses.
The API needs its own credentials for two of them:

- **Object storage (COS).** The API signs the upload and download URLs that
  browsers use, and checks the bucket's settings before it signs (Chapter 6,
  Chapter 14). Signing requires a key pair whose owner may perform the signed
  operations.
- **Email (SES).** The API sends account email through Tencent SES's API
  (Chapter 7, Chapter 14), which authenticates every call with a key pair.

Both are served by **CAM** (Cloud Access Management, 访问管理), Tencent
Cloud's account-wide identity service. CAM grants nothing by default: a new
user can do nothing until a policy is attached to it [@policies]. The pattern
of this chapter follows Chapter 3: one **sub-user** per purpose and per
environment, with programmatic access only, holding exactly one custom
policy that names exactly the resources it needs.

!!! definition "Sub-user"
    A CAM identity created under the root account (子用户). A sub-user can be
    given console access, programmatic access through a SecretId/SecretKey
    pair, or both. An application's sub-users get programmatic access only,
    so that a leaked key cannot sign in to the console.

| User | Policy | Used by | Variables in the API service |
|---|---|---|---|
| `myapp-staging-storage` | `MyAppStagingStorageAccess` | API service of staging | `COS_SECRET_ID`, `COS_SECRET_KEY` |
| `myapp-staging-mail` | `MyAppStagingMailAccess` | API service of staging | `SES_SECRET_ID`, `SES_SECRET_KEY` |

Table: A naming convention for one environment's CAM users. The variable names
are the application's choice; the ones shown are examples.

Per-environment users cost nothing and pay for themselves on the day an
environment is retired: its users, policies and keys are deleted together,
and no other environment notices. A user shared between environments would
have to keep its broadest policy until the last of them was gone.

## Policies on Tencent Cloud

A CAM policy is a JSON document with a `version` and a list of statements.
Each statement has an `effect` (`allow` or `deny`), the `action`s it covers
(API names prefixed by the service, such as `cos:GetObject`), the `resource`s
it applies to, and an optional `condition` [@camsyntax]. Policy versions are
`"2.0"` or `"3.0"`; the policies in this book use `"2.0"`.

Resources are named by a six-part description. For COS it reads

```text
qcs::cos:<region>:uid/<appid>:<bucket>/<object-key-pattern>
```

where `<appid>` is the account's APPID, the number that also ends the bucket
name, and the part after the bucket is a key pattern in which `*` matches any
sequence of characters [@camsyntax]. A resource that ends at `<bucket>/`
names the bucket itself, which is what bucket-level actions such as reading
the ACL require.

!!! remark "APPID and UIN"
    The APPID identifies the account in resource names (and bucket names).
    The UIN identifies the root account and each user in CAM itself, for
    example in `qcs::cam::uid/<appid>:uin/<uin>`. The two numbers differ; a
    policy that names a bucket uses the APPID.

A user can carry several policies, and a policy can serve several users.
An edit to a policy can be saved as a new version; CAM keeps up to five
versions per policy, and any of them can be set as the current one again,
which makes a policy change reversible. A policy's name can also be changed through the
`UpdatePolicy` API, which identifies the policy by its ID [@updatepolicy].
Attachments refer to the policy's ID, so a rename breaks nothing.

## Recipe: a storage user

The storage policy grants exactly what an API that signs browser transfers
needs (Chapter 6): read the bucket's ACL and versioning, put and get objects
under the application's prefix, and list that prefix. It grants no delete,
no bucket mutation and no wildcard actions. The example assumes the
application keeps documents under `workspaces/<id>/documents/<id>`; replace
the prefix with the application's own.

```json
{
    "version": "2.0",
    "statement": [
        {
            "effect": "allow",
            "action": ["cos:GetBucketACL", "cos:GetBucketVersioning"],
            "resource": ["qcs::cos:<region>:uid/<appid>:<bucket>/"]
        },
        {
            "effect": "allow",
            "action": ["cos:PutObject", "cos:GetObject"],
            "resource": ["qcs::cos:<region>:uid/<appid>:<bucket>/workspaces/*/documents/*"]
        },
        {
            "effect": "allow",
            "action": ["cos:GetBucket"],
            "resource": ["qcs::cos:<region>:uid/<appid>:<bucket>/"],
            "condition": {"string_like": {"cos:prefix": "workspaces/*/documents/*"}}
        }
    ]
}
```

Listing a bucket (`GetBucket`) is a bucket-level action, so the object
pattern does not restrict it; the `cos:prefix` condition does, by admitting
only listing requests whose prefix matches. Grant `cos:HeadObject` as well
only if the application sends HEAD requests; signed URLs bind their method,
so a URL signed for GET cannot be used for HEAD.

!!! recipe "Creating the storage policy"
    1. Open the CAM console (`https://console.cloud.tencent.com/cam`) →
       策略 (policies) → 新建自定义策略 (new custom policy) → 按策略语法创建
       (create from policy syntax) → 空白模板 (blank template).
    2. Name it after its user, for example `MyAppStagingStorageAccess`, and
       paste the JSON above with the region, APPID and bucket filled in.
    3. Save. The console validates the syntax before it accepts the policy.

    **Check:** the policy's detail page lists one service, 对象存储 (cos),
    with the actions and the `cos:prefix` condition shown in its summary
    (策略摘要).

!!! recipe "Creating the storage user"
    1. CAM → 用户 → 用户列表 (user list) → 新建用户 (new user) → 自定义创建
       (custom creation) → 可访问资源并接收消息 (can access resources and
       receive messages) [@subuser].
    2. User name `myapp-staging-storage`. Under 访问方式 (access method),
       select 编程访问 (programmatic access) only; leave 控制台访问 (console
       access) unselected.
    3. Under 设置用户权限 (user permissions), search for the policy created
       above, select it, and finish.
    4. The last page shows the SecretId and SecretKey once [@accesskeys].
       Paste them directly into the API service's variables (Chapter 13). If
       the key file is downloaded instead, delete it after use; the key exists
       in exactly two places, the service and, if wanted, the password
       manager.

    **Check:** the user's detail page shows the policy attached and one
    enabled API key; its 访问方式 shows programmatic access only.

The surest test of a storage user is the application itself: an upload and a
download through the web application (Chapter 15). Before users exist, the
same rights can be probed from inside the API's container, with the keys the
container already holds, so that no secret leaves it. The script below uses
the COS Node.js SDK, which an API that signs COS URLs already bundles; run it
from the directory whose `node_modules` contains the SDK, through the
service's Webshell (Chapter 13).

!!! recipe "Probing a storage user from the API container"
    ```bash
    cat > /tmp/cos-probe.js <<'EOF'
    const COS = require('cos-nodejs-sdk-v5');
    const cos = new COS({ SecretId: process.env.COS_SECRET_ID, SecretKey: process.env.COS_SECRET_KEY });
    const Bucket = process.env.COS_BUCKET, Region = process.env.COS_REGION;
    const probe = async (label, call) => {
      try { await call(); console.log(label, 'allowed'); }
      catch (error) { console.log(label, error.statusCode); }
    };
    (async () => {
      await probe('bucket ACL', () => cos.getBucketAcl({ Bucket, Region }));
      await probe('versioning', () => cos.getBucketVersioning({ Bucket, Region }));
      await probe('missing document', () => cos.getObject({ Bucket, Region, Key: 'workspaces/probe/documents/probe' }));
      await probe('list documents', () => cos.getBucket({ Bucket, Region, Prefix: 'workspaces/probe/documents/', MaxKeys: 1 }));
      await probe('list bucket root', () => cos.getBucket({ Bucket, Region, Prefix: '', MaxKeys: 1 }));
    })();
    EOF
    node /tmp/cos-probe.js; rm /tmp/cos-probe.js
    ```

    The expected output is `allowed` for the ACL, versioning and the
    documents listing, `404` for the missing document (the request was
    authorised and the object does not exist), and `403` for the listing of
    the bucket root. A `403` on the missing document means the object
    statement does not match; a `403` on the ACL means the bucket statement
    does not.

## Recipe: an email user

The API's email user needs only to send. Creating and updating templates is
an operator's task (Chapter 14); when it is done from the API container for
convenience, the same user also needs the template actions. The policy below
grants both; drop the three template actions if templates are managed
elsewhere.

```json
{
    "version": "2.0",
    "statement": [
        {
            "effect": "allow",
            "action": [
                "ses:SendEmail",
                "ses:CreateEmailTemplate",
                "ses:UpdateEmailTemplate",
                "ses:ListEmailTemplates"
            ],
            "resource": ["*"]
        }
    ]
}
```

The statement names the actions and leaves the resource open, because SES
sender domains and templates are account-wide rather than per environment.
The user is created exactly as the storage user is, with programmatic access
only and this policy attached.

## Rotation and retirement

A key pair is rotated without interrupting the service by giving the user a
new key before removing the old one [@accesskeys]:

!!! recipe "Rotating a sub-user's key"
    1. CAM → 用户列表 → the user → API 密钥 → 新建密钥 (new key). Copy the
       new SecretId and SecretKey; the SecretKey is shown only now.
    2. Put the new pair into the service's variables and deploy a new version
       (Chapter 13).
    3. Verify: the probe above for a storage user, a test email for an email
       user, then one real transfer or message through the application.
    4. Back in the user's key list, choose 禁用 (disable) on the old key. Wait
       a day for anything that still uses it to fail loudly, then 删除
       (delete) it; deletion cannot be undone.

    **Check:** the user lists one enabled key, the new one, and the service
    works.

When an environment is retired (Chapter 15), its users and policies go with
it, but only after the environment itself is gone: while it sits in
isolation it is the rollback, and its keys must still work if it is renewed.
Delete the user first (which deletes its keys), then the policy it was the
only holder of.

!!! warning "Widening a shared policy"
    When a new environment needs the same kind of access as an old one, it is
    tempting to add the new bucket to the old policy. That gives the old
    environment's keys access to the new environment's data, and the old
    keys usually sit in the settings of a service that is about to be
    abandoned. Create a new user and policy instead; it costs two minutes and
    removes the question entirely.

!!! summary "Key points"
    - The API needs CAM credentials for COS and SES; the CloudBase API key
      does not cover them.
    - One programmatic-only sub-user per purpose and per environment, each
      with one custom policy naming exactly its resources.
    - COS resources read `qcs::cos:<region>:uid/<appid>:<bucket>/<pattern>`;
      bucket-level actions need the `<bucket>/` form, and listing is narrowed
      with a `cos:prefix` condition.
    - Probe a storage user from inside the container with its own keys:
      allowed, allowed, 404, allowed, 403.
    - Rotate by adding a key, deploying, verifying, disabling, then deleting.
      Retire users only after their environment is destroyed, and never widen
      an old policy to cover a new environment.
