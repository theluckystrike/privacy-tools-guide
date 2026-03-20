---

layout: default
title: "Her Dating App Privacy: What LGBTQ+ Specific Data Is."
description: "A technical breakdown of data collection practices in Her, the LGBTQ+ dating app. Learn what personal information developers store, how it's processed, and privacy implications for users."
date: 2026-03-16
author: theluckystrike
permalink: /her-dating-app-privacy-what-lgbtq-specific-data-is-collected/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Understanding data collection practices in LGBTQ+ dating applications requires examining both standard mobile app telemetry and identity-specific data points. Her, one of the largest dating platforms designed for queer women and the LGBTQ+ community, collects various categories of user data that warrant careful analysis from a privacy engineering perspective.

## Data Categories Collected by Her

Her collects data across several distinct categories that developers and security-conscious users should understand. The application's data handling practices reveal important privacy implications for a user base that may face unique risks from data exposure.

### Account Registration and Identity Data

When creating a Her profile, users provide information that forms the core identity dataset:

```json
{
  "user_profile": {
    "email": "user@example.com",
    "phone_number": "+1234567890",
    "date_of_birth": "1990-01-15",
    "display_name": "Sarah",
    "profile_photos": ["photo1.jpg", "photo2.jpg"],
    "gender_identity": "woman",
    "sexual_orientation": ["lesbian", "queer"],
    "gender_pronouns": "she/her"
  }
}
```

The app specifically requests sexual orientation and gender identity during onboarding—data points rarely collected by mainstream dating applications but essential to Her's matching algorithm. This identity-specific information represents sensitive personal data that requires robust protection measures.

### Location and Geospatial Data

Like most dating applications, Her relies heavily on location data to function:

```javascript
// Approximate location data structure stored by Her
const locationData = {
  "last_known_location": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "accuracy_meters": 100,
    "timestamp": "2026-03-16T10:30:00Z"
  },
  "preferred_search_radius": 50,  // in kilometers
  "city": "San Francisco",
  "country": "US"
};
```

The app continuously tracks user proximity to other users for match purposes. Location history can reveal sensitive patterns—regular visits to LGBTQ+ venues, community centers, or healthcare facilities. Developers should note that this geospatial data often persists even after account deletion.

### Behavioral and Usage Analytics

Her collects substantial behavioral data through in-app interactions:

| Data Type | Purpose | Storage Duration |
|-----------|---------|------------------|
| Profile views | Matching algorithm | 2 years |
| Swipe patterns | Preference learning | Indefinite |
| Message content | Service delivery | Until deletion |
| In-app purchases | Payment processing | 7 years (financial records) |
| Session duration | Engagement metrics | 1 year |
| Feature usage | Product improvement | 2 years |

This telemetry often includes detailed interaction patterns that can infer lifestyle characteristics, political affiliations, and social connections within the LGBTQ+ community.

### Device and Technical Metadata

Mobile applications collect device-level information:

```bash
# Typical device fingerprint data points collected
- Device model and manufacturer
- Operating system version
- Screen resolution
- App version
- Network type (WiFi/5G/LTE)
- Advertising identifiers (IDFA/GAID)
- Push notification tokens
- Browser cookies (if web access exists)
```

## LGBTQ+-Specific Data Handling Concerns

The collection of sexual orientation and gender identity data raises specific privacy concerns that developers should understand when evaluating dating app ecosystems.

### Identity Verification and Outing Risk

Her stores verified identity markers that couldouting risk if exposed. Unlike mainstream dating apps where users might maintain pseudonyms, many Her users share authentic identities due to the app's community-focused nature. A data breach exposing the following could have serious consequences:

- User identities linked to their LGBTQ+ status
- Location data correlating with venue visits
- Message content revealing relationship-seeking behavior
- Profile photos that could identify users in professional contexts

### Algorithmic Profiling

The app's recommendation engine processes identity data to generate matches:

```python
# Simplified matching algorithm consideration
def calculate_compatibility(user_a, user_b):
    score = 0
    
    # Identity matching contributes to score
    if user_a.orientation in user_b.seeking:
        score += matching_weight["orientation"]
    
    # Location proximity is weighted heavily
    distance = haversine_distance(user_a.location, user_b.location)
    score += matching_weight["distance"](distance)
    
    # Community membership factors
    if user_a.community_tags & user_b.community_tags:
        score += matching_weight["community"]
    
    return score
```

This algorithmic approach means the platform maintains detailed profiles of user preferences, orientations, and behavioral patterns that extend beyond explicit profile data.

### Third-Party Data Sharing

Her's privacy policy indicates data sharing with:

- Advertising partners (for targeted advertising)
- Analytics providers (app performance optimization)
- Legal authorities (when legally required)
- Potential acquirers (in merger or acquisition scenarios)

Developers reviewing this ecosystem should note that LGBTQ+-specific data may be included in these shared datasets, potentially reaching parties with less stringent privacy commitments.

## Technical Privacy Considerations

For developers building similar applications or security researchers evaluating these platforms, several technical patterns emerge:

### Data Encryption at Rest

Production dating applications should implement encryption for sensitive identity fields:

```python
# Example: Encrypted field storage for sensitive data
from cryptography.fernet import Fernet

class SensitiveProfileData:
    def __init__(self, encryption_key):
        self.cipher = Fernet(encryption_key)
    
    def store_identity_fields(self, user_id, identity_data):
        encrypted = {
            "gender_identity": self.cipher.encrypt(
                identity_data["gender_identity"].encode()
            ),
            "sexual_orientation": self.cipher.encrypt(
                identity_data["sexual_orientation"].encode()
            ),
            "encrypted_at": datetime.utcnow().isoformat()
        }
        return self.save_to_user_record(user_id, encrypted)
```

### Data Retention Implementation

Responsible applications implement explicit retention policies:

```javascript
// Data lifecycle management pattern
const DATA_RETENTION_POLICY = {
  "messages": {
    "active": "until user deletion",
    "backup_deletion": "90 days after account deletion"
  },
  "location_history": {
    "active": "2 years",
    "anonymization": "Aggregate after 1 year"
  },
  "profile_data": {
    "active": "until account deletion",
    "deletion_grace_period": "30 days"
  },
  "analytics": {
    "retention": "13 months",
    "anonymization": "Immediate for IP addresses"
  }
};
```

### User Control Mechanisms

Privacy-conscious applications provide granular data controls:

- Profile visibility settings
- Location sharing toggles
- Message retention preferences
- Data export capabilities
- Account deletion workflows

## Recommendations for Privacy-Conscious Users

Users concerned about their data privacy on Her should consider these technical measures:

1. **Limit profile information**: Provide only necessary identity details
2. **Disable location history** when not actively using the app
3. **Use unique email** addresses specifically for dating profiles
4. **Review connected accounts** and third-party permissions regularly
5. **Utilize in-app blocking** features to limit exposure to specific users
6. **Request data exports** periodically to understand stored information
7. **Consider account deletion** rather than simply discontinuing use

## Conclusion

Her, like other LGBTQ+-focused dating platforms, collects a comprehensive dataset that includes identity-specific information beyond typical dating app telemetry. The combination of sexual orientation, gender identity, location data, and behavioral patterns creates detailed user profiles with unique privacy implications for the LGBTQ+ community.

Understanding these data collection practices empowers developers building similar applications to implement stronger privacy controls and helps users make informed decisions about their digital footprint in spaces designed for community connection.

---

**Related Reading**

- [Privacy Tools Guide Hub](/privacy-tools-guide/guides-hub/)


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
