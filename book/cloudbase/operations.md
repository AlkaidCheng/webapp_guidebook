---
title: Operating and Moving an Environment
summary: "Verifying a deployment, watching usage and cost, and moving a running application to another environment."
references:
  pricing: "Tencent Cloud, *云开发 CloudBase 计费说明* (billing), [cloud.tencent.com/document/product/876/127357](https://cloud.tencent.com/document/product/876/127357)."
  usage: "Tencent CloudBase, *tcb env usage*, [docs.cloudbase.net/cli-v1/envs/usage](https://docs.cloudbase.net/cli-v1/envs/usage)."
  createenv: "Tencent CloudBase, *创建云开发环境* (creating an environment), [docs.cloudbase.net/quick-start/create-env](https://docs.cloudbase.net/quick-start/create-env)."
  revisions: "Tencent CloudBase, *查看或删除版本* (viewing and deleting versions), [docs.cloudbase.net/run/deploy/managing/revisions](https://docs.cloudbase.net/run/deploy/managing/revisions)."
---

## Verifying a deployment

Every deployment, whether a release, a settings change or a key rotation, ends
with the same verification. Its steps go from cheap to thorough, and each one
exercises a link that the previous one did not (Chapter 8).

!!! recipe "Verifying a deployment"
    1. **Versions.** On each service's 部署版本 tab, the new version reads
       正常 with 100% of traffic, and its deployment log ends with the
       application's own "listening" line.
    2. **Chain.** Through the web service's public address: the sign-in page
       answers 200; `/api/health` answers 200; a session request without a
       cookie answers 401 (the curl loop of Chapter 13). The web service runs,
       reaches the API, and the API reaches its database.
    3. **Journey.** With a test account: sign in, create a record, edit it,
       undo the edit, upload an attachment and download it again, delete the
       record and find it in the bin. Each step uses a different part of the
       stack: the gateway's functions, the command history, COS and CORS.
    4. **Email.** Trigger one email, such as a verification code, to a test
       inbox.
    5. **Restart.** Set the API to zero instances and back to one, or deploy
       the same version again: it must start and pass its startup checks with
       no manual step.

    **Check:** all five pass. A failure in step 2 is configuration (an
    address, a key, a variable); a failure that appears only in step 3
    usually points at storage (CORS, the CAM policy) or at a missing
    migration.

The API's startup checks carry much of this weight. An API that verifies at
startup that its key is valid, that the database holds the functions it
needs, and that every email template it needs has an ID, turns most
misconfigurations into a failed version with the reason in its log, instead of
a running service that fails for its first users.

## Watching usage and cost

The package usage page (环境设置 → 套餐用量, route `#/env/package-usage/basic`)
shows the current cycle: points used from the package, from resource packs
and from overage, and a daily breakdown by resource under 用量明细 with the
资源点视图 (resource-point view) [@pricing]. The CLI reports the same figures
with `tcb env usage` [@usage].

For a containerised application the breakdown is dominated by Cloud Run,
whose cost is the number of instance-hours times their size (Section 9.3).
The usage page therefore answers most cost questions through one other page,
each service's instance count:

| Observation | Usual cause | Remedy |
|---|---|---|
| Steady daily cost with no users | A service at a minimum of 1 | Expected for the API; set the web service to 0 if cold starts are acceptable |
| Cost doubles after a release | The previous version still holds instances | Wait for it to drain (an hour or two); delete it if it is not a rollback |
| Cost with a crashed service | A failed version restarting | Delete the failed version |
| Points exhausted mid-cycle | Package too small for always-on instances | Move to a larger package, or buy a resource pack |

Table: Reading a CloudBase usage page.

Usage figures lag by up to two days, and overage is billed the next day when
it is enabled [@pricing], so a change in scaling shows up in the bill only
after a delay. Overage keeps services running when the allowance is spent; it
is the safe setting for an environment that serves users, and the setting to
watch on one that does not.

## Moving an application to another environment

An application moves to another environment when its package no longer fits,
when it changes account structure, or when a staging environment becomes
production. Environments cannot be transferred between accounts
[@createenv], and nothing copies one environment into another, so a move
rebuilds every environment-bound piece of Table 9.2 in the new environment and
reconfigures the account-level ones. The method is that of Chapter 8: prepare
everything without touching the old environment, move the data last in one
short sitting, and keep the old environment as the rollback.

