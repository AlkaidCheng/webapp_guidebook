---
title: Domains, Origins and HTTPS
summary: "How names reach a deployment: DNS records, certificates, origins, cookies and the browser rules that follow from them."
references:
  rfc1034: "P. Mockapetris, *Domain names: concepts and facilities*, RFC 1034 (1987), <https://www.rfc-editor.org/rfc/rfc1034>."
  rfc1035: "P. Mockapetris, *Domain names: implementation and specification*, RFC 1035 (1987), <https://www.rfc-editor.org/rfc/rfc1035>."
  rfc3596: "S. Thomson, C. Huitema, V. Ksinant and M. Souissi, *DNS Extensions to Support IP Version 6*, RFC 3596 (2003), <https://www.rfc-editor.org/rfc/rfc3596>."
  rfc8659: "P. Hallam-Baker, R. Stradling and J. Hoffman-Andrews, *DNS Certification Authority Authorization (CAA) Resource Record*, RFC 8659 (2019), <https://www.rfc-editor.org/rfc/rfc8659>."
  cf-flattening: "Cloudflare, *CNAME flattening*, Cloudflare DNS documentation, <https://developers.cloudflare.com/dns/cname-flattening/>."
  run-domains: "Google Cloud, *Mapping custom domains*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/mapping-custom-domains>."
  rfc8555: "R. Barnes, J. Hoffman-Andrews, D. McCarney and J. Kasten, *Automatic Certificate Management Environment (ACME)*, RFC 8555 (2019), <https://www.rfc-editor.org/rfc/rfc8555>."
  le-challenges: "Let's Encrypt, *Challenge Types*, <https://letsencrypt.org/docs/challenge-types/>."
  rfc6797: "J. Hodges, C. Jackson and A. Barth, *HTTP Strict Transport Security (HSTS)*, RFC 6797 (2012), <https://www.rfc-editor.org/rfc/rfc6797>."
  mdn-hsts: "MDN Web Docs, *Strict-Transport-Security header*, <https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security>."
  icp-overview: "Tencent Cloud, *ICP 备案：备案概述* (ICP filing overview), <https://cloud.tencent.com/document/product/243/18907>."
  icp-review: "Tencent Cloud, *ICP 备案：备案审核* (Filing review), <https://cloud.tencent.com/document/product/243/19650>."
  wx-network: "WeChat Open Platform, *小程序网络：使用说明* (Mini Program networking), <https://developers.weixin.qq.com/miniprogram/dev/framework/ability/network.html>."
  wx-cloudrun: "WeChat Open Platform, *微信云托管开发常识* (WeChat Cloud Run essentials), <https://developers.weixin.qq.com/minigame/dev/wxcloudrun/src/guide/debug/know.md>."
  rfc6454: "A. Barth, *The Web Origin Concept*, RFC 6454 (2011), <https://www.rfc-editor.org/rfc/rfc6454>."
  mdn-sop: "MDN Web Docs, *Same-origin policy*, <https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy>."
  rfc6265: "A. Barth, *HTTP State Management Mechanism*, RFC 6265 (2011), <https://www.rfc-editor.org/rfc/rfc6265>."
  mdn-set-cookie: "MDN Web Docs, *Set-Cookie header*, <https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie>."
  rfc9110: "R. Fielding, M. Nottingham and J. Reschke (eds.), *HTTP Semantics*, RFC 9110 (2022), <https://www.rfc-editor.org/rfc/rfc9110>."
  mdn-cors: "MDN Web Docs, *Cross-Origin Resource Sharing (CORS)*, <https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS>."
  fetch: "WHATWG, *Fetch Standard*, Living Standard, section CORS protocol, <https://fetch.spec.whatwg.org/>."
  mdn-csp: "MDN Web Docs, *Content Security Policy (CSP)*, <https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP>."
  csp3: "W3C, *Content Security Policy Level 3*, Working Draft, <https://www.w3.org/TR/CSP3/>."
  owasp-csp: "OWASP Cheat Sheet Series, *Content Security Policy Cheat Sheet*, <https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html>."
---

## Names and records

A browser reaches a deployment by name. The Domain Name System maps a name such
as `www.example.com` to the data needed to reach it, through a hierarchy of
zones each served by its own authoritative name servers. Every answer carries a
time to live (TTL), the number of seconds for which resolvers may cache it
[@rfc1034]. The TTL is what makes DNS changes slow: after a record is edited,
clients keep using the cached old answer until its TTL expires. Lowering the
TTL a day before a planned change, and raising it again afterwards, shortens
the window in which some users see the old address.

