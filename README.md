# Mamdani Dance Floor

A full-screen party visual. Put it on the TV, play music out loud, and he dances to it.

- **Beat sync** — locks to Spotify's own beat grid when available, otherwise listens
  to the room through the mic, otherwise runs an internal clock. The SYNC chip in the
  corner tells you which. `[` and `]` nudge the timing, `\` resets.
- **Spotify readout** — album art, title, artist and progress along the top.
  Connect once and this device reconnects by itself afterwards.
- **Nine backdrops**, each based on something that actually happened, cycling every 24s.

## Keys

`F` fullscreen · `N` next scene · `B` previous · `A` toggle auto-cycle
`space` tap tempo · `[` `]` nudge sync · `\` reset nudge · `S` Spotify setup · `H` hide the HUD

## Connecting Spotify

1. developer.spotify.com/dashboard → **Create app**
2. Redirect URI: the URL this page is served from. Open the page, click
   **connect spotify**, and use the COPY button so it matches exactly.
3. Tick **Web API**, save, copy the **Client ID**, paste it into the page.

Everything runs in the browser. No server, no analytics, mic audio never leaves the page.
