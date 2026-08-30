# TryHackMe — Missing Person (OSINT) Writeup

**Room:** Missing Person
**Category:** OSINT / Open-Source Intelligence
**Author:** OD1Nn00b

---

## Executive Summary

The "Missing Person" room presents a realistic OSINT scenario: a person went on holiday, sent two photos and a short message to a friend, and then disappeared. No tools or hints are provided — only the two images and the message text. The objective is to reconstruct the missing person's location, timeline, and social connections using pure open-source investigation techniques (reverse image search, metadata analysis, and social media enumeration), ultimately identifying the event, the venue, a contact (a local DJ), and a phone number tied to the case.

This room tests methodical OSINT reasoning rather than tool proficiency — success depends on cross-referencing small visual details against public information sources and verifying each lead before building on it.

---

## Reconnaissance

**Provided artifacts:**
- Two images shared by the missing person
- A short message: *"went on holiday and shared some photos, haven't heard from him since"*

**Initial observations:**
- Image 1: an indoor scene with visible text on a table surface
- Image 2: what appears to be a motorsport racing circuit

**Investigation plan:**
1. Reverse image search each photo to identify location/context
2. Correlate location with a real-world event and date
3. Extract any embedded metadata (EXIF) from the images
4. Use a follow-up message from the missing person to identify further leads (venue, contact, destination)
5. Cross-reference each lead against public sources (search engines, social media) before treating it as confirmed

---

## Investigation Phases

### Phase 1 — Reverse Image Search (Circuit Identification)
- **Action:** Ran the racing-circuit image through Google Images reverse search
- **Command/Tool:** `Google Image Search` (upload/search-by-image)
- **Result:** Identified the venue as the **Pertamina Mandalika International Street Circuit**, Indonesia
- **Outcome:** Established country and general context (a racing event)

### Phase 2 — Event Identification
- **Action:** Noted motorcycles in the image, narrowing the event type to MotoGP
- **Command/Tool:** Search query — `Mandalika MotoGP 2025 date`
- **Result:** Confirmed event dates: **03–05 October 2025**
- **Outcome:** Location + timeline both established

### Phase 3 — First Image Detail Analysis
- **Action:** Closely inspected the first photo for readable text/objects
- **Result:** Identified restaurant name on the table — **Cantina Mexicana** — confirmed as a real, searchable venue
- **Outcome:** Established a second physical location tied to the person, likely near the event

### Phase 4 — Metadata Extraction
- **Action:** Uploaded the image to an EXIF viewer to check for embedded metadata
- **Command/Tool:** `ezgif.com` (metadata/EXIF extraction)
- **Result:** Extracted `Date/Time Original: 19:55:30`
- **Outcome:** Added a precise timestamp to the emerging timeline

### Phase 5 — Message Analysis (New Leads)
- **Action:** Parsed the missing person's last message for actionable clues
- **Result:** Identified three new investigation targets:
  - An after-party bar/venue
  - A local DJ met at the party
  - A planned cave visit
- **Outcome:** Shifted investigation from image analysis to targeted OSINT searches

### Phase 6 — Venue (Bar) Identification
- **Action:** Iterated through multiple targeted search queries combining event, location, and date terms
- **Command/Tool:** Search engine queries (multiple phrasing attempts)
- **Result:** Identified address — **Jl. Raya Kuta, Kuta, Kec. Pujut, Kabupaten Lombok Tengah, Nusa Tenggara Barat**
- **Outcome:** Confirmed the after-party location; noted this required iteration rather than a single search hit

### Phase 7 — DJ Identification
- **Action:** Cross-referenced the venue result with event/lineup information
- **Result:** Identified the local DJ as **Bong Leleh**
- **Outcome:** Named contact tied to the missing person's last known social interaction

### Phase 8 — Cave Identification
- **Action:** Initial guess based on geographic proximity was incorrect; pivoted to AI-assisted search for verification
- **Command/Tool:** Gemini (Google AI search) for refined querying
- **Result:** Correct location identified as **Gua Sumur**
- **Outcome:** Reinforced the importance of verifying rather than assuming based on proximity alone

### Phase 9 — Contact Number Discovery
- **Action:** Searched for the DJ's contact details, including in the local language, and went beyond top search results into social media
- **Command/Tool:** Facebook profile search
- **Result:** Located a phone number in Indonesian international format (+62…), normalized to **085333137345**
- **Outcome:** Final flag/answer obtained; investigation complete

---

## Root Cause / Key Techniques Table

| Investigation Gap | Technique Used | Result |
|---|---|---|
| Unknown location in Image 2 | Reverse image search | Identified Mandalika circuit, Indonesia |
| Unknown event/date | Contextual search (bikes → MotoGP) | Confirmed event dates (Oct 2025) |
| Unclear venue in Image 1 | Close visual inspection of text | Identified restaurant (Cantina Mexicana) |
| No visible timestamp | EXIF/metadata extraction | Recovered exact photo timestamp |
| Cave misidentified on first guess | AI-assisted refined search instead of assumption | Corrected to Gua Sumur |
| Contact number not in top search results | Deeper search + social media (Facebook) enumeration | Recovered phone number |

---

## Lessons Learned

- **Reverse image search first:** it's often the fastest way to anchor an unknown image to a real-world location.
- **Verify, don't assume:** the cave step showed that a "reasonable guess" based on geography can be wrong — every lead needs independent confirmation.
- **Small visual details matter:** text on a table (restaurant name) was as valuable as the larger scene context.
- **Don't stop at page one:** the phone number was only found by going past top search engine results into social media.
- **OSINT is iterative:** several steps (bar identification especially) required multiple reworded queries before landing on the right combination of terms.

---

*Writeup based on personal completion of the TryHackMe "Missing Person" room, structured from techniques referenced in a public walkthrough by Soumodeep Das (Medium).*
