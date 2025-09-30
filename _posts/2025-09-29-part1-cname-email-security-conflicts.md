---
title: "DNS Hardening | Part 1: How CNAME Records Block Email Security"
description: "Investigating DNS conflicts between CNAME records and SPF, DKIM, and MX records in AWS Route 53"
author:
date: 2025-09-29
tags:
  - DNS
  - CNAME
  - AWS
  - Route53
  - Email Security
  - SPF
  - DKIM
  - MX Records
categories:
  - DevOps
  - Infrastructure
  - Security
series: "DNS CNAME to Alias Migration"
part: 1
---

# The problem: CNAMEs block SPF, DKIM, and MX records

This post walks through a DNS challenge we encountered while strengthening email security across our staging and development domains. I will share how the scope of the problem was identified, the technical limitation that caused it, and the approach to finding a solution.

By the end, you'll understand why CNAME records can't coexist with other DNS record types, how to identify affected domains in your infrastructure, and what alternatives exist for pointing to AWS services like CloudFront and ELB.

While working to apply SPF, DKIM and null MX records across staging and development domains, we ran into a DNS limitation in AWS Route 53. When attempting to add a TXT record, we were blocked by this error:

```
InvalidChangeBatch: RRSet of type TXT with DNS name
example.domain.gov.uk. is not permitted because a conflicting RRSet
of type CNAME already exists...
```

CNAME records in DNS cannot coexist with any other record types — including the ones used for email security. This became a problem for domains pointing to AWS services like CloudFront or Elastic Load Balancers (ELBs), as these were commonly configured using CNAMEs.

## Writing a script to identify affected domains

To understand the scale of the issue, we wrote a simple Bash script using dig to check for CNAMEs across a list of internal subdomains:

```bash
for domain in \
  example1.domain.gov.uk \
  example2.domain.gov.uk; do

  cname=$(dig +short CNAME "$domain")

  if [[ -n "$cname" ]]; then
    echo "$domain → $cname"
  else
    echo "$domain has no CNAME"
  fi
done
```

This gave us a clear view of which domains were using CNAMEs, and where those records pointed. We categorised them into three groups:

- Domains pointing to AWS CloudFront
- Domains pointing to AWS ELBs
- Domains pointing to third-party services

This helped us scope the problem and begin exploring options to resolve it — particularly for domains that needed email security records.

## Cross-checking with the service register

As we built up our internal view of DNS usage, we cross-referenced the list of domains from our script with entries in our internal service register. This surfaced something unexpected: several publicly resolvable domains weren't recorded at all, and others appeared multiple times with inconsistent metadata.

This gap posed a risk: unregistered domains are easy to overlook, misconfigure, or lose track of entirely. We began a cleanup effort to:

- Add missing domains to the service register with clear ownership
- Remove duplicates or consolidate where appropriate
- Ensure only active, relevant domains are being managed

This reinforced the idea that DNS isn't just technical, it's an organisational responsibility.

## Using NCSC Mail Check for visibility

To validate our findings and assess external visibility, we added our subdomains to NCSC Mail Check. This helped us confirm:

- Which domains were missing SPF or MX records
- Whether existing configurations were resolvable from outside AWS
- How our security posture compared to broader government benchmarks

Mail Check also helped us detect legacy domains we hadn't previously considered, strengthening our inventory and accountability.

## What we learned in the investigation phase

- CNAME records can block the implementation of critical security records
- Bash scripting and tools like dig are still invaluable for quick diagnostics
- Publicly resolvable domains must be tracked in internal registers to ensure ownership and visibility
- NCSC Mail Check complements technical changes with external validation
- DNS hygiene begins with clarity, knowing what you own, where it points, and what it's missing

## What's next

In Part 2, we'll cover the decision-making process that followed this investigation:

- Why we chose Route 53 alias records
- How we tested changes safely in staging
- And how we used Terraform to roll out updates across 60+ domains with confidence
