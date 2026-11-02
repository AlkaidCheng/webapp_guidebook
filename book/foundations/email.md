---
title: Transactional Email
summary: "Sending account email that arrives: sender domains, SPF, DKIM and DMARC, templates and their review, and choosing an interface."
references:
  gmail-senders: "Google, *Email sender guidelines*, Gmail Help, <https://support.google.com/mail/answer/81126>."
  rfc8058: "J. Levine and T. Herkula, *Signaling One-Click Functionality for List Email Headers*, RFC 8058 (2017), <https://www.rfc-editor.org/rfc/rfc8058>."
  rfc5321: "J. Klensin, *Simple Mail Transfer Protocol*, RFC 5321 (2008), <https://www.rfc-editor.org/rfc/rfc5321>."
  tc-ses-identity: "Tencent Cloud, *Identity Verification and Configuration*, Simple Email Service documentation, <https://www.tencentcloud.com/document/product/1084/42371>."
  ses-mail-from: "Amazon Web Services, *Using a custom MAIL FROM domain*, Amazon SES Developer Guide, <https://docs.aws.amazon.com/ses/latest/dg/mail-from.html>."
  rfc7208: "S. Kitterman, *Sender Policy Framework (SPF) for Authorizing Use of Domains in Email, Version 1*, RFC 7208 (2014), <https://www.rfc-editor.org/rfc/rfc7208>."
  rfc6376: "D. Crocker, T. Hansen and M. Kucherawy (eds.), *DomainKeys Identified Mail (DKIM) Signatures*, RFC 6376 (2011), <https://www.rfc-editor.org/rfc/rfc6376>."
  rfc9989: "T. Herr and J. Levine (eds.), *Domain-Based Message Authentication, Reporting, and Conformance (DMARC)*, RFC 9989 (2026), <https://www.rfc-editor.org/rfc/rfc9989>."
  tc-ses-config: "Tencent Cloud, *Email Configuration*, Simple Email Service documentation, <https://www.tencentcloud.com/document/product/1084/48862>."
  tc-ses-sending: "Tencent Cloud, *Email Sending*, Simple Email Service documentation, <https://www.tencentcloud.com/document/product/1084/48863>."
  tc-ses-review: "Tencent Cloud, *Template* (review time), Simple Email Service FAQ, <https://www.tencentcloud.com/document/product/1084/39651>."
  rfc6409: "R. Gellens and J. Klensin, *Message Submission for Mail*, RFC 6409 (2011), <https://www.rfc-editor.org/rfc/rfc6409>."
  rfc8314: "K. Moore and C. Newman, *Cleartext Considered Obsolete: Use of Transport Layer Security (TLS) for Email Submission and Access*, RFC 8314 (2018), <https://www.rfc-editor.org/rfc/rfc8314>."
  ses-smtp: "Amazon Web Services, *Using the Amazon SES SMTP interface to send email*, Amazon SES Developer Guide, <https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html>."
  ses-api: "Amazon Web Services, *Using the Amazon SES API to send email*, Amazon SES Developer Guide, <https://docs.aws.amazon.com/ses/latest/dg/send-email-api.html>."
  rfc5322: "P. Resnick (ed.), *Internet Message Format*, RFC 5322 (2008), <https://www.rfc-editor.org/rfc/rfc5322>."
  owasp-forgot: "OWASP Cheat Sheet Series, *Forgot Password Cheat Sheet*, <https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html>."
---

## Transactional and marketing email

An application sends email for two different reasons, and the difference
decides how the mail is sent.

!!! definition "Transactional email"
    A message sent to one recipient because of something that recipient did
    or that concerns their account: a sign-up verification code, a password
    reset, an invitation, a receipt, a security notice. The recipient expects
    it, usually within seconds. *Marketing* email, in contrast, is sent to a
    list on the sender's initiative: newsletters, announcements, offers.

Mailbox providers treat the two differently. Gmail, for one, requires
marketing and other subscribed messages from bulk senders to offer one-click
unsubscription, and requires bulk senders to authenticate their domain with
SPF, DKIM and DMARC and to keep their spam-complaint rate low
[@gmail-senders]. One-click unsubscription is
signalled by a pair of message headers that let the mailbox provider
unsubscribe the user with a single request [@rfc8058]. A verification code
has no list to leave, and its value lies in arriving at once.

