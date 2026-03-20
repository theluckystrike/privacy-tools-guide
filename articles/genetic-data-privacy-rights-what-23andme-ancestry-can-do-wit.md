---
layout: default
title: "Genetic Data Privacy Rights What 23andme Ancestry Can Do Wit"
description: "A technical guide for developers and power users on genetic data privacy rights, covering what 23andMe and AncestryDNA can legally do with your DNA."
date: 2026-03-16
author: "theluckystrike"
permalink: /genetic-data-privacy-rights-what-23andme-ancestry-can-do-wit/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

23andMe and AncestryDNA can use genetic data for research partnerships, pharmaceutical company collaboration, and law enforcement requests (with legal process). Genetic data cannot be changed like passwords if breached, and HIPAA does not cover direct-to-consumer services. GDPR (EU residents), CCPA/CPRA (California), and GINA (US) provide varying protections; export your data before deletion since residual copies persist in research databases and backups. For developers handling genetic data, apply differential privacy, federated learning, or homomorphic encryption to prevent re-identification attacks.

## How Genetic Testing Companies Process Your Data

When you mail a saliva sample to 23andMe or AncestryDNA, you are not just getting ancestry reports. The laboratory extracts your DNA, genotypes hundreds of thousands of genetic markers (single nucleotide polymorphisms or SNPs), and stores that genetic profile indefinitely. Your genetic data becomes a valuable asset that companies can use in ways you may not expect.

Both companies operate under the premise of informed consent, but the scope of that consent is broader than most users realize. By agreeing to their terms of service, you typically grant permission for research partnerships, aggregate data sharing, and population studies.

## What 23andMe Can Legally Do With Your DNA

23andMe's terms of service (as of 2026) allow several key uses:

### Research and Development

The company can use your genetic data for internal research and development, including developing new products, improving algorithms, and publishing scientific studies. Under their consent framework, you can opt out of research participation, but your genetic data still remains in their system unless you specifically request deletion.

### Therapeutic Partnerships

23andMe has partnered with pharmaceutical companies including GSK (GlaxoSmithKline) and AbbVie. These partnerships involve sharing anonymized or de-identified genetic data for drug target discovery. While companies claim this data cannot be traced back to individuals, re-identification attacks have been demonstrated in academic literature, raising concerns about true anonymization.

### Law Enforcement Requests

Both 23andMe and AncestryDNA can respond to valid legal requests from law enforcement. This includes subpoenas, court orders, and search warrants. In 2019, investigators used GEDmatch (a different platform) to identify the Golden State Killer, demonstrating how genetic databases can be used for criminal investigations. While 23andMe has stated they require a warrant for criminal investigations, the legal landscape continues to evolve.

### Data Breaches

Genetic data, once stolen, cannot be changed like a password. In 2023, 23andMe suffered a breach where attackers accessed family tree data and profile information. Users with "DNA Relatives" features enabled were particularly vulnerable. This incident highlights the permanent nature of genetic data exposure.

## What AncestryDNA Can Do With Your DNA

AncestryDNA operates under similar frameworks with some notable differences:

### Ownership Claims

Ancestry's terms of service have historically included language suggesting they retain certain rights to your genetic data. This has faced criticism from privacy advocates who argue users should maintain full ownership. Review current terms carefully before submitting samples.

### Law Enforcement and Third Parties

AncestryDNA has acknowledged responding to law enforcement requests. They maintain a transparency report showing the number of requests received. Like 23andMe, they require legal process before releasing individual-level data.

### Population Studies

Your genetic data may be included in aggregate studies about population genetics, migration patterns, and health research. While individual identification is theoretically impossible, the cumulative dataset represents a significant privacy concern.

## Your Legal Rights Under Current Regulations

Several regulations provide genetic data protections, though gaps remain:

### GDPR (European Union)

If you are an EU resident, the General Data Protection Regulation provides strong protections for genetic data. You have the right to access your data, request deletion ("right to be forgotten"), and object to processing. Companies must respond to deletion requests within 30 days. For developers building applications that interact with genetic data, GDPR compliance requires explicit consent for each processing purpose.

### CCPA/CPRA (California)

California residents have the right to know what personal information is collected, request deletion, and opt out of sale. Genetic data is considered sensitive personal information under CPRA, requiring even stronger protections. The California Privacy Rights Act explicitly addresses genetic data handling.

### GINA (United States)

The Genetic Information Nondiscrimination Act prohibits health insurers and employers from discriminating based on genetic information. However, GINA does not cover life insurance, disability insurance, or long-term care insurance, creating significant gaps in protection.

### HIPAA Limitations

HIPAA does not apply to direct-to-consumer genetic testing companies. These companies are not covered entities under HIPAA, meaning your genetic data receives weaker federal protection than health records held by medical providers. This is a critical distinction for developers building health-related applications.

## How to Exercise Your Data Rights

### Requesting Data Deletion

Both 23andMe and AncestryDNA provide account deletion options in their settings. For 23andMe, navigate to Settings → Privacy → Delete Your Account. For AncestryDNA, use the "Delete DNA Test" option in your DNA settings. However, deletion may not be complete:

```python
# After account deletion, residual data may persist:
# - Sample data in research databases (if you consented to research)
# - Anonymized data in academic collaborations
# - Law enforcement holds or litigation holds
# - Backup systems with retention policies
```

### Understanding Residual Data

Even after account deletion, companies may retain:
- De-identified data used in published research
- Aggregate statistics that cannot be linked to individuals
- Data required for legal compliance or fraud prevention
- Information in system backups

### Export Your Data First

Before deletion, download your complete data archive. Both services provide this feature:

- **23andMe**: Settings → Download your data → Request your archive
- **Ancestry**: Account Settings → Export DNA Data

This gives you a local copy of your raw genetic data, which you can then manage independently or upload to alternative platforms.

## Technical Considerations for Developers

If you are building applications that handle genetic data, consider these privacy-preserving approaches:

### Differential Privacy

Apply differential privacy techniques when working with genetic datasets. This adds calibrated noise to query results, preventing individual re-identification while maintaining statistical utility.

```python
# Example: Adding Laplace noise for differential privacy
import numpy as np

def laplace_mechanism(true_count, epsilon=1.0):
    scale = 1.0 / epsilon
    noise = np.random.laplace(0, scale)
    return int(true_count + noise)
```

### Federated Learning

Process genetic data locally on user devices rather than centralizing sensitive datasets. Federated learning allows model training without raw data leaving the user's control.

### Homomorphic Encryption

For computations on encrypted genetic data, homomorphic encryption enables processing without decryption. While computationally expensive, emerging libraries make this more practical:

```python
# Conceptual example using SEAL library
# Encrypt genetic markers, perform analysis, decrypt results
# without exposing raw data to the processing server
```

## Alternatives to Commercial Genetic Testing

For privacy-conscious users, alternatives include:

- **Self-hosted sequencing**: Services like Nebula Genomics offer more control but still require trust in the provider
- **Open-source tools**: Programs like PLINK work with your locally-held genetic data
- **Declining participation**: The safest option is not to participate in commercial genetic databases

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