A deployment uses a handful of record types (Table 2.1). Each is stored under a
name in the zone, and the same name may hold several types at once, with one
exception: a name that holds a CNAME holds nothing else [@rfc1034].

| Type | Maps a name to | Used for |
|---|---|---|
| A | An IPv4 address [@rfc1035] | Pointing a name at a platform's IPv4 address |
| AAAA | An IPv6 address [@rfc3596] | The same for IPv6 |
| CNAME | Another name, of which this one is an alias [@rfc1034] | Pointing a subdomain at a platform's default domain |
| TXT | Free text [@rfc1035] | Ownership verification; SPF, DKIM and DMARC ([Chapter 7](email.md)) |
| MX | The hosts that accept mail for the name, with preferences [@rfc1035] | Mailboxes; bounce handling for a sender domain |
| CAA | The certificate authorities allowed to issue for the name [@rfc8659] | Restricting certificate issuance |

Table: DNS record types that a web deployment uses.

The CNAME exception has a practical consequence. The apex of a zone
(`example.com` itself) always holds the zone's SOA and NS records, so it cannot
also hold a CNAME, and a platform that asks for a CNAME can be given only a
subdomain such as `www.example.com`. DNS providers work around this with
non-standard record types that resolve the target at query time and answer
with its addresses, sold as ALIAS records or *CNAME flattening*
[@cf-flattening]; the alternative is to point the apex at the platform's fixed
A and AAAA addresses and redirect it to `www`.

!!! recipe "Inspecting a domain's records"
    `dig` asks a resolver for one record type at one name. `+short` prints
    only the answers; naming a server with `@` bypasses caches by asking that
    server directly.

    ```bash
    dig +short A     www.example.com
    dig +short AAAA  www.example.com
    dig +short CNAME www.example.com
    dig +short TXT   example.com
    dig +short MX    mail.example.com
    dig +short CAA   example.com
    dig +short NS    example.com                 # the zone's authoritative servers
    dig +short A www.example.com @<authoritative-server>
    ```

    When an authoritative server already returns the new value but a public
    resolver still returns the old one, the change is complete and only the
    TTL remains.

## Default and custom domains

Hosting platforms give each deployed service a *default domain*, a
generated name under the platform's own domain, such as a URL ending in
`run.app` on Google Cloud Run [@run-domains]. It works the moment the service
starts and comes with a certificate, which makes it the natural address for
testing. It is not an address to give to users. It belongs to the platform, it
changes when the service is recreated or moved to another environment, and it
may carry restrictions. WeChat's container hosting, for example, describes its
default public domain as suitable only for interface testing, not for
production, and does not allow it to be configured as a Mini Program server
domain [@wx-cloudrun].

A *custom domain* is one the application's owner registers and controls. Adding
it to a platform takes three steps: proving control of the domain, usually
with a TXT record or a file the platform names; adding the A, AAAA or CNAME
records the platform specifies; and waiting for the platform to obtain a
certificate, which on Cloud Run usually takes about 15 minutes and can take up
to 24 hours [@run-domains].

### Certificates

HTTPS requires a certificate for the name, issued by a certificate authority
that browsers trust. For a web application this is a *domain-validated*
certificate.

!!! definition "Domain-validated certificate"
    A certificate whose issuer has checked only that the applicant controls
    the domain, not who the applicant is. Control is shown by answering a
    challenge from the authority: serving a token over HTTP on port 80 under
    `/.well-known/acme-challenge/`, publishing a TXT record at
    `_acme-challenge.<domain>`, or answering a special TLS handshake on
    port 443 [@le-challenges]. Only the DNS challenge can validate a wildcard
    name such as `*.example.com`.

The ACME protocol automates the exchange: a client on the server, or the
platform itself, requests a certificate, answers the challenge and installs the
result, and repeats the process before expiry [@rfc8555]. Let's Encrypt is the
best-known authority that issues certificates this way, and managed platforms
run the same protocol on the owner's behalf. Two things stop it. A CAA record
on the domain that does not name the platform's authority forbids issuance
[@rfc8659], which leaves a newly added custom domain without a certificate; and
an HTTP challenge cannot succeed while the name still points elsewhere.

