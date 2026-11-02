---
title: CloudBase at a Glance
summary: "What a CloudBase environment contains, how it is billed, and where each part of a deployment lives in the console and the CLI."
references:
  envoverview: "Tencent CloudBase, *云开发环境介绍* (environment overview), [docs.cloudbase.net/quick-start/env-overview](https://docs.cloudbase.net/quick-start/env-overview)."
  createenv: "Tencent CloudBase, *创建云开发环境* (creating an environment), [docs.cloudbase.net/quick-start/create-env](https://docs.cloudbase.net/quick-start/create-env)."
  pricing: "Tencent Cloud, *云开发 CloudBase 计费说明* (billing), [cloud.tencent.com/document/product/876/127357](https://cloud.tencent.com/document/product/876/127357)."
  cliinstall: "Tencent CloudBase, *CLI 安装* (installing the CLI), [docs.cloudbase.net/cli-v1/install](https://docs.cloudbase.net/cli-v1/install)."
  clilogin: "Tencent CloudBase, *CLI 登录认证* (signing the CLI in), [docs.cloudbase.net/cli-v1/login](https://docs.cloudbase.net/cli-v1/login)."
  camsyntax: "Tencent Cloud, *CAM 策略语法* (policy syntax), [cloud.tencent.com/document/product/598/10603](https://cloud.tencent.com/document/product/598/10603)."
---

## Environments and what they hold

Tencent CloudBase (云开发) is a backend platform built from Tencent Cloud
services and sold per **environment** (环境). The environment is both the unit
of isolation and the unit of billing: it holds its own database, storage,
compute and authentication, and nothing in one environment can see the data
of another [@envoverview]. A project typically has one environment per stage
of its release process (Chapter 8), and moving an application between
environments means recreating every environment-bound piece (Chapter 15).

!!! definition "Environment"
    A CloudBase environment is a named, separately billed bundle of a
    PostgreSQL database, a storage bucket, Cloud Run services, cloud
    functions, static hosting, an HTTP gateway and an authentication
    service. Its **environment ID** (环境 ID), of the form
    `<name>-<suffix>`, is globally unique and is what every SDK, CLI command
    and gateway address names.

Environments created with a PostgreSQL database run in **PG mode**, in which
the database, authentication and storage share one SQL permission model.
Older environments run in a traditional mode built around a document
database; a mode cannot be changed after creation [@envoverview]. Everything
in this part assumes PG mode. Table 9.1 lists what an environment contains and
which chapter of this part sets each piece up.

| Component | Console name | What the application uses it for | Chapter |
|---|---|---|---|
| PostgreSQL database | 数据库 | Schema, data and SQL functions, reached through the HTTP gateway | 12 |
| Cloud Run | 云托管 | The API and web services as containers | 13 |
| Storage bucket | 云存储 | Attachments, through signed URLs | 14 |
| HTTP gateway | HTTP 网关 | Database REST and RPC calls; custom domains for services | 12, 13 |
| Authentication | 身份认证 | Optional; an application may run its own accounts instead | — |
| Server API key | API Key | The credential the API uses to call the gateway | 10 |

Table: The parts of a PG-mode environment that a containerised web
application uses. Cloud functions and static hosting exist as well but are not
needed when both services run on Cloud Run.

## What lives outside the environment

Several pieces of a deployment belong to the Tencent Cloud **account** rather
than to any environment. They survive when an environment is destroyed, they
can serve several environments at once, and they are managed in their own
consoles. Telling the two kinds apart is the first step of any move between
environments, because the environment-bound pieces must be recreated and the
account-level ones only reconfigured.

| Piece | Belongs to | Console |
|---|---|---|
| Database, its roles and data | Environment | CloudBase |
| Cloud Run services, their versions, variables and default domains | Environment | CloudBase |
| Server API keys | Environment | CloudBase |
| The environment's own storage bucket | Environment (destroyed with it) | CloudBase, COS |
| Git authorisation for Cloud Run | Environment (granted once per environment) | CloudBase |
| CAM users, policies and their keys | Account | CAM |
| Standalone COS buckets | Account | COS |
| SES sender domains, sender addresses and templates | Account | SES |
| Domain names, DNS records, certificates | Account | DNSPod, SSL certificates |

Table: Environment-bound and account-level pieces of a CloudBase deployment.