Mixing the two on one sending domain lets complaints about marketing mail
lower the reputation on which the delivery of codes depends. A transactional
sender therefore has a domain, a provider configuration and often a provider
product of its own (Table 7.1).

| Property | Transactional | Marketing |
|---|---|---|
| Trigger | A user's action or account event | The sender's schedule |
| Recipients | One per message | A list |
| Expected latency | Seconds | Hours are acceptable |
| Unsubscribe | Not applicable | Required, one-click for bulk senders [@gmail-senders] |
| Content | Fixed templates with a few variables | Varies per campaign |

Table: Transactional and marketing email.

## Sender domains

Mail is accepted by receiving servers on the strength of its domain, so a
transactional sender needs a domain whose DNS it controls. The convention is
a dedicated subdomain such as `mail.example.com`, with addresses like
`no-reply@mail.example.com`, for a reason found in the records that sending
requires.

Receiving servers deliver mail for a domain to the hosts named in that
domain's MX records [@rfc5321]. Email providers ask for an MX record on the
sender domain: Tencent Cloud's email service lists an MX record pointing at
its own mail servers among the records that verify a sender domain
[@tc-ses-identity], and Amazon SES requires one on a custom MAIL FROM domain to
receive bounces and complaints, while recommending that the domain not be
used to receive mail [@ses-mail-from]. An MX record on the parent domain
`example.com` would redirect the organisation's own mailboxes to the email
provider. Placing the sender on a subdomain gives it records of its own and
leaves the parent domain free for mailboxes.

Three authentication mechanisms let a receiver check that mail claiming to
come from the domain was sent with the domain owner's consent. Each is
published as a DNS TXT record.

!!! definition "SPF, DKIM and DMARC"
    *SPF* lists the hosts authorized to send mail with a given domain in the
    envelope sender (the SMTP `MAIL FROM`), as a TXT record beginning
    `v=spf1` at that domain [@rfc7208]. *DKIM* attaches a signature to each
    message, made with a private key held by the sender; the public key is
    published as a TXT record at `<selector>._domainkey.<domain>`, and the
    signature names the signing domain [@rfc6376]. *DMARC* lets the owner of
    the domain in the visible `From:` header state what receivers should do
    with mail that passes neither SPF nor DKIM for a domain *aligned* with it,
    through a TXT record at `_dmarc.<domain>` [@rfc9989].

Alignment is what connects the three. SPF authenticates the envelope sender
and DKIM the signing domain, neither of which the reader sees; DMARC requires
one of them to match the domain of the `From:` address. In the default
relaxed mode a domain matches when both belong to the same organizational
domain, so a message from `no-reply@mail.example.com` signed by
`mail.example.com` aligns [@rfc9989]. A domain owner starts with the policy
`p=none` and a reporting address, reads the aggregate reports to find
legitimate mail that fails, and only then tightens the policy to `quarantine`
or `reject` [@rfc9989].

| Name | Type | Value (example) | Purpose |
|---|---|---|---|
| `mail.example.com` | MX | `10 <provider-mx-host>.` | Bounces; provider verification |
| `mail.example.com` | TXT | `v=spf1 include:<provider-spf-domain> ~all` | SPF for the envelope sender |
| `<selector>._domainkey.mail.example.com` | TXT | `v=DKIM1; k=rsa; p=<public-key>` | DKIM public key |
| `_dmarc.mail.example.com` | TXT | `v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com` | DMARC policy and reports |

Table: The DNS records of a sender subdomain. The provider supplies the
host names, include domain, selector and key.

When the sender is a subdomain, DMARC also expects a policy record at the
organizational domain, `_dmarc.example.com` [@rfc9989]. A name holds at most
one SPF record: when several services send for the same domain, they are
combined in one record, since more than one makes SPF evaluation fail with a
permanent error [@rfc7208; @tc-ses-identity]. After the records are published,
the provider checks them and marks the domain verified; Tencent Cloud notes
that this can take from five minutes to two hours [@tc-ses-config].

!!! recipe "Checking a sender domain's records"
    ```bash
    dig +short MX  mail.example.com
    dig +short TXT mail.example.com                       # one v=spf1 record
    dig +short TXT <selector>._domainkey.mail.example.com
    dig +short TXT _dmarc.mail.example.com
    ```

    Compare each answer with the value the provider's console shows. Once the
    domain is verified, send one message to a mailbox at a large provider and
    read its headers: the `Authentication-Results` header should report
    `spf=pass`, `dkim=pass` and `dmarc=pass`.