!!! recipe "Preparing the new environment"
    Nothing in this phase changes the old environment, and the steps can be
    spread over days.

    1. **Inventory.** List the old environment's pieces using Table 9.2:
       schema version, data size, services and their variables, API key, the
       bucket in use, CAM users, SES templates, the web address.
    2. **Stop cost on the old one** if it is overrunning: set its API to
       minimum 0 once nobody depends on it, and turn off its automatic
       renewal so that it lapses rather than renews (Chapter 10).
    3. **Schema.** Sign the CLI in, apply the migrations to the new database
       with the revokes, and compare its catalog with the old one's
       (Chapter 12).
    4. **Key.** Create the new environment's server API key (Chapter 10).
    5. **Repository.** Push the release to deploy to the deployment
       repository, and authorise Git in the new environment (Chapter 13).
    6. **Services.** Create the API and web services (Chapter 13). Copy the
       API's variables from the old service, then change everything that
       names the environment: its ID, the bucket, the public web address. Keep
       the API at minimum 0 until the data is in.
    7. **Storage.** Prepare the new bucket with its CORS rule for the new web
       origin, and create a new CAM user and policy for it; put the new keys
       in the new API's variables and probe them (Chapters 11 and 14). Never
       widen the old user's policy.
    8. **Email.** Create templates for the new web origin and wait for their
       review (Chapter 14).
    9. **Rehearse** the data copy against a local database and, as a dry run
       that rolls back, against the new environment (Chapter 12).

    **Check:** the new web service answers its chain check (the API runs and
    reaches the empty database), the storage probe passes, and the templates
    are approved.

!!! recipe "Cutting over"
    One sitting, with the old environment frozen for its duration.

    1. Confirm the new database is still empty. Its web service has been
       public since step 6, and a single sign-up there makes the load refuse
       to run.
    2. Freeze the old environment: both services at a minimum of 0 and no
       instances running. A request may still wake the web service, but with
       the API at zero it cannot write.
    3. Export, load and verify the data (Recipe 12.2). Copy the bucket's
       objects under the same keys if the application keeps attachments in
       the old environment's bucket.
    4. Deploy the new API with the final settings: the new template IDs, the
       new public web address, and a minimum of one instance.
    5. Verify the deployment (Recipe 15.1) with an existing account.
    6. Tell users the new address. Accounts and passwords carry over, but
       every user signs in again, because sessions are tied to the old
       domain; links already sent by email still name the old address.

    **Check:** verification passes with data from before the move.

Until users write to the new environment, the rollback is to restart the old
one: its data is unchanged since the freeze, its keys and templates still
work, and its bucket still holds its objects. After users have written to the
new environment, rolling back loses those writes, so the decision to roll back
belongs in the first hours.

!!! recipe "After the move"
    1. Leave the old environment to lapse. Its isolation period is the last
       rollback window; nothing in it needs deleting by hand.
    2. Update the documents that name the environment: runbooks, deployment
       notes, local configuration files of developers and tools.
    3. Delete the exported data files; they hold password hashes and email
       addresses.
    4. Sign the CLI out (`tcb logout`).
    5. Once the old environment is destroyed, delete its CAM users and their
       policies (Chapter 11), and the templates that only it used.

## Housekeeping

A deployment that runs for months accumulates dates and leftovers. A short
calendar keeps them from turning into outages:

| Item | Where its date is | What happens when it passes |
|---|---|---|
| Environment package | 环境设置 → 套餐用量 | Isolation, then destruction, unless renewed |
| Server API key with an expiry | 环境配置 → API Key | The API refuses to start or loses database access |
| Custom domain certificate | SSL certificate console | Browsers refuse the site |
| Resource packs | 套餐用量 → resources | Remaining points expire after a year |
| CAM keys with a rotation policy | CAM → user → API 密钥 | Nothing automatic; rotate on schedule |

Table: Dates that a CloudBase deployment depends on.

Old versions are deleted once a newer version has proved itself, keeping one
known-good version per service for rollback [@revisions]. Failed versions are
deleted at once. CAM users without a running service behind them are
deleted, and so are templates and CORS origins that no environment uses. A
deployment where every identity, key and rule has a reason to exist is one
whose next move takes a day rather than a week.

!!! summary "Key points"
    - Verify every deployment in five steps: versions, the chain through the
      web address (200, 200, 401), a full user journey, one email, and a
      restart.
    - Cloud Run instance-hours dominate the bill; read the usage page
      together with each service's instance count.
    - A move rebuilds every environment-bound piece in the new environment
      first, moves the data last in one frozen sitting, and keeps the old
      environment as the rollback until it lapses.
    - After a move, users sign in again and old links name the old address;
      exports are deleted and old identities retired once the old
      environment is gone.
    - Keep a calendar of package, key, certificate and resource-pack dates.
