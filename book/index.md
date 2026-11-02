---
title: Web App Guidebook
summary: A working reference to building and hosting web applications, from the ideas behind domains, identities, databases and services to step-by-step procedures on a specific platform.
---

## About this book

A web application that real people use rests on more than its code. It needs a
domain and a certificate, a database whose schema changes safely, services that
start, scale and stop, storage for files, a way to send email, identities and
keys that grant exactly what each part needs, and a release procedure that can
be undone. Each of these is simple on its own; together they make a setup that
is easy to get wrong and hard to remember a year later. This book collects that
know-how in one place, so that the next project starts from a procedure rather
than from memory.

The book has two kinds of part. **Foundations** explains the concepts in
provider-neutral terms: what each piece of a deployment is for, how the pieces
depend on one another, and which choices are conventions rather than
necessities. Each **platform part** then turns those concepts into numbered
procedures on one provider, with the console paths, settings and checks that
make a step reproducible. The first platform part covers Tencent CloudBase.

Chapters are self-contained: a reader who only needs to create a storage policy
should be able to open that chapter, find the procedure, and leave. Platform
chapters refer back to the foundation chapter that explains the idea behind
each step.

## How to read it

Sections are numbered within chapters (3.2, 3.2.1), as are figures and tables,
so that any statement can be referred to precisely. Boxed environments set
apart material of a particular kind:

Definition
:   A term fixed for use throughout the book.

Example
:   A worked instance of the surrounding idea.

Remark
:   A caveat, pitfall or side observation worth remembering.

Recipe
:   A concrete, reproducible procedure: console steps, commands or settings.

Warning
:   Something that silently breaks a deployment or leaks access.

Names in angle brackets, such as `<env-id>` or `<appid>`, are placeholders for
values from the reader's own account. Example values follow the providers' own
documentation, for instance the bucket `examplebucket-1250000000`. Console
labels on Chinese-language consoles are given in Chinese, as they appear on
screen, with an English gloss on first use. Consoles change their layout more
often than their concepts, so each platform chapter states when its procedures
were last checked.

Citations appear as bracketed numbers and resolve to the reference list at the
end of each chapter, mostly the providers' own documentation. Chapters marked
*in preparation* have a settled outline but unfinished text.
