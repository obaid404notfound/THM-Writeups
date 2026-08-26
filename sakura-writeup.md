# TryHackMe — Sakura Room Writeup

**Category:** OSINT
**Author:** OD1Nn00b

> Flags are intentionally omitted below — this walkthrough covers the methodology only. Try each step yourself before reading further.

## Task 1 — Introduction

Just a read-through task to confirm you've gone over the room's background story before starting the investigation.

## Task 2 — Tip-Off

The room gives you an image tied to the initial "attack." The first move is to look past the visible content of the image itself — there's a binary string hidden in the background that decodes to a hint pointing at metadata rather than the picture itself.

Since the file is an SVG, metadata isn't buried in EXIF the way it would be for a JPEG — it's sitting in the markup. Right-clicking the image in-browser and inspecting the element exposes the raw SVG source, which includes authoring details and a file path. That path leaks a username, which becomes the pivot point for the rest of the investigation.

**Technique used:** SVG source inspection via browser dev tools.

## Task 3 — Reconnaissance

With a username in hand, the next step is classic OSINT username-correlation: run it through a tool like WhatsMyName to check which platforms have an account under that same handle. This turns up a GitHub profile.

From there, digging through the account's repos for anything crypto/PGP-related pays off — one repo holds a public PGP key. Extracting the email tied to that key (via `gpg --import` or a GUI tool like Kleopatra) gives you the attacker's email address.

For the attacker's real name, GitHub alone isn't enough — a plain Google search for the username surfaces a public LinkedIn profile with the full name.

**Techniques used:** Username enumeration (WhatsMyName), GitHub repo review, PGP key metadata extraction, search-engine pivoting.

## Task 4 — Unveil

This task is about digging into version history rather than current state. The GitHub account has a repo referencing a mining script — the current version has been edited, but GitHub keeps commit history. Pulling up the original ("create") commit reveals the unedited file, which contains a cryptocurrency wallet address and the mining pool it was tied to.

From there, plugging the wallet address into a blockchain explorer for that currency surfaces the full public transaction history — timestamps, counterparties, and (for well-known addresses) labels identifying mining pools or exchanges the attacker interacted with.

**Techniques used:** Git commit history review, blockchain explorer analysis.

## Task 5 — Taunt

A screenshot of a conversation is provided as the new lead. The username shown in it doesn't match the attacker's current handle, so the play is to search that old username on Twitter/X to find an early post where they introduce their new handle — giving you their current account.

From there, scrolling their timeline turns up a post referencing saved WiFi credentials, along with an MD5 hash and a cryptic reference to a "PASTE" site on the "DEEP" web — a nod to the Tor-hosted paste site DeepPaste. Searching that hash on DeepPaste (via Tor Browser) pulls up the page where SSIDs and passwords were dumped.

With an SSID in hand, cross-referencing it on Wigle.net (a wardriving/WiFi-mapping database) locates the physical network and reveals its BSSID.

**Techniques used:** Social media pivoting, dark web / Tor paste-site searching, WiFi geolocation via Wigle.

## Task 6 — Homebound

The final task is pure geolocation chained across several Twitter posts:

1. A "last cherry blossoms before heading home" photo needs to be geolocated using visible landmarks (a monument, water, and rail infrastructure) — placing it near a specific airport.
2. A "final layover" photo taken inside an airport lounge can be reverse-image-searched (Google Images) to identify the lounge and, from there, the airport.
3. A satellite/map screenshot posted mid-flight shows a lake, which — combined with the flight's northward direction from the layover airport — narrows down the destination region.
4. Finally, cross-referencing the WiFi SSIDs recovered in Task 5 (Home WiFi, a McDonald's, and a "City Free WiFi") on Wigle.net shows all three clustering in the same city, confirming the attacker's home city.

**Techniques used:** Photo geolocation from landmarks, reverse image search, WiFi-based geolocation triangulation.

---

## Key Takeaways

- OSINT investigations chain small leaks together — a username, a stale commit, an old handle — none of which is damning alone.
- Metadata (SVG source, PGP keys, EXIF-equivalent data) is often a bigger leak than the visible content itself.
- Version control history (Git) can resurrect "deleted" or edited sensitive data.
- WiFi SSID databases like Wigle are an underrated geolocation tool when GPS/EXIF data isn't available.
- OPSEC failures usually come from username reuse across platforms with very different privacy postures (GitHub vs. Twitter vs. LinkedIn).