Once a site is reachable over HTTPS, the `Strict-Transport-Security` response
header tells browsers to use HTTPS for that host, without first trying plain
HTTP, for the number of seconds given in `max-age` [@rfc6797]. With
`includeSubDomains` the rule covers every subdomain as well [@mdn-hsts].

!!! remark "HSTS cannot be withdrawn quickly"
    A browser that has seen the header keeps enforcing it until `max-age`
    expires, whatever the server sends later. Starting with a short `max-age`
    and adding `includeSubDomains` only once every subdomain serves HTTPS avoids
    locking users out of a host that is not ready.

### ICP filing in mainland China

A site served to users in mainland China from a server in a mainland region
needs an ICP filing (ICP 备案): it may be opened for access only after the
communications administration has issued a filing number, and Tencent Cloud,
for one, blocks an unfiled domain that points at one of its mainland servers
[@icp-overview]. The filing takes weeks, not hours: the provider's own check
takes one to two working days, and the provincial administration's review
takes up to twenty working days more, excluding weekends and public holidays
[@icp-review]. Other channels into the mainland market impose the same
condition. A WeChat Mini Program may call only server domains that use HTTPS
and are ICP-filed, never an IP address or `localhost` [@wx-network].

!!! warning "Start the filing before the code is ready"
    The filing, not the application, is usually the last thing standing
    between a mainland deployment and its users. A domain registered and filed
    at the start of a project costs a few weeks of waiting that run in
    parallel with development; filed at the end, the same weeks delay the
    launch. The platform of Part II serves the mainland market, so its custom
    domains fall under this rule ([Chapter 9](../cloudbase/overview.md)).

## Origins and cookies

The browser's security rules are written in terms of origins, not domains.

!!! definition "Origin"
    The triple of scheme, host and port of a URL [@rfc6454]. Two URLs have the
    same origin only if all three match: `https://example.com`,
    `https://www.example.com`, `http://example.com` and
    `https://example.com:8443` are four different origins. The same-origin
    policy lets a script read responses only from its own origin unless the
    other origin allows it [@mdn-sop].

Cookies follow an older and looser rule. A cookie set without a `Domain`
attribute is *host-only*: the browser returns it only to the exact host that
set it [@rfc6265]. With `Domain=example.com` it is returned to that host and
all of its subdomains, and in neither case are cookies isolated by port. The
`__Host-` name prefix makes the browser reject a cookie unless it is `Secure`,
has `Path=/` and carries no `Domain` attribute, which pins a session cookie to
one host [@mdn-set-cookie].

Host scoping is what makes a change of domain disruptive. Sessions are cookies
on the old host, and the browser will not send them to the new one, so moving
an application to a new domain signs every user out. Links already sent, such
as invitation emails, password-reset links and bookmarks, still name the old
domain. The old domain therefore has to keep answering, with a permanent
redirect to the same path on the new origin; `308 Permanent Redirect` keeps the
request method, where `301` allows clients to change a `POST` into a `GET`
[@rfc9110].

!!! recipe "Changing an application's origin"
    1. Add the new domain to the platform and wait until its certificate is
       issued.
    2. Change every configuration value that names the origin, on every
       service (`WEB_ORIGIN`, `PUBLIC_ORIGIN` in the inventory of
       [Chapter 1](anatomy.md)), and release.
    3. Add the new origin to the bucket's CORS rule
       ([Chapter 6](storage.md)), keeping the old one until the move is
       complete.
    4. Prepare new versions of email templates that embed the origin, early
       enough for any review ([Chapter 7](email.md)).
    5. Update every allowlist kept outside the application: OAuth redirect
       URIs, Mini Program server domains, a CSP that names the origin.
    6. Point the old domain at a redirect to the new origin that keeps the
       path, and leave it running for as long as old links may be followed.
    7. Tell users in advance that they will have to sign in again.

## Cross-origin requests

When a page needs to read a response from another origin, the other origin
grants permission through response headers defined by the CORS protocol
[@fetch; @mdn-cors]. A request that a plain HTML form could also have sent
(`GET`, `HEAD` or `POST`, with a limited set of headers and content types) goes
out directly, and the browser hides the response from the script unless it
carries a matching `Access-Control-Allow-Origin`. Any other request is first
*preflighted*: the browser sends an `OPTIONS` request describing the request
it intends to make and sends the real request only if the answer allows it.

