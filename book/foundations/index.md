---
title: Foundations
summary: "The pieces of a hosted web application and how they depend on one another: domains, identities, databases, services, storage, email and releases."
---

A hosted web application is a small distributed system. Browsers reach it
through a **domain** and a certificate; a **web service** answers them and
forwards API calls to an **API service**; the API reads and writes a
**database**, keeps files in **object storage**, and sends **email**. Every one
of these parts acts under an **identity** that some policy allows to do exactly
what it needs, and the whole arrangement exists more than once, as separate
**environments** that a **release** procedure moves code through.

The chapters in this part take those pieces in that order. They are written
without reference to any one provider, but they point out where providers
differ in ways that matter, and they record the failure modes that a working
setup has to avoid. The platform parts that follow assume these chapters and
refer back to them.
