---
title: Environments and Releases
summary: "Moving code from a commit to users: environments, deployment repositories, versions and rollback, verification, and moving a deployment."
references:
  twelve-codebase: "A. Wiggins, *The Twelve-Factor App: I. Codebase*, <https://12factor.net/codebase>."
  twelve-dev-prod: "A. Wiggins, *The Twelve-Factor App: X. Dev/prod parity*, <https://12factor.net/dev-prod-parity>."
  twelve-config: "A. Wiggins, *The Twelve-Factor App: III. Config*, <https://12factor.net/config>."
  gh-oauth-scopes: "GitHub, *Scopes for OAuth apps*, GitHub Docs, <https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps>."
  gh-deploy-keys: "GitHub, *Managing deploy keys*, GitHub Docs, <https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys>."
  gh-actions: "GitHub, *Managing GitHub Actions settings for a repository*, GitHub Docs, <https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository>."
  gh-dependabot: "GitHub, *Configuring Dependabot security updates*, GitHub Docs, <https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configure-security-updates>."
  git-push: "Git, *git-push*, Git reference manual, section Note about fast-forwards, <https://git-scm.com/docs/git-push>."
  twelve-build: "A. Wiggins, *The Twelve-Factor App: V. Build, release, run*, <https://12factor.net/build-release-run>."
  run-rollouts: "Google Cloud, *Rollbacks, gradual rollouts, and traffic migration*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration>."
---

## Environments

A deployment exists more than once. The application runs on developers'
machines, in a staging environment where releases are tried, and in production
where people use it. Each is an *environment* in the sense of
[Chapter 1](anatomy.md): a complete copy with its own services, database,
bucket, identities, keys and domain. All of them run from the same codebase
[@twelve-codebase] and differ only in configuration [@twelve-config], which is
what makes a release verified in staging meaningful for production.

| Property | Local | Staging | Production |
|---|---|---|---|
| Purpose | Development and tests | Trying a release on real infrastructure | Serving users |
| Data | Disposable fixtures | Test accounts and data | Users' data |
| Origin | `http://localhost:<port>` | `https://<staging-origin>` | `https://<web-origin>` |
| Identities and keys | Developer's own or none | Staging's own ([Chapter 3](access.md)) | Production's own |
| Email | Written to the log | Real delivery, test recipients | Real delivery |
| Receives a release | On every change | Before production | After staging passes |

Table: The usual environments of a web application.

The gap between environments is a source of failures that only production
shows. The twelve-factor guidance names three gaps to keep small: time
between writing code and deploying it, the people who write and deploy it,
and the tools and backing services used in each place [@twelve-dev-prod].
The last is the one an infrastructure setup controls: staging runs the same
database engine and version, the same kind of bucket with a CORS rule for its
own origin, real email through its own sender identity, and the same platform
settings as production, with smaller sizes.

## The deployment repository

A platform that builds images from Git ([Chapter 5](services.md)) needs read
access to a repository, and the usual way to grant it, an OAuth authorization
of the platform by a Git hosting account, is broad. On GitHub, the `repo` scope
that private repositories require grants full read and write access to every
public and private repository the account can reach [@gh-oauth-scopes]. A
developer's account authorized this way hands the platform all of that
developer's code.

!!! definition "Deployment repository"
    A private mirror of the application's main branch, owned by an account
    that exists only for this purpose. The platform is authorized through
    that account, and the account owns nothing but the mirror, so the
    authorization covers nothing else. GitHub's documentation describes such
    machine users as the way to give automation an account of its own
    [@gh-deploy-keys].

The mirror receives commits in one way only: a push of the main branch, after
that commit's checks have passed, as a fast-forward. A fast-forward update
moves the branch to a descendant of its current commit and loses no history,
and `git push` refuses any other update unless forced [@git-push]. The mirror
therefore always holds a prefix of the main branch's history, and a commit
that never passed the checks never reaches it.

