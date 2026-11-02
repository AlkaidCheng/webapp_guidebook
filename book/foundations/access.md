---
title: Identity and Access
summary: "Who may do what: accounts, users and roles, policies, keys and their lifecycle, and the least-privilege habits that keep a deployment safe."
references:
  aws-root: "Amazon Web Services, *AWS account root user*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html>."
  aws-temp: "Amazon Web Services, *Temporary security credentials in IAM*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html>."
  gcp-iam: "Google Cloud, *IAM overview*, Identity and Access Management documentation, <https://docs.cloud.google.com/iam/docs/overview>."
  azure-rbac: "Microsoft, *What is Azure role-based access control (Azure RBAC)?*, Microsoft Learn, <https://learn.microsoft.com/en-us/azure/role-based-access-control/overview>."
  cam-elements: "Tencent Cloud, *Element Reference Overview*, Cloud Access Management documentation, <https://www.tencentcloud.com/document/product/598/10603>."
  aws-elements: "Amazon Web Services, *IAM JSON policy element reference*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html>."
  aws-eval: "Amazon Web Services, *Policy evaluation logic*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html>."
  aws-best: "Amazon Web Services, *Security best practices in IAM*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html>."
  aws-versions: "Amazon Web Services, *Versioning IAM policies*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-versioning.html>."
  gcp-sa: "Google Cloud, *Best practices for using service accounts securely*, IAM documentation, <https://docs.cloud.google.com/iam/docs/best-practices-service-accounts>."
  rfc7519: "M. Jones, J. Bradley and N. Sakimura, *JSON Web Token (JWT)*, RFC 7519 (2015), <https://www.rfc-editor.org/rfc/rfc7519>."
  owasp-secrets: "OWASP Cheat Sheet Series, *Secrets Management Cheat Sheet*, <https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html>."
  aws-rotate: "Amazon Web Services, *Update access keys*, IAM User Guide, <https://docs.aws.amazon.com/IAM/latest/UserGuide/id-credentials-access-keys-update.html>."
---

## Principals

Every request to a cloud provider's API is made by some *principal*, an
identity that the provider can authenticate, and is allowed or refused
according to the policies attached to that principal or to the resource it
touches. A deployment involves more principals than people: the API service
that writes to a bucket, the platform that pulls code from a repository and the
operator who opens the console each act under their own.

!!! definition "Principal"
    An authenticated identity on whose behalf a request is made: a person
    signing in to a console, or a program presenting a credential. Policies
    grant permissions to principals; a request carries exactly one.

Providers offer three kinds. The *root account* is the identity that created
the provider account and owns its billing. It has complete access to every
resource in the account, which is why providers recommend using it only for
the few tasks that require it, never for everyday work [@aws-root]. *Users*
are long-lived identities created inside the account, each with its own
credentials: a password for console access, an access-key pair for
programmatic access, or both. *Roles* have no credentials of their own; a
trusted principal assumes a role and receives temporary credentials that
expire after a set time [@aws-temp]. Table 3.1 gives the names each provider
uses.

| Provider | Account owner | Long-lived identity | Assumed identity |
|---|---|---|---|
| AWS | Root user [@aws-root] | IAM user | IAM role, with temporary credentials [@aws-temp] |
| Google Cloud | Organization or project owner | Service account [@gcp-iam] | Service-account impersonation [@gcp-sa] |
| Azure | Subscription owner | Service principal [@azure-rbac] | Managed identity [@azure-rbac] |
| Tencent Cloud | Root account (主账号) | Sub-user (子用户) [@cam-elements] | Role (角色) |

Table: The kinds of principal under each provider's names.

The distinction between console and programmatic access matters for every
identity a deployment creates. A person needs console access with a password
and a second factor, and no access keys. A program needs programmatic access,
and nothing else: an identity created for the API service to write to a
bucket should not be able to sign in to the console at all, so that a leaked
key cannot be used to explore the account through the web interface.

## Policies

A policy is a document that grants or denies permissions. Its unit is the
*statement*, and every provider's statements are built from the same four
parts: an *effect* (allow or deny), the *actions* it covers, the *resources*
they apply to, and optional *conditions* under which the statement holds
[@aws-elements; @cam-elements]. A policy attached to a resource, such as a
bucket policy, also names the *principal* it applies to. A request is refused
unless some statement allows it, and an explicit deny overrides any allow
[@aws-eval].

