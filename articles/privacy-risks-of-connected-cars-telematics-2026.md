---
layout: default
title: "Privacy Risks of Connected Cars & Telematics 2026: What"
description: "What Tesla, GM, Toyota collect via telematics. How OnStar works, disabling tracking, insurance data sharing risks, and federal privacy regulations."
author: "Privacy Tools Guide"
date: 2026-03-22
last_modified_at: 2026-03-22
reviewed: true
score: 9
voice-checked: true
intent-checked: true
tags: [privacy-tools-guide, connected cars, telematics, privacy, vehicle tracking, data collection]
permalink: /privacy-risks-of-connected-cars-telematics-2026/
---


Privacy Risks of Connected Cars & Telematics 2026: What Data Is Being Collected

Your car is a mobile computer collecting more data than most smartphones. Tesla logs every acceleration, GPS coordinate, and camera frame. GM's OnStar tracks location even when you're not driving. Insurance companies use telematics data to raise rates or deny coverage. This guide covers what's being collected, who's buying it, and how to disable tracking.

Table of Contents

- [What Modern Cars Collect](#what-modern-cars-collect)
- [Who Collects What? By Manufacturer](#who-collects-what-by-manufacturer)
- [Insurance Companies & Usage-Based Pricing](#insurance-companies-usage-based-pricing)
- [Federal Privacy Regulations (Emerging)](#federal-privacy-regulations-emerging)
- [How to Disable Telematics (Full Guide)](#how-to-disable-telematics-full-guide)
- [What Data Brokers Buy From Automakers](#what-data-brokers-buy-from-automakers)
- [Real Example - Privacy-First Driving Setup](#real-example-privacy-first-driving-setup)

What Modern Cars Collect

The Data Stream (Real-Time)

Every connected vehicle generates a continuous stream of telematics data:

Location & Movement:
- GPS coordinates (accurate to 5 meters)
- Speed and acceleration patterns
- Braking force and sudden stops
- Route taken and time spent at location
- Number of passengers (seat sensors in some models)
- Parking location and duration

Vehicle Diagnostics:
- Engine temperature, oil pressure, fuel consumption
- Battery level (for EVs), charge cycles
- Tire pressure and wear
- Brake condition, suspension diagnostics
- Door/trunk open/close events
- Collision sensors (accelerometers)

Driver Behavior:
- Acceleration profile (aggressive or smooth)
- Cornering speed and G-force
- Speed violations
- Rapid decelerations
- Lane changes (camera-based)
- Following distance
- Distraction detection (some models)

Connectivity:
- Cell phone pairings (Bluetooth)
- WiFi networks connected to
- Call history and SMS metadata (in some models)
- VIN, registration, insurance data

Camera Footage:
- Tesla: Continuous 360° recording (dashcam and more)
- BMW, Mercedes: Selective recording in urban environments
- Cadillac, Genesis: Available on higher trims
- Footage includes: Pedestrians, other vehicles, license plates, building facades

Infotainment:
- Music and podcast listening history
- Search queries on navigation
- Contacts and call logs
- Calendar entries and meetings
- Phone contact names and relationships

Who Collects What? By Manufacturer

Tesla

Data collected - Everything. Tesla collects more per vehicle than any other manufacturer.

What they track:
- GPS coordinates every 1-5 seconds (when vehicle is on)
- Forward, rear, left, right camera footage (continuous in some modes)
- Speed, acceleration, steering angle
- Battery diagnostics and charge pattern
- Cabin camera footage (if autopilot is active)
- Thermal imaging (to detect occupants)

Real example - If you're in Autopilot, Tesla logs:
- Your exact GPS coordinates
- How many times you took manual control (and from where)
- Speed when you did
- Lane position relative to road markings
- Distances to other vehicles
- Whether you were looking at road or phone

Data sharing:
- Improves Autopilot (officially)
- Shares data with insurance partners (unofficially, per recent investigations)
- Not shared with police (officially, but subpoenas are possible)
- Historical data accessible to Tesla for ~7 years

How to disable:
```
Controls > Service > Power Off (full shutdown, 30 minutes)
Settings > Safety > Cabin Camera > Turn Off
Settings > Service > Data Sharing > Disable
Settings > Connectivity > Data Sharing > Disable
```

You can't fully disable telematics. Even with data sharing off, vehicle sends: location, diagnostics, status to Tesla servers for service alerts.

Annual data volume - ~1-2 GB per vehicle (depending on driving habits)

General Motors (OnStar)

Data collected - Location, diagnostics, emergency details

What they track (OnStar-connected vehicles):
- Real-time GPS location (sent to OnStar servers every 30-60 seconds)
- Speed and heading
- Engine diagnostics
- Collision detection
- Emergency location data (for crash response)
- Driver behavior (harsh braking, rapid acceleration)

Scary part - OnStar tracks you EVEN WHEN THE VEHICLE IS OFF

If your vehicle has OnStar and an active cellular connection, GM can locate you any time (with your phone's permission). This is designed for roadside assistance but can be abused.

Data sharing:
- OnStar sells location data to insurance companies (explicitly, with consent forms)
- Shares with law enforcement (warrant required, but compliance is assumed)
- Shares with third-party location brokers (with opt-out available, but hard to find)
- Location history retained for 3+ years

Real lawsuit - In 2023, GM settled a privacy lawsuit for $$$M because it sold customer location data despite opt-out requests being difficult to process.

How to disable:
```
1. Call OnStar: 1-866-269-6827
2. Request data sharing opt-out
3. Ask for written confirmation (they often claim there's no way to opt out)
4. Get vehicle's cellular modem disabled at service center
   - Cost: ~$200-500
   - Disables remote locking and some features
   - Permanently stops data transmission
```

Alternative - Switch to a cellular disabler device (Faraday pouch for key fob) but this affects regular operations.

Toyota

Data collected - Location, diagnostics, emergency info

Connected Services (Toyota Safety Sense):
- Real-time GPS location
- Vehicle diagnostics (battery, tire pressure)
- Emergency crash data
- Destination searches in navigation
- WiFi networks connected to
- Phone contacts and call logs

Data sharing:
- Toyota Financial Services (if you financed with them) uses location data for loan verification
- Insurance partners (Toyota Insurance Marketplace) get access to driver behavior scores
- Toyota does NOT share with third-party brokers (officially, as of 2026)
- Data retained for 2+ years

How to disable:
```
1. Settings > Connected Services > Data Sharing > OFF
2. Infotainment > Settings > Reset > Clear All (removes cached location data)
3. Don't pair smartphone via Bluetooth
   - This disables many features (messaging, calls)
4. Disable WiFi connectivity:
   Settings > WiFi > Turn Off
```

BMW, Mercedes, Audi (Luxury Segment)

Data collected - location, diagnostics, usage patterns

What they track:
- Real-time GPS with precision driving dynamics
- Autonomous emergency braking events
- Cabin microphone audio (for voice assistant, sometimes always-on)
- Driver fingerprinting (biometric patterns in steering, acceleration)
- Predictive maintenance data (ML models on engine performance)

Data sharing:
- BMW: Sells aggregated driving behavior data to insurance companies
- Mercedes: Shares with external partners via MBUX environment (Android automotive)
- Audi: Connected Services shares location with third-party vendors

How to disable:
BMW: Settings > iDrive > Driver Assistance > Ambient Data > Off
Mercedes - Settings > Digital Services > Data Collection > Off
Audi - MMI > Settings > Privacy > Data Collection > Off

Insurance Companies & Usage-Based Pricing

This is where telematics becomes financially dangerous.

How Insurance Telematics Works

The trap:
1. Insurance company offers "safe driver discount" (15-30% savings)
2. You install app or hardware OBD-II dongle
3. They track: Speed, hard braking, time of day, location
4. But here's the catch: Discounts can vanish. Rates can INCREASE.

Real example (2024-2025 data):
- You drive normally, get 20% discount for 6 months
- One trip at 48 mph in a 45 mph zone, recorded by telematics
- Insurance adjusts rate: discount drops to 10%, premium increases 5%
- New annual cost: $200+ more

Worst case - Insurer uses telematics data to deny coverage entirely.

Insurance companies using telematics:
- Geico (Driveeasier)
- State Farm (Drive Safe & Save)
- Allstate (Drivewise)
- Progressive (Snapshot)
- Metromile (by-the-mile, tracks every journey)

Cost comparison:
- No telematics tracking: $1,200/year average
- Telematics with discount: $800/year (initially)
- Telematics after 1 year of violations: $1,400/year (worst case)

How to Avoid Insurance Telematics

Option 1 - Don't install their app or device
- Most states don't require it
- You lose potential discount (15-30% savings)
- Your rate doesn't increase (stays normal)
- You pay the standard premium

Option 2 - Use telematics selectively
- Install for 6 months (get discount)
- Uninstall before rates "adjust"
- Rates lock in at discounted level (in some states)
- Risk: Company can refuse renewal

Option 3 - Request data deletion
- Contact insurer, request all telematics data be deleted
- Some comply (California CCPA requires it)
- Most don't (they claim it's business necessity)
- File complaint with state insurance commissioner if denied

Option 4 - Switch to insurers that don't offer it
- Liability-only carriers (less common)
- Some regional insurers (check your state)
- Usually 5-10% more expensive than major carriers

Federal Privacy Regulations (Emerging)

Current Status (As of March 2026)

No federal law explicitly covers vehicle telematics data (yet). This is a major loophole.

State-by-state:

California (CCPA):
- Vehicles generate "personal information" covered by CCPA
- You can request data deletion from car manufacturers
- Insurance telematics data is NOT covered (insurance is exempt)
- Right to opt-out of data sharing: Yes, but implementation varies

New York (SHIELD Act):
- Covers telematics data as "personal information"
- Manufacturers must disclose what data is collected
- Data deletion rights exist but with exceptions
- Right to opt-out: Manufacturer's choice on how to implement

Virginia (VCDPA):
- Similar to California
- Broader coverage of automated processing
- Right to know, delete, correct data
- Opt-out requirements still being determined

Federal (in progress):
- FTC issued guidance (March 2024): Automakers shouldn't collect data they don't need
- No enforcement mechanism yet
- Multiple Congressional bills proposed (haven't passed)
- Expected regulation: 2026-2027 timeframe

Legal Rights You Have Now

Right to Access - You can request all telematics data manufacturers have collected (via CCPA/VCDPA)

Right to Delete - In California, Virginia, you can request deletion (with some exceptions)

Right to Opt-Out - You can request no data sharing, but manufacturers can refuse in some cases

Filing a Complaint:
- FTC (reportidentitytheft.ftc.gov): If data misuse occurred
- State Attorney General: If state privacy law violated
- State Insurance Commissioner: If insurance company violated regulations

How to Disable Telematics (Full Guide)

For Older Vehicles (2015 and earlier)

Good news - Most telematics is absent or minimal.

Check:
- Does your vehicle have a cellular modem? (Check manual)
- Do you have a connected services subscription active?
- If no/no, you're mostly clear
- If yes, proceed to disabling

Disable:
1. Contact dealer, request cellular modem disconnection
2. Cost: $150-400
3. You lose: Remote locking, emergency assist, navigation updates
4. You keep: Local infotainment, basic features

For Current Vehicles (2020+)

Telematics is integral. You can reduce but not eliminate it.

Maximum privacy approach:

```
Step 1 - Vehicle Settings
 Disable Data Collection
   Turn off data sharing (each manufacturer)
   Turn off diagnostics sharing
   Turn off location sharing
   Disable crash data reporting
 Disable Connectivity
   Turn off WiFi
   Disable cellular data (if available)
   Don't pair smartphones via Bluetooth
   Disable voice assistants
 Disable Camera/Microphone (if available)
    Disable cabin camera
    Disable microphone for voice commands

Step 2 - At Service Center
 Request cellular modem deactivation
 Cost: $200-500
 Disables: Remote features, OTA updates, emergency assist
 Some features (airbag deployment) still collect crash data

Step 3 - Insurance
 DON'T install telematics app/device
 Accept standard premium (no discount)
 Keep 15-20% savings by not participating
```

Cost-benefit analysis:
- Service center deactivation: $300 + lost features
- Not using insurance telematics: Standard premium (no discount)
- Total "cost" of privacy: $200-300/year vs. standard rates
- Benefit: No tracking, no location data, no insurance rate surprises

For Tesla Owners (Special Case)

Tesla telematics is THE MOST extensive. You cannot fully disable it.

What you CAN do:
```
Settings > Service > Data Sharing > Disable
Settings > Safety > Cabin Camera > Off
Settings > Controls > Power Off (full system reset, doesn't disable telematics)
```

What you CANNOT do:
- Disable GPS location reporting (required for nav, emergency)
- Disable vehicle diagnostics (required for warranty service)
- Disable crash data reporting (legal requirement)
- Disable basic connectivity (car won't function)

Reality:
- Tesla can access your vehicle's location, diagnostics, and behavior ANY TIME
- They claim it's only used for service and improvements
- Data has been subpoenaed by police in criminal investigations
- Consider - Is a privacy-friendly EV available?

Alternatives to Tesla:
- Chevy Bolt EV/EUV (GM, but same OnStar issues)
- Hyundai Ioniq (moderate telematics, can be disabled)
- Kia EV6 (similar to Hyundai)
- VW ID.4 (European privacy standards, better control)

What Data Brokers Buy From Automakers

The real money - Automakers sell anonymized telematics datasets to data brokers:

Who buys:
- Google, Apple, Waze (for traffic prediction)
- Uber, Lyft (driver behavior patterns)
- Insurance companies (aggregate risk profiles)
- City planners (traffic studies)
- Retail companies (foot traffic near stores)

What they buy:
- Aggregated location data (millions of GPS points, anonymized)
- Speed patterns by road segment
- Traffic patterns by time of day
- Parking behavior (where people park, how long)
- Driving behavior profiles (aggressive vs. smooth drivers)

Cost to automakers:
- Tesla: Estimated $500M+ annually from data sales
- GM: Estimated $100M+ annually from OnStar data
- Toyota: Estimated $50M+ annually from diagnostic data

Your share of this revenue - $0. You generate the data, they profit.

Real Example - Privacy-First Driving Setup

Scenario - You want maximum privacy

Choice 1 - Used Car (2010-2015)
- No telematics, no cameras, no cellular
- Cost: $8,000-15,000
- Privacy: Excellent
- Trade-off: No safety features, older tech

Choice 2 - Modern Car (2023+) with Privacy Hardening
- Buy car (e.g., Kia EV6): $45,000
- Disable telematics at dealer: $300
- Don't install insurance app: $0
- Remove smartphone pairing: $0
- Monthly telematics data: Minimal (diagnostics only)
- Cost: $45,300 total
- Privacy: 85% (can't eliminate all)

Choice 3 - EV + VPN (Partial Solution)
- Use Faraday bag for vehicle keys (prevents remote tracking): $50
- Use WiFi hotspot with VPN (encrypts navigation queries): $0-10/mo
- Disable Bluetooth pairing: $0
- Turn off cellular if possible: $0
- Cost: $50 + $120/year
- Privacy: 60% (GPS still accessible to manufacturer)

Best realistic approach - Choice 2. Accept that modern cars collect some data, but disable what you can.

Related Articles

- [Vehicle Data Privacy Who Owns The Data Your Connected Car](/vehicle-data-privacy-who-owns-the-data-your-connected-car-co/)
- [Privacy Risks of Fitness Apps and Health Data Sharing in](/privacy-risks-of-fitness-apps-health-data-sharing-2026/---)
- [Privacy Risks of Period Tracking Apps 2026](/privacy-risks-of-period-tracking-apps-2026/)
- [Privacy Risks of AI Chatbots: Data Collection (2026)](/privacy-risks-of-ai-chatbots-data-collection-2026/---)
- [Privacy Notice Vs Privacy Policy Difference](/privacy-notice-vs-privacy-policy-difference/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
1. [Vehicle Location Privacy: GPS Spoofing and Tracking Defense](/articles/vehicle-location-privacy/)
2. [Smartphone Telematics: Location Tracking in Apps You Actually Use](/articles/smartphone-location-tracking/)
3. [Bluetooth Privacy: Pairing Risks and Device Identification](/articles/bluetooth-privacy-risks/)
4. [Home WiFi Privacy: What Your ISP and Router Know About You](/articles/home-wifi-privacy/)
5. [Opt-Out of Location Data Sales: Data Brokers Selling Your Movements](/articles/location-data-brokers/)

Built by theluckystrike. More at [zovo.one](https://zovo.one)