The push is made with a *deploy key*: an SSH key attached to the mirror alone,
with write access enabled, since deploy keys are read-only by default
[@gh-deploy-keys]. A leaked deploy key can push to the mirror and nothing
else. Like every key without expiry it is listed in the identity inventory and
rotated on a schedule ([Chapter 3](access.md)).

Everything else on the mirror is switched off. Its workflows are copies of the
main repository's, and running them again on the mirror would duplicate CI and
need secrets there, so GitHub Actions is disabled for it [@gh-actions];
Dependabot security updates are disabled as well [@gh-dependabot]. A pull
request opened on the mirror by any automation is closed unmerged, since the
mirror accepts only fast-forwards of the main branch.

!!! recipe "Releasing to the deployment repository"
    One-time setup: an SSH host alias that selects the deploy key, and a
    remote that uses it.

    ```text
    # ~/.ssh/config
    Host deploy-mirror
      HostName github.com
      User git
      IdentityFile ~/.ssh/<deploy-key>
      IdentitiesOnly yes
    ```

    ```bash
    git remote add deploy deploy-mirror:<mirror-owner>/<repository>.git
    ```

    Each release, from an up-to-date main branch whose checks have passed:

    ```bash
    git switch main
    git pull --ff-only origin main
    git push deploy main:main
    ```

    `git push` refuses with a "non-fast-forward" error if the mirror holds a
    commit that main does not. That means someone pushed to the mirror
    directly; investigate before doing anything else, and never resolve it
    with `--force`.

## Release order

A release combines a build with one environment's configuration
[@twelve-build], and in a deployment with a database it has a fixed order:
the schema first, then the code that needs it. The new code expects the new
schema, so it cannot start first; the old code keeps running while the
migrations apply, so the migrations must not break it.

The convention that satisfies both is to split schema changes into an
*expanding* part and a *contracting* part. The expanding migrations add
tables, columns and functions that the old code ignores, and run before the
new code starts. The contracting migrations remove what the old code needed,
and run in a later release, once no deployed version, including the one kept
for rollback, still uses it.

!!! recipe "The order of one release"
    1. Apply the release's migrations to the environment's database
       ([Chapter 4](databases.md)), stopping writers first if any migration
       changes an existing table. Reload the gateway's schema cache if the
       database is reached through one.
    2. Deploy the new API version. It must pass its startup check and health
       probe before it receives traffic.
    3. Deploy the new web version, which may call API routes added in step 2.
    4. Verify (Section 8.4).
    5. Keep the previous versions until the next release has been verified;
       delete failed versions.

Rolling back reverses steps 3 and 2 by routing traffic to the previous
versions, which takes effect at once [@run-rollouts]. Step 1 is not reversed:
the expanded schema stays, which the previous code tolerates by construction.
A migration that cannot be made compatible with the previous code turns a
rollback into a restore, and is scheduled with the writers stopped and a
verified export taken first.

## Verification after every deploy

A deploy is finished when it has been verified, not when the platform reports
success. The platform knows that a container started and answered a probe; it
does not know whether the forwarding from web to API works, whether
authentication is enforced, or whether an upload reaches the bucket.
Verification is a fixed list, the same after every release, run through the
public origin so that every layer of Figure 1.1 is crossed.

!!! recipe "Post-deploy verification"
    1. The health endpoint answers `200` through the public origin:

        ```bash
        curl -sS -o /dev/null -w '%{http_code}\n' https://<web-origin>/api/health
        ```

    2. An unauthenticated request for the current session answers `401`:

        ```bash
        curl -sS -o /dev/null -w '%{http_code}\n' https://<web-origin>/api/session
        ```

        A `401` proves that the request reached the API and that the API
        refuses anonymous callers. A `200` means authentication is broken; a
        `502` or `504` means the API is unreachable from the web service.

    3. One complete user journey in a browser: sign in, create a record,
       upload a file and download it again, trigger an email and receive it,
       sign out. Each step crosses a different part: the session cookie, the
       database, the signed URLs and the bucket's CORS rule, the email
       template and sender.
    4. Record the release: date, commit, the versions now serving, which
       checks passed, and what was not checked, such as a physical device or a
       mail provider not tried.

