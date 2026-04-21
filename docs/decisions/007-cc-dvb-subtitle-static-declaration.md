# ADR-007: CC / DVB Subtitle Support via Static Stream Declaration

**Date**: 2026-04-21
**Status**: Accepted
**Affects**: `XtreamTunerHost.CreateMediaSourceInfo()`, `PluginConfiguration`, Live TV config UI

---

## Context

Users watching DVB broadcasts expect CC / DVB subtitle tracks to be selectable in Emby.
The plugin disables stream probing by design to enable fast channel switching —
`AnalyzeDurationMs = 0` and `SupportsProbing = false` are set for Dispatcharr proxy URLs.
Without probing, Emby never discovers subtitle tracks embedded in the MPEG-TS stream, so
no subtitle options appear in the player UI.

## Problem

How do we surface CC / DVB subtitle tracks in Emby without re-enabling stream probing?

## Alternatives Considered

### 1. Re-enable probing when subtitles are requested

Set `AnalyzeDurationMs > 0` when a subtitle option is enabled.

**Rejected.** For Dispatcharr proxy URLs this is destructive: the probe opens a short-lived
connection, Dispatcharr interprets the close as the last client leaving and tears down the
channel, causing a retry storm. See CONTRIBUTING.md and ADR-001 for the full explanation.

### 2. Fetch subtitle track info from Dispatcharr stream stats

Dispatcharr's `/api/channels/channels/?include_streams=true` already provides per-stream
codec metadata via Streamflow. The plan was to extend `StreamStatsInfo` with subtitle fields
and use them to declare tracks with correct language tags.

**Rejected.** Querying the live stats confirmed that Dispatcharr does not expose subtitle
or language metadata — all fields were absent across 150 streams. Streamflow only captures
video and audio codec info.

### 3. Manual language configuration

Let users configure a language code (e.g. `nor`) to attach to declared subtitle tracks.

**Rejected.** Adds UI complexity for marginal benefit. The tracks function correctly without
a language tag; they simply show as "DVB Subtitles" rather than e.g. "Norwegian (DVBSUB)".

## Decision

Declare `dvb_subtitle` `MediaStream` entries statically in `CreateMediaSourceInfo()` when
the `DeclareDvbSubtitles` config flag is set. This requires no probing and preserves fast
channel switching.

A single toggle — **"Enable CC / DVB subtitles"** — is added to the Live TV settings tab,
outside the Dispatcharr section so it applies to both direct Xtream and Dispatcharr users.

## Consequences

- CC / DVB subtitle tracks appear in Emby without any probing overhead.
- Fast channel switching is preserved.
- Tracks show without a language name since the stream source does not provide this.
- If Dispatcharr adds language metadata in a future release, language inference can be
  added without changing the overall approach.