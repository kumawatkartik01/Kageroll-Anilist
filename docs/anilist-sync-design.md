# AniList sync: implementation design proposal

**Status:** exploratory proposal for issue #6  
**Scope:** architecture and questions to resolve before implementation; this document does not claim that the current Kageroll patch framework already exposes the required UI or playback hooks.

## Goal

Offer an opt-in way for a signed-in AniList user to update anime episode progress while watching Crunchyroll in the patched Android app.

Kageroll is currently a Morphe patch bundle focused on Crunchyroll. The existing patch set demonstrates bytecode-level changes and an extension module, but the repository does not currently contain an AniList client or a general account/settings UI. The design should therefore begin by proving what can be safely integrated into the patched app.

## Suggested user flow

1. The user explicitly enables AniList sync and chooses **Connect AniList**.
2. Authentication takes place through the system browser using an OAuth flow suitable for a public Android client. No client secret is embedded in the APK.
3. The user confirms the Crunchyroll-to-AniList series match when it is ambiguous.
4. After an episode is watched sufficiently, Kageroll offers or performs an opt-in progress update.
5. The user can pause sync, change the match, disconnect, and clear locally stored credentials.

The feature should be disabled by default. It should not upload viewing history without an explicit opt-in.

## Authentication

AniList documents OAuth and authenticated GraphQL requests:

- Authentication guide: https://docs.anilist.co/guide/auth/
- GraphQL guide: https://docs.anilist.co/guide/graphql/

The implementation should use a public-client flow that does not require shipping a client secret. The exact flow and redirect mechanism must be validated against AniList's current documentation and Android's supported callback mechanisms before coding.

Questions to resolve:

- Can a Kageroll patch reliably add the required manifest intent filter / deep-link callback?
- Can the patched app launch a browser and receive the callback without conflicting with Crunchyroll's own activities?
- Where can credentials be stored in app-private storage, and can storage be protected with Android Keystore-backed encryption in this runtime?
- What is the clean logout and token-revocation/expiry behavior? AniList's documented token lifetime and refresh behavior should be handled explicitly rather than assuming refresh tokens exist.

Never log access tokens or include them in diagnostics.

## Playback metadata and episode detection

The most important technical dependency is obtaining trustworthy playback metadata.

The existing playback-speed patch locates Crunchyroll player settings through bytecode fingerprints. That does **not** establish that episode identifiers, series identifiers, or playback completion state are currently available to Kageroll.

Before implementing sync, inspect the target Crunchyroll versions and identify a stable internal source for:

- Series identifier and title
- Season / episode identifier and episode number
- Playback position and duration, if exposed
- A reliable signal that playback has ended or crossed a completion threshold

Prefer structured player/model data over scraping visible text from the UI. Do not infer an episode number from a title string when a stable identifier is available. Fingerprints should fail safely when the expected method or data shape changes.

## Crunchyroll-to-AniList matching

Crunchyroll seasons and AniList media entries do not always map one-to-one. Alternate titles, split cours, specials, movies, and sequel seasons can make automatic matching ambiguous.

A resolver should:

1. Search AniList using the series title and known aliases.
2. Present candidates when confidence is low.
3. Persist the user's confirmed mapping locally.
4. Keep episode numbering / season offsets explicit in the mapping.
5. Never silently update a different anime because a title match was approximate.

The mapping should be editable and removable by the user.

## Progress updates

AniList exposes a GraphQL API at https://graphql.anilist.co. The implementation should use documented queries and mutations, with the authenticated user's identity checked before writing.

Recommended behavior:

- Update progress only after a configurable completion threshold or a confirmed episode-end event.
- Debounce repeated events so seeking, rotation, process recreation, or player callbacks cannot send duplicate writes.
- Do not reduce an existing AniList progress value automatically.
- Handle specials and non-standard episode numbering as unresolved cases until a mapping is confirmed.
- Queue or retry transient network failures conservatively; never retry indefinitely.
- Handle authentication expiry, rate limits, and GraphQL errors without breaking playback.
- Keep sync independent from the playback path so an AniList outage cannot stop video playback.

The exact GraphQL schema fields and mutation behavior should be verified against the current AniList API before implementation.

## Privacy and controls

- Off by default; clear opt-in before any progress is sent.
- Explain what data is sent (AniList media ID and episode progress).
- Provide pause, disconnect, and clear-mapping controls.
- Do not collect or transmit viewing history beyond the progress update the user requested.
- Do not write tokens, account identifiers, or viewing details to logs.
- Avoid background network activity when sync is disabled.

## Integration questions for maintainers

These need investigation before a code implementation can be responsibly proposed:

1. What supported Morphe extension mechanism can provide settings or an account-connection UI?
2. Can a patch add a browser OAuth callback safely to the target Crunchyroll manifest?
3. Which Crunchyroll classes/methods expose stable episode metadata and playback completion?
4. Can the extension module host networking and secure credential storage, or must some responsibilities live in the patched app?
5. What is the project's preferred approach for version-specific fingerprints and tests?

## Suggested implementation sequence

1. **Feasibility spike:** identify playback metadata and confirm what UI / manifest hooks Morphe supports. No AniList writes yet.
2. **Read-only prototype:** implement OAuth and an authenticated AniList viewer/media lookup, with tokens kept out of logs.
3. **Mapping prototype:** let the user confirm and persist one Crunchyroll-to-AniList mapping.
4. **Progress prototype:** update a test account only after explicit user action; test duplicate events and failure cases.
5. **Opt-in release:** add clear controls, privacy text, version compatibility checks, and regression tests.

This staged approach keeps the first contribution honest: authentication, metadata access, and patch-framework integration are separate unknowns that should be demonstrated before promising automatic synchronization.