The first two checks take seconds and can be automated; the journey is worth
its few minutes, because it is the only check that exercises the
configuration a release can break silently, such as an origin missing from a
CORS rule or a template not yet approved.

## Moving a deployment

Sometimes a whole deployment has to move: to another environment of the same
platform, because a package or region must change, or to another provider.
The move is a release of everything at once, and it succeeds when it is
planned as an ordered list in which data comes last.

The first step is an inventory of what the deployment consists of, divided by
whether it belongs to the environment, and so must be recreated, or to the
account, and so stays (Table 8.2).

| Item | Belongs to | In the move |
|---|---|---|
| Services, their versions and variables | Environment | Redeployed from the deployment repository; variables set anew |
| Internal addresses, default domains | Environment | Change; every variable naming them is updated |
| Database | Environment | Schema by migrations; data exported, loaded and verified ([Chapter 4](databases.md)) |
| Platform-provided bucket | Environment | Objects copied under the same keys ([Chapter 6](storage.md)) |
| Platform API keys | Environment | Issued anew |
| Access identities for storage and email | Account | New per-environment identities created; old ones retired ([Chapter 3](access.md)) |
| Standalone bucket | Account | Stays; its CORS rule gains the new origin |
| Email sender domain and templates | Account | Domain stays; templates that embed a new origin need new versions ([Chapter 7](email.md)) |
| Custom domain and DNS records | Registrar or DNS provider | Switched at the end |
| Deployment repository | Git host | Unchanged; the new services are connected to it |

Table: Inventory for moving a deployment.

!!! recipe "Moving a deployment, in order"
    1. Write the inventory of Table 8.2 for the deployment at hand.
    2. Create the new environment, its identities and keys, and record them
       in the password manager.
    3. Apply the migrations to the new, empty database. Deploy the services
       from the deployment repository with the new configuration, and run the
       first two verification checks, which write nothing.
    4. Submit the templates the new origin needs; set the bucket's CORS rule
       for the new origin; prepare the bucket or the object copy.
    5. Freeze writes in the old environment by stopping its API service or
       switching it to a read-only mode.
    6. Export the data, load it into the new database, verify it; copy and
       check the objects.
    7. Switch users to the new environment: change DNS, or move the custom
       domain, or announce the new origin.
    8. Run the full verification, including the user journey, in the new
       environment.
    9. Keep the old environment intact and frozen as the rollback until its
       term lapses, then delete it, its identities and its keys, and the
       exports.

The order has reasons. Everything with a lead time, identities, deployments
and template reviews, comes before the freeze, so that the freeze lasts only
as long as the copy. The user journey waits until after the load, because it
writes data, and a load into a database that already holds rows refuses to run
(Section 4.4). And the old environment stays untouched after the switch: as
long as it exists, switching back is a configuration change. Writes made in the
new environment after the switch would not be in the old one, so the decision
to roll back is taken early, within a window agreed before the move. A move
that changes the origin also signs every user out and strands old links
([Chapter 2](domains.md)), which belongs in the announcement.

[Chapter 15](../cloudbase/operations.md) carries out this procedure between two
CloudBase environments, and records the verification and the cost checks that
follow a release there.

!!! summary "Key points"
    - Environments are complete copies that differ only in configuration;
      keep staging's backing services the same kind as production's.
    - Let the platform build from a private deployment repository owned by a
      dedicated account, pushed by fast-forward from a checked main branch
      with a deploy key, with Actions and Dependabot off.
    - Release in order: expanding migrations, API, web, verification.
      Contracting migrations wait for a later release; rollback switches
      traffic back and leaves the schema.
    - Verify every deploy through the public origin: health `200`, an
      unauthenticated session request `401`, then one full user journey.
      Record what was and was not checked.
    - Move a deployment from an inventory: identities, services and
      templates first, data last, behind a write freeze; keep the old
      environment as the rollback until it lapses, then delete its
      identities and keys.