![A CloudBase deployment. Everything inside the environment frame is created per environment and destroyed with it; the account-level services on the right outlive it. Solid arrows are requests; the dashed arrow is the browser's direct transfer with signed URLs.](figures/deployment.svg)

Figure 9.1 shows how the two kinds connect. The web and API services, the
gateway, the database and the environment's bucket are all inside the
environment; the keys the API uses for storage and email, the email service
itself and the domain names are outside it.

!!! warning "The environment's bucket is not account-level"
    Each environment comes with a COS bucket named
    `<prefix>-<env-id>-<appid>`, listed in the COS console with the tag
    云开发桶 (CloudBase bucket). It looks like any other bucket, but it is
    part of the environment and is destroyed with it. An application that
    keeps its attachments there must copy them out before the environment
    lapses (Chapter 14).

CAM, Tencent Cloud's access management, is account-wide. Its policies name
resources by a six-part description that includes the account's **APPID**,
the number that also ends every bucket name [@camsyntax]. The account's
other identifier, the **UIN**, names users and the root account. Chapter 11
uses both.

The account's verification level also matters. An account under individual
real-name verification can create environments and services like any other,
but some account-level services restrict it: Tencent SES, for example, sends
such an account's email only from reviewed templates and offers it no SMTP
(Chapter 14).

## Packages and resource points

An environment is bought as a monthly **package** (套餐) that includes an
allowance of **resource points** (资源点). Every billed resource, from database
calls to Cloud Run CPU time, is converted to points at a published rate, and
1,000 points cost 1 CNY. Table 9.3 lists the packages on sale in 2026
[@pricing].

| Package | Price per month | Points per month |
|---|---|---|
| 免费体验版 (free trial) | 0 | 3,000 |
| 个人版 (personal) | 19.9 CNY (promotional) | 40,000 |
| 标准版 (standard) | 199 CNY | 330,000 |
| 企业版 (enterprise) | 999 CNY | 1,500,000 |
| 企业高级版 (enterprise advanced) | on request | 6,500,000 |

Table: CloudBase packages and their resource-point allowances.

Points are drawn first from the package, then from any **resource packs**
(资源包, bought separately and valid for one year), and finally from
pay-as-you-go overage (按量付费) if it is enabled. With overage enabled, use
beyond the allowance is billed the next day; a free-trial environment cannot
enable it at all [@pricing]. Usage statistics lag by up to two days, so the
usage page is a trailing indicator.

For a containerised application, Cloud Run dominates the bill. In 2026 the
usage page itemised Cloud Run at 55 points per core-hour of CPU and 32 points
per GiB-hour of memory, so one instance of 1 core and 2 GiB that never scales
to zero costs 119 points an hour, about 2,860 a day and 86,000 a month. That
single figure decides the package: an always-on API exceeds the personal
package's 40,000 points by itself, while the standard package covers it with
room for a web service that scales to zero.

!!! example "Sizing a package"
    An API kept at one warm instance (1 core, 2 GiB) uses about 86,000 points
    a month. A web service of the same size that runs only while people use
    it, say six hours a day, adds about 21,000. Together they fit the
    standard package's 330,000 points. On the personal package the same
    deployment overruns its allowance in the second week and continues on
    overage at about 2.9 CNY a day per always-on instance.

A package runs for a fixed term and can renew automatically. An environment
whose package expires is isolated (停服隔离): its services stop and its data
becomes unreachable, and if it is not renewed within the isolation period it
is destroyed with everything in it [@pricing]. Chapter 10 covers renewal and
turning an environment off.

## The console and the CLI

The CloudBase console lives at `https://tcb.cloud.tencent.com/dev` and signs
in with a Tencent Cloud account, by default through a WeChat QR code
[@createenv]. A direct load of that address sometimes stays blank until the
application has booted once in the tab; opening it from the Tencent Cloud
console (`https://console.cloud.tencent.com/tcb`, then the environment, then
前往新版开发平台) always works. Once loaded, its pages are addressed by hash
routes that can be bookmarked per environment.

| Page | Route after `/dev?envId=<env-id>` |
|---|---|
| Cloud Run services | `#/platform-run` |
| One service (versions, settings, logs) | `#/platform-run/service/detail?serverName=<service>` |
| PostgreSQL SQL editor | `#/db/postgres/sql-editor` |
| Package usage | `#/env/package-usage/basic` |

Table: Console pages used throughout this part, as routed in November 2026.

The **CloudBase CLI** (`@cloudbase/cli`, command `tcb`) does from a terminal
what the console does by hand, and it is the better tool for anything that
must be repeated or checked: running SQL against an environment, listing and
creating API keys, deploying and rolling back Cloud Run versions
[@cliinstall]. It needs Node.js and signs in through a browser authorisation
that is stored on the machine [@clilogin].

!!! recipe "Installing and signing in the CLI"
    ```bash
    npm install -g @cloudbase/cli
    tcb login          # opens a browser to authorise this machine
    tcb env list       # confirms the session: lists the account's environments
    ```

    The stored authorisation can act on every environment of the account.
    Sign out with `tcb logout` when the task that needed it is done, and never
    copy the stored credentials file anywhere.

Both tools act with the account's full rights. The application itself never
uses either: it acts through the narrower credentials of Chapters 10 and 11.

!!! summary "Key points"
    - An environment is CloudBase's unit of isolation and billing; it holds
      the database, Cloud Run services, a storage bucket, the HTTP gateway
      and its API keys. Use PG-mode environments.
    - CAM identities, standalone buckets, SES and DNS belong to the account
      and outlive environments; the environment's own bucket does not.
    - Packages are allowances of resource points (1,000 points = 1 CNY).
      One always-on 1-core, 2 GiB Cloud Run instance costs about 86,000
      points a month, which by itself decides between the personal and the
      standard package.
    - An expired package isolates the environment and, if not renewed in
      time, destroys it.
    - The console is addressed by hash routes per environment; the `tcb` CLI
      repeats console actions from a terminal and should be signed out when
      not in use.
