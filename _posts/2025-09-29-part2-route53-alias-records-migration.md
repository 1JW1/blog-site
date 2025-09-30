---
title: "DNS Hardening | Part 2: Migrating to Route 53 Alias Records"
description: "Implementing alias records with Terraform to resolve CNAME conflicts and enable email security across 44 domains"
author:
date: 2025-09-29
tags:
  - DNS
  - Route53
  - Alias Records
  - Terraform
  - AWS
  - CloudFront
  - ELB
  - Infrastructure as Code
categories:
  - DevOps
  - Infrastructure
  - Security
series: "DNS CNAME to Alias Migration"
part: 2
related:
  - part1-cname-email-security-conflicts.md
---

# Implementing the Alias Records Solution

## Recap: What we learned in Part 1

In Part 1, we uncovered a recurring DNS issue that was preventing us from applying SPF, DKIM and MX records to staging and development domains. The problem stemmed from the use of CNAME records, which cannot coexist with other record types at the same DNS label.

We also discovered:

- Several domains pointing to AWS services (like CloudFront and ELB) via CNAME
- Gaps in our service register
- Missing or misconfigured email records across our internal estate

With these findings in hand, the next step was to agree on a consistent approach and begin implementing a fix.

## Making the decision

Before making any changes, we recorded a formal decision with input from the cloud engineering team. Our options were:

- Retain CNAMEs for domains that don't need email records
- Use conditional logic to handle DNS conflicts on a case-by-case basis
- Replace all CNAMEs pointing to AWS resources with Route 53 alias records

After testing and discussion, we agreed to standardise on alias records for any domain pointing to CloudFront or ELB. This would allow us to apply SPF, DKIM and MX records consistently, reduce complexity, and align with AWS and NCSC best practices.

## Side effects and trade-offs

Replacing CNAMEs with alias records wasn't without risk. We needed to destroy existing CNAME records and recreate them as alias records, which could cause DNS resolution issues if not properly sequenced.

During early changes, some development environments experienced brief downtime. This taught us two critical lessons: inform affected teams before making changes, and schedule deployments for times when those environments are least likely to be in use. Combined with our two-step deployment approach (first removing CNAMEs, then adding alias records), these practices minimized disruption across the remaining rollout. These lessons will be essential when we eventually tackle production domains, helping ensure a smoother transition with less disruption to live services.

## Updating Terraform to use alias records

Our DNS is managed via Terraform, which allowed us to roll out changes safely and repeatably. Below is an example of how we defined an alias record for domains that previously used a CNAME:

```terraform
locals {
  alias_domains = {
    "test-documents.api.hackney.gov.uk" = {
      target_dns     = "d1234.cloudfront.net"
      target_zone_id = "Z2FDTNDATAQYW2"
    }
  }
}

resource "aws_route53_record" "alias_record" {
  for_each = local.alias_domains

  zone_id = aws_route53_zone.hackney_gov_uk.zone_id
  name    = each.key
  type    = "A"

  alias {
    name                   = each.value.target_dns
    zone_id                = each.value.target_zone_id
    evaluate_target_health = false
  }
}
```

We also added supporting DNS records for non-mail-sending domains:

```terraform
resource "aws_route53_record" "null_spf" {
  for_each = local.alias_domains

  zone_id = aws_route53_zone.hackney_gov_uk.zone_id
  name    = each.key
  type    = "TXT"
  ttl     = 3600
  records = ["v=spf1 -all"]
}

resource "aws_route53_record" "null_mx" {
  for_each = local.alias_domains

  zone_id = aws_route53_zone.hackney_gov_uk.zone_id
  name    = each.key
  type    = "MX"
  ttl     = 3600
  records = ["0 ."]
}
```

This setup allowed us to remove the conflicting CNAME, replace it with an alias, and immediately apply mail-related DNS records.

## Testing on staging and development domains

We started small: two staging domains were selected as test cases.

- test-documents.api.hackney.gov.uk
- support-test.hackney.gov.uk

We replaced their CNAME records with alias records pointing to their respective CloudFront distributions. We also added null SPF and MX records to reflect their non-mail-sending status.

Once deployed, we used:

- dig and other command-line tools to verify the new records
- NCSC Mail Check to confirm external visibility and compliance
- AWS console and Terraform state to validate alias record creation

The results were exactly as expected: the alias records functioned like CNAMEs, but without blocking the TXT and MX records we needed to apply.

## Scaling it out safely

With the successful test complete, we proceeded to roll out the same pattern to over 44 internal domains.

To manage risk, we submitted changes in two separate pull requests:

1. One to remove the CNAME records from affected domains
2. One to add the alias records and associated SPF, DKIM, and MX records

This two-step approach allowed us to avoid DNS conflicts during the Terraform apply process and kept our changes clear and auditable.

## What we learned in staging

- Alias records are a reliable replacement for CNAMEs pointing to AWS services
- Terraform enabled safe, repeatable deployments, especially when split across separate pull requests
- NCSC Mail Check was critical for validation, helping us confirm external visibility of our records
- Testing small first built team confidence, and uncovered no negative side effects
- Mail records like SPF and DKIM can now be applied consistently, strengthening our email security posture

## What's next

With staging successfully migrated, the next steps are:

- Updating production domains using the same approach
- Coordinating with teams and vendors managing third-party DNS
- Navigating constraints around partial access and shared infrastructure
- Continuing to monitor changes through NCSC Mail Check and internal dashboards
