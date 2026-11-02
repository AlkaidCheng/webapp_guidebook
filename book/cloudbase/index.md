---
title: Tencent CloudBase
summary: "Hosting a containerised web application on Tencent CloudBase: environments and keys, CAM identities, the PostgreSQL database, Cloud Run services, storage, email and day-to-day operation."
---

Tencent CloudBase (云开发) bundles the pieces of Part I into one product for the
mainland-China market. An **environment** (环境) holds a PostgreSQL database, a
storage bucket and any number of **Cloud Run** (云托管) services, and is billed
as a package of resource points. Identities and keys that reach beyond one
environment, such as the users that read a bucket or send email, live in
Tencent Cloud's account-wide access management, **CAM**.

The chapters in this part follow the order in which a new project is set up:
first the environment and its key, then the CAM identities, the database
schema, the two services, storage and email, and finally the procedures that
keep the deployment running and move it when it has to move. Each procedure
names the console path, the values to enter, and the check that confirms it
worked. The console is in Chinese; its labels are quoted as they appear, with
an English gloss on first use.

The procedures were last checked against the console in November 2026, while
moving a running application from one environment to another.