## Templates

A transactional message is a template with a few variables: a greeting, a
code, a link. Some providers accept only templates they have reviewed.
Tencent Cloud's email service reviews each template after submission and
allows it to be used only once approved [@tc-ses-sending]; a template
submitted on a business day is reviewed within one business day, one submitted
on a weekend on the next business day [@tc-ses-review]. Every change to the
text is a new submission, so templates are finished a working day before the
release that needs them, not on the day.

Templates that contain a link, such as an invitation or a password-reset
message, embed the web origin. Where the provider allows the whole link to be
a variable, one template serves every environment; where the origin is part of
the reviewed text, each origin needs its own version of the template. Staging
and production then have separate templates, and a move to a new domain
([Chapter 2](domains.md)) requires new versions, reviewed before the switch.

!!! warning "A template change is on the release's critical path"
    A release that changes an email's wording or the application's origin
    cannot be completed until the new templates are approved. Submit them as
    soon as their text is final, and keep the previous templates until the
    release is verified, so that a rollback still has templates to send.

## Interfaces

Providers offer two ways to send (Table 7.3). *SMTP* is the standard mail
submission protocol, spoken on port 587 with `STARTTLS` [@rfc6409] or with TLS
from the first byte on port 465 [@rfc8314]. An *HTTP API* is the provider's own
interface, called with the provider's SDK or a signed HTTPS request; Amazon SES
offers both [@ses-smtp; @ses-api].

| Property | SMTP | HTTP API |
|---|---|---|
| Portability | Any provider; switching is a configuration change | Provider-specific code |
| Message content | Composed by the application | A template ID with variables, where the provider keeps templates |
| Credentials | SMTP user name and password | Access keys of an identity ([Chapter 3](access.md)) |
| Errors | SMTP reply codes at submission; bounces arrive later as mail | Structured responses to each request |

Table: The two interfaces to an email provider.

SMTP keeps the application independent of the provider and is the better
default where the provider does not insist on reviewed templates. Where it
does, the HTTP API is the only way to use them, and the application then
holds a template identifier per message type in its configuration. Either way,
the sending identity belongs to one environment, and development uses a sender
that writes messages to the log instead of delivering them.

The `From:` header carries a display name as well as the address, in the form
`Example App <no-reply@mail.example.com>` [@rfc5322]. The display name is what
recipients read in their inbox, and a bare address looks like what it is, a
machine. A `Reply-To:` header pointing at a monitored mailbox gives
recipients somewhere to answer.

## Verification codes

The most common transactional message carries a short code that proves
control of an address: at sign-up, at sign-in from a new device, or for a
password reset. The code is a credential while it lives, and is handled as
one [@owasp-forgot].

!!! recipe "Rules for emailed codes"
    1. Generate the code with a cryptographically secure random generator.
    2. Store only a hash of it, with the account, the purpose and an expiry
       time; never log it.
    3. Expire it within minutes, and accept it once.
    4. Limit verification attempts per code and per account: a six-digit
       code has a million values, which is enough only if guessing is
       limited.
    5. Limit how often a code can be sent to one address, and from one
       client, so that the form cannot be used to flood a mailbox.
    6. Answer a request for an unknown address exactly as for a known one, so
       that the form does not reveal which addresses have accounts.

[Chapter 14](../cloudbase/storage-email.md) sets up Tencent Cloud's email
service: the sender subdomain and its records, templates and their review, and
the identity that sends through the API.

!!! summary "Key points"
    - Transactional email is expected, per-recipient and urgent; keep it
      apart from marketing mail, which carries unsubscribe obligations.
    - Send from a subdomain such as `mail.example.com`. The provider's MX
      record then sits on the subdomain, leaving the parent domain's mailboxes
      untouched.
    - Publish SPF (one record per name), DKIM and DMARC; DMARC requires the
      `From:` domain to align with SPF or DKIM. Start with `p=none` and read
      the reports.
    - Where templates are reviewed, submit them a working day ahead; a
      template that embeds the origin needs a version per origin.
    - SMTP is portable; an HTTP API is needed for reviewed templates. Give
      `From:` a display name.
    - Codes are short-lived, single-use, hashed, rate-limited credentials,
      and requests for unknown addresses get the same answer as for known
      ones.
