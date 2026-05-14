---
layout: default
title: "Detecting Impossible Travel with the Haversine Formula"
permalink: /tech-learning/impossible-travel-detection/
description: "Building a detection rule for impossible travel in Microsoft Sentinel using the Haversine formula and KQL."
---

<h1>{{ page.title }}</h1>
<p class="subtitle">May 2026 &middot; {% include reading-time.html %}</p>

Impossible travel is one of those detections that sounds straightforward on paper - two logins from geographically distant locations within a timeframe no human could physically cover - but building a reliable version that does not drown your analysts in false positives requires more thought than simply flagging logins from different countries.

This article walks through building an impossible travel detection rule in Microsoft Sentinel using KQL, implementing the Haversine formula to calculate great-circle distances between login locations, and layering in the speed threshold and tuning logic that makes the rule usable in practice.

## Why Build Your Own?

Microsoft Entra ID ships an impossible travel risk detection as part of Identity Protection. So why roll your own?

A few reasons worth considering:

- **Tuning control** - the built-in detection is a black box. You cannot adjust the speed threshold, minimum distance filter, or exclusion logic without a P2 licence, and even then options are limited.
- **Enrichment** - a custom rule lets you append whatever context matters: user risk score, device compliance state, the specific application accessed.
- **Detection-as-code** - storing the rule in your own detection library means it goes through your review and testing process, not a vendor's update cycle. You own the logic.

## The Haversine Formula

The core challenge is calculating the distance between two GPS coordinates accurately. A naive approach - treating latitude and longitude as flat Cartesian coordinates - works near the equator but introduces significant error at higher latitudes. The correct approach is to account for the curvature of the Earth by calculating the **great-circle distance**: the shortest path between two points on the surface of a sphere.

The Haversine formula does exactly this:

```
a = sin²(Δlat/2) + cos(lat₁) · cos(lat₂) · sin²(Δlon/2)
c = 2 · atan2(√a, √(1−a))
d = R · c
```

Where:
- **Δlat** and **Δlon** are the differences in latitude and longitude, converted to radians
- **R** is the mean radius of the Earth - 6,371 km
- **d** is the resulting great-circle distance in km

The name comes from the *haversine function*: `hav(θ) = sin²(θ/2)`, a trigonometric identity that keeps values numerically stable for both small and large distances.

As a concrete example: London (51.5°N, 0.1°W) to New York (40.7°N, 74.0°W) gives a great-circle distance of approximately 5,570 km. Commercial aircraft cruise at around 900 km/h, meaning a legitimate traveller could cover that route in roughly six hours. Any login pair showing that distance in under six hours warrants investigation.

## Implementing Haversine in KQL

KQL exposes the trigonometric functions needed - `sin()`, `cos()`, `atan2()`, `sqrt()`, and `radians()` - so the formula translates directly. We define it as a reusable scalar function using a `let` statement at the top of the query:

```kql
let haversine = (lat1:real, lon1:real, lat2:real, lon2:real) {
    let R = 6371.0;
    let dLat = radians(lat2 - lat1);
    let dLon = radians(lon2 - lon1);
    let a = sin(dLat / 2) * sin(dLat / 2)
          + cos(radians(lat1)) * cos(radians(lat2))
          * sin(dLon / 2) * sin(dLon / 2);
    2 * R * atan2(sqrt(a), sqrt(1 - a))
};
```

This returns the distance in kilometres between two coordinate pairs. KQL does have a native `geo_distance_2points()` function that achieves the same result - but implementing Haversine explicitly makes the logic fully auditable and portable to any SIEM or data platform that supports basic trigonometry.

## The Detection Rule

With the distance function defined, the detection logic follows a clear pattern:

1. Pull successful sign-ins from `SigninLogs`
2. For each user, order events by time and compare each login to the previous one
3. Calculate distance (Haversine) and time delta between the pair
4. Derive the implied travel speed: `distance / time`
5. Alert where speed exceeds the threshold

