---
layout: default
title: "Gdpr Controller Vs Processor Obligations Understanding Your"
description: "A practical developer guide to GDPR controller and processor roles, obligations, and your rights when interacting with services that handle your"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /gdpr-controller-vs-processor-obligations-understanding-your-/
categories: [guides]
tags: [privacy-tools-guide, tools, comparison]
reviewed: true
score: 7
voice-checked: true
intent-checked: true
---

Under GDPR, every entity handling personal data is either a **controller** (decides why and how data is processed) or a **processor** (processes data on the controller's behalf). Getting this wrong leads to compliance failures and fines.

**Controllers** determine the purpose and means of processing. If your company decides to collect user emails for a newsletter, your company is the controller. Controllers bear primary responsibility: they must establish a legal basis for processing, respond to data subject requests, report breaches within 72 hours, and conduct data protection impact assessments.

**Processors** act on the controller's instructions. Your email service provider (Mailchimp, SendGrid) is a processor -- it sends emails because you told it to. Processors must maintain processing records, implement appropriate security measures, notify the controller of breaches without undue delay, and not engage sub-processors without the controller's authorization.

| Obligation | Controller | Processor |
|-----------|-----------|-----------|
| Legal basis for processing | Must establish | N/A (follows controller) |
| Data subject requests (access, deletion) | Must respond within 30 days | Must assist controller |
| Breach notification | To supervisory authority (72h) | To controller (without undue delay) |
| DPA required | Must sign with processors | Must sign with controllers |
| DPIA (impact assessment) | Required for high-risk processing | Must assist |
| Records of processing | Required | Required |
| Fines (maximum) | 4% annual revenue / 20M EUR | 4% annual revenue / 20M EUR |

**For developers:** if you build a SaaS product, you are typically the processor for your customers' data and the controller for your own user account data. You need Data Processing Agreements (DPAs) with every sub-processor you use (cloud hosting, analytics, payment providers). Map your data flows to identify every controller-processor relationship in your stack.

A single entity can be both controller and processor for different data sets. This is common and expected. Document each role clearly.

## Related Articles

- [Gdpr Joint Controller Agreement Template](/gdpr-joint-controller-agreement-template/)
- [Ccpa Vs Gdpr Comparison Guide 2026](/ccpa-vs-gdpr-comparison-guide-2026/)
- [Encrypted Cloud Storage Gdpr Compliant 2026](/encrypted-cloud-storage-gdpr-compliant-2026/)
- [Enterprise Privacy Compliance Tool Comparison for GDPR.](/enterprise-privacy-compliance-tool-comparison-for-gdpr-and-ccpa/)
- [GDPR Article 17 Erasure Implementation Code](/gdpr-article-17-erasure-implementation-code/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