!!! example "A complete policy"
    The following AWS policy lets its holder read objects under one prefix of
    one bucket, and only over TLS. `amzn-s3-demo-bucket` is AWS's documented
    example bucket name.

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "ReadDocumentsOverTls",
          "Effect": "Allow",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::amzn-s3-demo-bucket/documents/*",
          "Condition": {
            "Bool": { "aws:SecureTransport": "true" }
          }
        }
      ]
    }
    ```

    `Version` names the policy language, not a revision of this policy.
    Tencent Cloud's CAM uses the same structure with lower-case keys
    (`version`, `statement`, `effect`, `action`, `resource`, `condition`) and
    language version `2.0` [@cam-elements]; [Chapter 11](../cloudbase/cam.md)
    gives the CAM policies for a storage and an email identity.

*Least privilege* is the rule that a principal receives the actions it uses,
on the resources it uses, and nothing more [@aws-best]. It limits what a
leaked credential or a bug can do: an identity that may only put and get
objects under `documents/` cannot delete the bucket, read another
application's data or create new users. The practical difficulty is that
providers name hundreds of actions, and a policy written by guessing is either
too broad or breaks the application in a path nobody tested. The cure is to
derive the policy from the calls the code makes.

!!! recipe "Deriving a least-privilege policy"
    1. List every provider API call the component makes, from the code (each
       SDK method or REST operation) rather than from memory.
    2. Map each call to its action name in the provider's reference.
    3. Scope each action to the narrowest resource: one bucket, one prefix,
       one sender identity. Add conditions where the provider offers them.
    4. Attach the policy to an identity that belongs to this component in
       this environment alone (Section 3.3).
    5. Verify both directions: one allowed call succeeds, and one plausible
       forbidden call, such as a delete, is refused with an access-denied
       error.
    6. Keep the policy document in the repository next to the code that needs
       it. A policy contains no secret, and a reviewed copy shows when the
       code's calls and the grant drift apart.

Policies change, and a change can break a running service. Providers that
keep *policy versions* make the change reversible: AWS stores up to five
versions of a managed policy, and setting an earlier version as the default
rolls a change back at once [@aws-versions]. Editing a policy in place is
therefore safer as "create a new version, set it as default, verify" than as an
overwrite.

## One identity per environment and purpose

Each component that calls a provider API gets its own identity, and each
environment gets its own set. The API service of the staging environment has a
storage identity and an email identity; the production API has two others.
This multiplies the number of identities, and it is worth it, for two reasons.
A credential then grants exactly one component in one environment what it
needs, so a key leaked from staging opens nothing in production. And
retiring an environment becomes a matter of deleting its identities, with no
risk of breaking another environment that shared one. Google's guidance
states the second reason directly: an identity shared between applications
makes it unclear whether it can ever be retired [@gcp-sa].

| Identity (convention) | Used by | Allowed to | Credential |
|---|---|---|---|
| `<app>-<env>-storage` | API service | Put and get objects under the document prefix of the environment's bucket ([Chapter 6](storage.md)) | Access-key pair |
| `<app>-<env>-email` | API service | Send through the environment's sender identity ([Chapter 7](email.md)) | Access-key pair |
| Platform API key | API service | Call the environment's own platform services, such as a database gateway ([Chapter 4](databases.md)) | Key issued by the platform |
| Deploy key | Release procedure | Push to the deployment repository ([Chapter 8](releases.md)) | SSH key pair |
| Operator | A person | Administer the account through the console | Password and second factor |

Table: The identities of one environment. The naming pattern is a
convention; what matters is that the name states environment and purpose.

!!! warning "Never share an identity across environments"
    An identity used by both staging and production cannot be rotated,
    restricted or deleted for one without affecting the other, so in practice
    it is never rotated, restricted or deleted. Create the per-environment
    identities when the environment is created, even when the policies are
    identical.

## Credentials and where they live

A credential is what a principal presents to prove its identity, and its
lifetime decides how a deployment fails. Table 3.3 lists the kinds a web
deployment meets.

| Kind | Lifetime | Stops working when | Typical use |
|---|---|---|---|
| Access-key pair (ID and secret) | Unlimited | Deactivated or deleted | A service calling a provider API from outside that provider's runtime |
| Temporary credentials | Minutes to hours | They expire [@aws-temp] | Workloads that run under a role inside the provider |
| Platform API key, often a JWT | As set at issue, possibly unlimited | Its `exp` time passes, or it is deleted | Calling the platform's own services |
| Console password | Until changed | Changed or disabled | People, together with a second factor |

Table: Kinds of credential.

Temporary credentials are the strongest choice wherever the workload runs
under a role, because a leaked copy stops working by itself [@aws-best]. A
service that runs on one platform and calls another provider's API cannot
assume roles there, and needs a long-lived access-key pair instead, whose
secret half is shown only once, at creation [@aws-rotate].

Platform API keys are frequently JSON Web Tokens. A signed JWT carries its
claims as a readable payload, and its expiry is the optional `exp` claim, a
time after which the token must be rejected [@rfc7519]. Some platforms let the
issuer choose a lifetime or no expiry at all, and the choice is a trade-off. A
key with no expiry never breaks the service on a deadline, but a leaked copy
stays valid until someone notices and deletes it. A key that expires limits
the damage of a leak, but turns forgetting to rotate it into an outage on a
known date. Where the platform can issue and rotate the key automatically, an
expiring key is better. Where rotation is manual, a key without expiry,
stored only in the service's secret variables, combined with a scheduled
rotation and an inventory entry, avoids the self-inflicted outage while
keeping the exposure bounded by the rotation interval.

Secrets live in exactly two places: the secret environment variables of the
service that uses them, and a password manager entry that records what the
secret is for (Chapter 1). The entry holds the identity it belongs to, the
environment, the services that use it, the creation date and any expiry, so
that rotation and retirement can be done from the record without
archaeology. Secrets never go into the repository, an image, a log line, a
ticket or a chat message [@owasp-secrets].

!!! warning "A secret pasted into chat is leaked"
    Chat messages, tickets and terminal transcripts are copied to servers,
    notifications, search indexes and backups beyond anyone's control. A
    secret that appeared in one is treated as known to others: rotate it at
    once, and record the rotation.

## Rotation and retirement

Rotation replaces a credential with a new one without interrupting the
service. Because providers such as AWS allow an identity two active access keys
at a time, the procedure can create the new key before removing the old one
[@aws-rotate]. The order is the whole point: create new, deploy, verify,
delete old.

!!! recipe "Rotating an access key"
    1. Create a second access key on the same identity. Store its secret in
       the password manager entry at once; it will not be shown again.
    2. Set the new key in the secret variables of every service that uses the
       identity, and release them.
    3. Verify that each service works with the new key, using a request that
       exercises the permission, such as an upload for a storage identity.
    4. Check that the old key is no longer used. Providers record a key's last
       use; wait until it stops advancing [@aws-rotate].
    5. Deactivate the old key, verify again, then delete it.
    6. Update the password manager entry with the new key's ID and the date.

Retirement is rotation's end state. When an environment is retired, its
identities and keys are deleted with it: a credential for an environment that
no longer exists has no legitimate user, so any use of it is an attacker's.
Disabling first and deleting after a waiting period is safer than deleting at
once, since a disabled identity can be restored if something still depended on
it, while a deleted one recreated under the same name is a new identity
without its old bindings [@gcp-sa]. An inventory of identities per
environment, kept with the password manager entries, makes this step a
checklist instead of a search.

[Chapter 10](../cloudbase/environment.md) issues the platform API key of a
CloudBase environment, and [Chapter 11](../cloudbase/cam.md) creates the storage
and email identities with their policies, keys and rotation.

!!! summary "Key points"
    - Principals are the root account, users and roles. Use the root account
      only for tasks that require it; give programs programmatic access only.
    - Policy statements combine effect, action, resource and condition.
      Everything not allowed is refused, and an explicit deny wins. Derive
      policies from the calls the code makes, and use policy versions to
      roll back.
    - Give each component in each environment its own identity, so that a
      leak stays in one place and an environment can be retired by deleting
      its identities.
    - Prefer temporary credentials; a never-expiring key cannot break on a
      deadline but stays valid when leaked. Choose deliberately and record the
      choice.
    - Secrets live in the services' secret variables and a password manager,
      never in the repository, images, logs or chat.
    - Rotate as create new, deploy, verify, delete old; delete an
      environment's identities when it is retired.