!!! example "A preflight and its answer"
    A script on `https://www.example.com` wants to `PUT` a JSON body with a
    custom header to `https://api.example.com`. The browser first sends:

    ```http
    OPTIONS /items/42 HTTP/1.1
    Host: api.example.com
    Origin: https://www.example.com
    Access-Control-Request-Method: PUT
    Access-Control-Request-Headers: content-type, x-request-id
    ```

    and proceeds only if the response names the origin, the method and both
    headers:

    ```http
    HTTP/1.1 204 No Content
    Access-Control-Allow-Origin: https://www.example.com
    Access-Control-Allow-Methods: GET, PUT
    Access-Control-Allow-Headers: content-type, x-request-id
    Access-Control-Max-Age: 600
    Vary: Origin
    ```

    `Access-Control-Max-Age` lets the browser cache the answer for that many
    seconds. Response headers beyond a small safe set are visible to the script
    only if listed in `Access-Control-Expose-Headers` on the real response.

Two rules cause most CORS failures. A request that carries cookies or other
credentials is allowed only if the response names the exact origin and adds
`Access-Control-Allow-Credentials: true`; the wildcard `*` is not accepted for
it [@mdn-cors]. And CORS is enforced by the browser alone. It protects users
of a browser from scripts on other sites, not the server from other clients:
`curl` ignores it, so it never replaces authentication. An application built on
the same-origin pattern of [Chapter 1](anatomy.md) needs CORS only for direct
transfers to object storage.

## Content Security Policy

A Content Security Policy (CSP) is a response header listing the sources from
which a page may load scripts, styles, images, frames and connections
[@mdn-csp; @csp3]. Its main purpose is to make injected script inert: if an
attacker manages to place a `<script>` element in a page, the browser refuses
to run it because it does not match the policy.

The strongest practical form is a *nonce-based* policy. For every response the
server generates a fresh random value, sends it in the header, and adds it as a
`nonce` attribute to each script element it emits; the browser runs only
scripts that carry it. The nonce must differ for every response and must not be
predictable, so a page that carries one cannot be served as static HTML
[@mdn-csp]. The `'strict-dynamic'` keyword extends trust from a script that
carries the nonce to the scripts it loads, which lets bundlers split code into
chunks without listing them [@mdn-csp; @owasp-csp].

```http
Content-Security-Policy: default-src 'self'; script-src 'nonce-<random>' 'strict-dynamic'; connect-src 'self' https://<storage-endpoint>; img-src 'self' data: https://<storage-endpoint>; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
```

Each directive has a reason. `connect-src` governs `fetch` targets, so a
direct upload to object storage fails unless the storage endpoint is listed
there; `img-src` does the same for images shown through signed URLs;
`object-src 'none'` and `base-uri 'none'` close two old injection routes; and
`frame-ancestors 'none'` forbids other sites from framing the application. In a
same-origin deployment the web service generates the nonce, since it produces
the pages.

!!! tip "Introduce a policy in report-only mode"
    The header `Content-Security-Policy-Report-Only` takes the same policy but
    only reports violations instead of blocking them [@mdn-csp]. Running a new
    policy in that mode against real traffic first shows which legitimate
    loads it would break.

In Part II, [Chapter 13](../cloudbase/cloud-run.md) deploys the services that a
domain points at, and [Chapter 15](../cloudbase/operations.md) moves an
application between environments, with the change of origin that a move can
bring.

!!! summary "Key points"
    - DNS answers are cached for their TTL; lower it before a planned change.
      A name with a CNAME holds no other record, so the zone apex needs A and
      AAAA records or a provider's flattening.
    - Default platform domains are for testing; some platforms forbid their
      production use. Custom domains need ownership proof, records and a
      certificate, which ACME issues automatically unless a CAA record forbids
      it.
    - In mainland China a site served from a mainland region needs an ICP
      filing first, and the review takes weeks; start it early.
    - An origin is scheme, host and port. Cookies are scoped to hosts, so a
      change of domain signs everyone out and strands links already sent;
      keep the old domain redirecting with `308`.
    - CORS is the browser's permission check for cross-origin reads, with a
      preflight for most requests; it does not replace authentication.
    - A nonce-based CSP with `'strict-dynamic'` makes injected script inert;
      list the storage endpoint in `connect-src` for direct uploads.
