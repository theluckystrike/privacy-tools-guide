---
layout: default
title: "Genetic Data Privacy Rights What 23andme Ancestry Can Do Wit"
description: "A technical guide for developers and power users on genetic data privacy rights, covering what 23andMe and AncestryDNA can legally do with your DNA"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "theluckystrike"
permalink: /genetic-data-privacy-rights-what-23andme-ancestry-can-do-wit/
categories: [guides]
tags: [privacy-tools-guide, tools, privacy]
reviewed: true
score: 9
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

## Genetic Privacy Tool Comparison

| Service | Privacy Model | Encryption | Data Ownership | Cost |
|---------|---------------|-----------|-----------------|------|
| 23andMe | Server holds data | TLS only | Company owns | $99-199 |
| AncestryDNA | Server holds data | TLS only | Ambiguous | $99-199 |
| Nebula Genomics | User option | E2E encryption | User | $299-499 |
| MyHeritage | Server holds data | TLS only | Company | $99-199 |
| FamilyTreeDNA | Server holds data | TLS | Company | $99+ |

Nebula Genomics and AncestryDNA (with privacy mode) provide stronger privacy, though no service offers complete anonymity once submitted.

## Understanding Re-Identification Risks

Even "anonymized" genetic data can be de-identified through various attacks:

### Linking Attack

Combine genetic data with public records:

```python
# Conceptual example of re-identification
genetic_markers = {
    "rs1234567": "G",  # Genetic SNP
    "rs7654321": "A"
}

# Cross-reference with DNA genealogy databases
# Like GEDmatch, WikiTree, etc.
# Identify closest genetic matches

# Combined with:
# - Public birth records
# - Surname frequency
# - Geographic information

# Result: Individual identified from "anonymous" data
```

Researchers have demonstrated identifying individuals from supposedly anonymized genetic datasets with 98%+ accuracy.

### Attribute Inference

Predict sensitive attributes from genetic data:

```python
def infer_traits(genetic_data: dict) -> dict:
    """Infer health and physical traits from genotype"""
    traits = {}

    # Example: Lactose intolerance
    if genetic_data.get("rs4988235") == "CC":
        traits["lactose_intolerant"] = True

    # Example: Height (polygenetic)
    if count_tall_alleles(genetic_data) > 200:
        traits["likely_tall"] = True

    # Example: APOE and Alzheimer's risk
    if genetic_data.get("APOE") == "e4/e4":
        traits["alzheimer_risk"] = "high"

    return traits
```

Even without identifying you directly, genetic data reveals sensitive health information.

## Genetic Discrimination Concerns

Despite legal protections like GINA, gaps remain:

### Life Insurance

GINA does not cover life insurance. Companies can request genetic tests and adjust rates based on results. In 2025, three major life insurers began requesting genetic screening.

### Disability Insurance

Similarly unprotected. Companies can legally use genetic predisposition to deny coverage or charge higher premiums.

### Long-Term Care Insurance

No protection. Genetic predisposition to dementia or other conditions directly impacts insurability and pricing.

### Employment Context

GINA prohibits discrimination, but some employers use genetic testing for:
- Drug testing (genetic sensitivity to substances)
- Occupational health screening
- Wellness programs that request genetic data

Legal battles continue around what constitutes "genetic testing" under GINA.

## Requesting Regulatory Oversight

Users can take action:

```
Contact your state attorney general:
- Request investigation of genetic data sharing
- Advocate for stronger genetic privacy laws

Write to legislature:
- Propose extension of GINA to life/disability insurance
- Support genetic data ownership laws
- Request mandatory transparency reports on law enforcement requests
```

Several states (Colorado, Minnesota) passed genetic privacy legislation in 2024-2025.

## Managing Genetic Data Across Services

If you've already submitted genetic data:

### Audit Submission Status

```python
# Check which services have your genetic data

services_with_data = [
    "23andMe",
    "AncestryDNA",
    "MyHeritage",
    "FamilyTreeDNA"
]

# For each service:
# 1. Log in to account
# 2. Navigate to DNA settings
# 3. Check consent status for research/sharing
# 4. Note download completion date
# 5. Document genetic file format
```

### Managing DNA Relatives Feature

The "DNA Relatives" feature creates privacy leakage:

```
Privacy implications:
- Your genetic data becomes identifiable through relatives
- Relatives' privacy violated without their explicit consent
- Law enforcement can identify you through family members
- Distant relatives become part of your surveillance surface
```

Disable DNA Relatives feature to reduce exposure, though this limits utility.

### Uploading Raw Data to GEDmatch

Some users export genetic data and upload to GEDmatch for genealogy research. This increases re-identification risk dramatically:

```
Risks of uploading to GEDmatch:
- Permanent storage (GEDmatch acquired by MyHeritage in 2019)
- Law enforcement access (no warrant required historically)
- Public access (researchers can search)
- Lack of deletion guarantees
```

Avoid uploading unless your threat model explicitly allows law enforcement access.

## Scenario: Genetic Testing and Career Security

Example threat model for professional concern:

```
Scenario: You're a software developer considering genetic testing
Threat: Future employer uses genetic predisposition info to disqualify you

Mitigations:
1. Don't test through major commercial services
2. If using service, opt out of research programs
3. Ensure no DNA relatives test at same services
4. Disable genetic relative matching features
5. Decline health reports (focus on ancestry only)
6. Delete genetic data after extracting ancestry info
```

Different threat models require different approaches.

## When Genetic Testing Makes Sense

Despite risks, genetic testing can be valuable:

- **Genuine medical need**: Consulting physician recommends specific genetic test
- **Family planning**: Carrier screening for genetic conditions
- **Ancestry research**: Understanding family history without health integration

In these cases, weigh benefits against risks carefully and use providers with strongest privacy guarantees (Nebula Genomics, privacy-focused services).

## Open-Source Genetic Analysis

For those with raw genetic data, analyze it locally:

```bash
# Install PLINK for genetic analysis
brew install plink

# Load genetic data in standard formats
plink --vcf yourdata.vcf --freq --out analysis

# Perform analysis without uploading to companies
# Maintain complete data privacy and ownership
```

This approach requires technical skill but provides maximum privacy.



## Related Articles

- [How To Exercise Montana Consumer Data Privacy Act Rights Dat](/privacy-tools-guide/how-to-exercise-montana-consumer-data-privacy-act-rights-dat/)
- [Smart City Surveillance: What Data Municipal Cameras and.](/privacy-tools-guide/smart-city-surveillance-privacy-rights-what-data-municipal-c/)
- [Data Subject Rights Automation Tools 2026: A Practical Guide](/privacy-tools-guide/data-subject-rights-automation-tools-2026/)
- [Gdpr Data Breach Notification Rights What Company Must.](/privacy-tools-guide/gdpr-data-breach-notification-rights-what-company-must-tell-you-within-seventy-two-hours/)
- [Children's Online Privacy Protection Act](/privacy-tools-guide/children-online-privacy-protection-act-coppa-rights-what-par/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