```kql
let max_speed_kmh  = 900.0;   // Approximate max commercial aircraft cruising speed
let min_distance_km = 100.0;  // Filter GPS noise and same-city geolocation variance
let haversine = (lat1:real, lon1:real, lat2:real, lon2:real) {
    let R = 6371.0;
    let dLat = radians(lat2 - lat1);
    let dLon = radians(lon2 - lon1);
    let a = sin(dLat / 2) * sin(dLat / 2)
          + cos(radians(lat1)) * cos(radians(lat2))
          * sin(dLon / 2) * sin(dLon / 2);
    2 * R * atan2(sqrt(a), sqrt(1 - a))
};
SigninLogs
| where TimeGenerated > ago(1d)
| where ResultType == 0                              // Successful sign-ins only
| where isnotempty(LocationDetails.geoCoordinates.latitude)
| project
    TimeGenerated,
    UserPrincipalName,
    Latitude  = toreal(LocationDetails.geoCoordinates.latitude),
    Longitude = toreal(LocationDetails.geoCoordinates.longitude),
    City      = tostring(LocationDetails.city),
    Country   = tostring(LocationDetails.countryOrRegion),
    IPAddress,
    AppDisplayName
| sort by UserPrincipalName asc, TimeGenerated asc
| serialize
| extend
    PrevTime    = prev(TimeGenerated, 1),
    PrevLat     = prev(Latitude, 1),
    PrevLon     = prev(Longitude, 1),
    PrevCity    = prev(City, 1),
    PrevCountry = prev(Country, 1),
    PrevUser    = prev(UserPrincipalName, 1)
| where UserPrincipalName == PrevUser                // Scope comparisons to same user
| extend TimeDiffHours = datetime_diff('minute', TimeGenerated, PrevTime) / 60.0
| where TimeDiffHours > 0                            // Exclude simultaneous sessions
| extend DistanceKm = haversine(PrevLat, PrevLon, Latitude, Longitude)
| where DistanceKm > min_distance_km
| extend ImpliedSpeedKmh = DistanceKm / TimeDiffHours
| where ImpliedSpeedKmh > max_speed_kmh
| project
    TimeGenerated,
    UserPrincipalName,
    PreviousLocation = strcat(PrevCity, ", ", PrevCountry),
    CurrentLocation  = strcat(City, ", ", Country),
    DistanceKm       = round(DistanceKm, 0),
    TimeDiffHours    = round(TimeDiffHours, 2),
    ImpliedSpeedKmh  = round(ImpliedSpeedKmh, 0),
    IPAddress,
    AppDisplayName
| sort by ImpliedSpeedKmh desc
```

A few implementation notes worth calling out:

- `serialize` is required before `prev()` - it makes row order deterministic within the query so the windowing function behaves correctly.
- Filtering on `ResultType == 0` keeps focus on successful authentications. Failed attempts can be layered in separately if hunting for credential stuffing alongside account compromise.
- The `min_distance_km` of 100 km absorbs IP geolocation inaccuracy - most providers carry a margin of 50+ km, so filtering below that threshold generates noise rather than signal.

## Tuning Considerations

No detection rule ships production-ready on day one. Impossible travel generates a predictable set of false positives worth building exclusions for from the start.

**VPN and proxy traffic** - a user in London connecting through a US-based VPN exit node will appear to teleport. Cross-referencing `IPAddress` against a known VPN/proxy watchlist, or filtering on `IsCompliant` device state, reduces this significantly.

**Shared accounts and service accounts** - automation runs, shared mailboxes, and break-glass accounts will trigger this rule reliably. Exclude them by UPN suffix or a dedicated Sentinel watchlist so analysts do not waste time on known false positives.

**Legitimate fast travel** - senior executives on private jets can legitimately exceed 900 km/h, particularly on short-haul routes where the ratio of flight speed to total elapsed time is compressed by check-in, boarding, and transit time. Consider a higher threshold for specific user groups, or a minimum `TimeDiffHours` floor to avoid penalising short windows.

**Geolocation provider variance** - the same physical device can resolve to different coordinates across sign-ins depending on which IP geolocation database Entra ID used at the time of lookup. The 100 km minimum handles most of this, but reviewing low-distance, high-speed alerts in the first week of deployment is worthwhile before enabling auto-close.

## MITRE ATT&CK Mapping

| Tactic | Technique | Relevance |
|--------|-----------|-----------|
| Initial Access | [T1078 - Valid Accounts](https://attack.mitre.org/techniques/T1078/) | Stolen credentials used to authenticate from a remote location |
| Defence Evasion | [T1078.004 - Cloud Accounts](https://attack.mitre.org/techniques/T1078/004/) | Compromised cloud identity used to access SaaS applications |
| Lateral Movement | [T1550.001 - Application Access Token](https://attack.mitre.org/techniques/T1550/001/) | Token replay across geographically separated sessions |
