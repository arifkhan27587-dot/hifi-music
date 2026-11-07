# Tryptify modding session: progress log

Project: /root/tryptifi (extracted from /sdcard/tryptifi hifi/Tryptifi-source-code.zip). Not a git repo.
Owner's phone: Moto G96 5G, Android 16. Owner talks in Hinglish.
Owner has subscriptions: TIDAL, Qobuz, Apple Music, Deezer, Spotify.

## Build command (works on this phone, low memory)
```
cd /root/tryptifi && ./gradlew --no-daemon -Pkotlin.compiler.execution.strategy=in-process \
  "-Dorg.gradle.jvmargs=-Xmx2600m -XX:+UseSerialGC -XX:MaxMetaspaceSize=512m -XX:ReservedCodeCacheSize=128m -Dfile.encoding=UTF-8" :app:compileDebugKotlin
```
Final APK: use build type `benchmark` (release-optimized, signed with app/monotrypt-debug.keystore),
`:app:assembleBenchmark`. No release keystore available. Copy APK to `/sdcard/tryptifi hifi/`.

NOTE 2026-11-04: Xmx3200m got Termux/proot killed by Android (OOM) mid compileDebugKotlin. Use 2600m + SerialGC,
and avoid heavy work while compiling. HiFiApiClient duplicate block (conflicting overloads) already fixed.

## Decisions told to the owner
- No DRM circumvention: Spotify / Apple Music give login + library + catalogue; audio plays from
  TIDAL/Qobuz by ISRC. Do not extend the Apple "wrapper decrypt" path.
- Device testing is the owner's job (DLNA, logins, DAC); say clearly what is unverified.

## Task list (owner's request order)
1. [done, compiled] DLNA/UPnP for BubbleUPnP / mConnect
   - New dlna/DlnaRenderer.kt (persistent MediaController, real transport state, GENA SUBSCRIBE/NOTIFY,
     SetNextAVTransportURI, volume/mute), DlnaServer.kt request parsing by bytes (Unicode fix),
     SOAPACTION, XML unescape, full SCPDs, renderer ConnectionManager Sink protocols,
     stable UUIDs (SharedPreferences dlna_identity), network-change restart, SSDP per-requester IP, 3x announce.
2. [done, compiled] Logins
   - data/auth/TidalAuth.kt: correct endpoint /device_authorization, polling errors, refresh, expiry,
     account country (was hard-coded US), optional client secret; HiFiApiClient uses it + 401 retry.
   - Qobuz: POST login, token verified via user/get, saves the app id that works, official web login
     (ui/settings/OfficialLoginWebDialog.kt, localStorage token capture).
   - Deezer: ARL verified via deezer.getUserData, official web login capturing `arl` cookie.
   - Spotify: data/auth/SpotifyAuth.kt PKCE (tryptify://spotify-callback, manifest + MainActivity wired,
     SpotifyApiClient uses it). DONE: Spotify dialog UI + honest account card texts.
   - TODO: Apple Music dialog: honest text (token saved but catalogue uses iTunes public API).
3. [done, compiled] Fidelity + Spectro on player
   - SourceAuthenticityAnalyzerTap: occupancy/persistence cutoff detection added, spectrogram 128x220.
   - DONE: TrueAudioBadges.kt 5-tab inspector (Fidelity chain, Lossless, Spectro w/ kHz axis + cutoff line,
     Output, Loudness w/ PLR). ResonadaAudioBadges.kt is dead code (unused).
4. [done, compiled] Lyrics Apple-style: LyricLine.agent/background/endMs, TTML parser
   (ttm:agent duet, x-bg separate), LyricsHero: duet left/right, bg vocals under line, word lift/swell,
   depth blur (API 31+), owner asked: sung word brightest -> soft-feather fill, upcoming 30% alpha.
   CHECK: TtmlLyricsParser unit tests may expect x-bg merged into lead words -> update tests.
5. [done, compiled] Artist catalogue: ArtistDetail.compilations/appearsOn + screen sections
   (strings artist_compilations/artist_appears_on in strings.tsv), TIDAL paged + filters and direct-first
   when signed in, Spotify `next` paging all groups, Deezer index paging, Qobuz limit 500, Apple 200.
   (old audit: (audit: TIDAL limit=50 no paging, no EP/compilation
   filters; Qobuz limit=50; Deezer limit=50 no index; Spotify limit=30 no next; Apple limit=50).
6. [done, compiled] USB DAC: feedback scaled per packet (bInterval>1) w/ nominal check,
   UAC2 non-PCM (DSD/raw) alts skipped (bmFormats), float->int bit-exact (x2^(n-1), dither only if needed;
   FloatToSubslotSampleTest rewritten), BitPerfectEvaluator reports software DAC level + rate null not locked,
   bit ladder 24,32,16 for 32-bit-only DACs. NOT done (bigger): hardware Feature-Unit volume, gapless on
   rate change (drains ring), GET_CUR readback, A14 mixer attr per track, DoP/native DSD.
7. [done, compiled] Cast button (dlna/UpnpCastManager.kt): discovery + playback improvements.
8. [done: assessed, no code change] Dolby Atmos. Already present: E-AC-3 JOC object renderer (cpp/atmos +
   Cavern port, binaural HRTF / VBAP 5.1-7.1.4), FFmpeg decode, Direct mode -> platform Dolby decoder +
   Spatializer, HDMI passthrough via DefaultAudioSink.Builder(context), local detection (MediaStore mime,
   TagReader, "Dolby Atmos" text), TIDAL Atmos (direct token + server). Limits told to owner: AC-4 only via
   platform decoder; TrueHD Atmos = 7.1 bed only; Apple Music Atmos is DRM -> not done; Qobuz/Deezer have no Atmos.
9. [DONE 15:53] Tryptify-mod-2026-11-04.apk (benchmark, debug-key signed). Build benchmark APK, copy to /sdcard/tryptifi hifi/.
10. [DONE] Tryptify-mod-source-2026-11-04.zip. Zip full modified source (exclude build/, .gradle/, .kotlin/) to /sdcard/tryptifi hifi/.

## Login audit facts (from subagent)
- Deezer public api.deezer.com ignores ARL cookie; streaming via third-party server only.
- Apple user token is saved and never sent; search uses itunes.apple.com.
- Spotify sp_dc exchange endpoint now needs TOTP -> likely broken; PKCE added instead.
- getArtist falls through Deezer->Apple->Qobuz with the same numeric id -> can show wrong artist.

NOTE: at 15:53 /sdcard/tryptifi hifi/ was EMPTY before copy: original Tryptifi-source-code.zip no longer there.
Next ideas offered (not started): "Open in Apple Music" button for Atmos tracks missing on TIDAL.

## ROUND 2 (2026-11-04 evening): owner's bug report
Sources: /sdcard/tryptify eror/*.png (14 screenshots), /sdcard/Download/monotrypt-crash-20261104-21363x.log.txt
Owner said: remove Apple Music (OK'd) or give DRM-bypass login -> DRM bypass DECLINED, Apple Music to be removed.
Fix ONLY these, then rebuild benchmark APK -> /sdcard/tryptifi hifi/.
- [ ] B1 CRASH: ForegroundServiceStartNotAllowedException (media3 MediaNotificationManager.startForeground) when
      BubbleUPnP -> DMR SetAVTransportURI/Play while app in background. PlaybackService.
- [ ] B2 DLNA renderer: BubbleUPnP plays to "Tryptify Renderer" but Tryptify doesn't show same cover/metadata;
      mConnect: connects but song does not play at all.
- [ ] B3 Remove Apple Music everywhere (search source chip, account card/login, artist fallback, catalogue).
- [ ] B4 Artist page opened from Deezer/Qobuz shows Apple Music 30s previews as Top tracks (Sia/Deezer, Arijit/Qobuz).
      Player shows "LOSSY STREAM · ALAC 24/44.1k" for those previews.
- [ ] B5 TIDAL search = "No results found" (owner thinks TIDAL logs out by itself). "official site connects, then
      'edit nahi aata'" (unclear; likely nothing shows after login).
- [ ] B6 Deezer search (Tracks + Deezer filter) = "No results found".
- [ ] B7 Logins slow/sometimes fail. Qobuz email login error: "Invalid or missing app_id parameter" (app id field
      798273057). Qobuz official sign-in WebView = blank page.
- [ ] B8 Lyrics: some lines go slightly off screen (right-aligned duet lines + word swell).
- [ ] B9 Fidelity + Spectro inspector slow.
- [ ] B10 (owner, later msg): for ANY service, artist page must show that same service's full catalogue
      (top songs, albums, singles/EPs, compilations, everything), "like mConnect". No cross-service/Apple fallback.
- [ ] B11 (owner): check BOTH directions of UPnP/DLNA: renderer (receive from BubbleUPnP/mConnect) AND
      cast/server (Tryptify sending to other renderers, UpnpCastManager + media server).
FINDINGS so far:
- B1 cause: control point on SAME phone => Tryptify always in background when Play arrives; PlaybackService not
  foreground => startForegroundService denied; media3 1.5.1 catches sync path but async bitmap callback path
  (DefaultMediaNotificationProvider.OnBitmapLoadedFutureCallback) is uncaught => crash. Also Android 15+ denies
  audio focus without FGS => silent. dlnaRendererEnabled defaults TRUE (PreferencesManager:1086).
  PLAN: (a) wrap notification provider callback in try/catch; (b) PlaybackService keeps itself foreground
  ("renderer hold") while renderer enabled: start at onCreate when app visible, override onUpdateNotification to
  force foreground + show renderer-ready notif (id 1001, channel default_channel_id) when no items; onTaskRemoved
  no stopSelf while hold.
- B2 cause: DlnaRenderer sets MediaItem straight on player; UI reads QueueManager.currentTrack => old song/cover.
  Also PlaybackService.onTrackEnded()/syncGaplessNext() advance APP queue after a DLNA item.
  PLAN: DlnaRenderer sets QueueManager to a synthetic Track (title/artist/cover from DIDL); PlaybackService skips
  onTrackEnded/gapless for mediaId "dlna-dmr:"; set MIME for DASH/HLS from DIDL protocolInfo.
- B4/B10 cause: HiFiApiClient.getQobuzArtist direct path (~L1527) fills topTracks from searchItunesArtistDirect (Apple 30s);
  getDeezerArtist direct path (~L1233) fills from iTunes when Deezer empty; ArtistDetailViewModel falls back to
  APPLE + iTunes; getAlbum (~L1802) iTunes album fallback. PLAN: remove all; Qobuz direct uses artist/page
  (same JSON shape as QobuzArtistDetail model) for top tracks + releases.
- B6 cause (TESTED from this phone): api.deezer.com returns data:[] in India (geo). gw-light guest also no tracks.
  PLAN: when Deezer ARL saved, use gw-light (deezer.pageSearch / deezer.pageArtist) with arl cookie.
- B5 cause: TIDAL search only via HiFi instances (fetchWithRetry "/search/?s="); TESTED: api.tidal.com/v1/search/tracks
  with x-tidal-token CzET4vdadNUFQ5JU works from India (cc US and IN). PLAN: direct TIDAL search first, instances fallback.
- B7 cause (TESTED via catalogue GET): app id 71210876 in SettingsViewModel.qobuzWebAppIds is INVALID (typo) ->
  its "Invalid or missing app_id" error masks the real error. 798273057 & 950096963 valid (need auth), 712109809 valid.
  PLAN: drop typo id, keep first meaningful error, stop early on bad-credentials. (Login POST probing was blocked by
  the auto-mode classifier: don't retry it.) play.qobuz.com/login reachable (SPA), WebView blank = unknown/geo.
DONE (code written, NOT compiled yet):
- B1: player/SafeMediaNotificationProvider.kt (wrap callback), player/RendererHold.kt (flags + request()),
  PlaybackService: setMediaNotificationProvider, renderer hold (holdForRenderer/releaseRendererHold,
  onUpdateNotification override, onStartCommand ACTION, onTaskRemoved keeps hold), MainActivity.onStart -> request.
  strings dlna_renderer_ready_title/_text added to strings.tsv + regenerated XML.
- B2: DlnaRenderer: queueManager+onTakeOver ctor, TRACK_TYPE, isRendererItem/isRendererTrack, synthetic Track
  into QueueManager on new item, DASH/HLS mime from DIDL, IDLE+error->STOPPED. PlaybackService guards in
  onPlayerError/playQueue/onTrackEnded/syncGaplessNext. DlnaServerManager passes queueManager + radio stop.
  PlaybackStateRepository skips persisting renderer tracks.
- B3/B4/B10 DONE (not compiled): SearchViewModel: Apple search -> null, APPLE chip removed. ArtistDetailViewModel:
  no APPLE source, no iTunes fallback; Apple/Deezer/Spotify ids that fail -> TIDAL name lookup. HiFiApiClient:
  getQobuzArtist rewritten (qobuzArtistPageToDetail shared mapper, direct artist/page + getReleasesList paging,
  artist/get extra=albums,tracks fallback; QOBUZ_ARTIST_TIMEOUT_MS=20s); iTunes fills removed from Deezer/TIDAL
  artist + Deezer/TIDAL album paths; Apple album/artist branches removed; getArtist cross-catalogue same-id
  fallback removed.
- [ ] B12 (owner): NO API server pre-selected automatically; old saved API servers from earlier mod builds must be
      cleared (one-time migration on first run of this build), and no default servers seeded. Keep logins.
- B3 extra DONE: StreamResolver Apple paths -> findLosslessFallbackStream (full length TIDAL/Qobuz), registry
  isAppleTrack no longer routes playback.
- B12 DONE (not compiled): PreferencesManager: default onrender server removed (empty list), one-time clear of saved servers + legacy keys via API_SERVERS_CLEARED_ONCE in seedBuiltInServer.
- B5 DONE (not compiled): HiFiApiClient.tidalDirectSearch (api.tidal.com/v1/search/{kind}, Bearer if signed in +
  x-tidal-token TIDAL_WEB_TOKEN) used first by searchTracks/Albums/Artists/Playlists, instances fallback.
- B6 DONE (not compiled): new data/api/DeezerGateway.kt (gw-light session via ARL: deezer.pageSearch,
  deezer.pageArtist + album.getDiscography, deezer.pageAlbum). HiFiApiClient: deezerGateway/deezerArl/registerGwTrack,
  used in searchDeezer, getDeezerArtist, getDeezerAlbum before public API. Deezer 30s preview playback removed
  (StreamResolver isPreviewUri -> not playable; getDeezerDownloadUrl no preview).
NEXT: B7 Qobuz app ids (SettingsViewModel.qobuzWebAppIds typo 71210876 -> remove; best error), B8 lyrics
  lyricFitWidth headroom, B9 spectro bitmap, then compile.
- B7 DONE (not compiled): SettingsViewModel qobuzWebAppIds = 798273057, 950096963 (typo id removed); login loop keeps
  account error over app_id error, skips to next id on app_id error, stops on 401.
- B8 DONE (not compiled): LyricsHero.lyricFitWidth divides by glyphHeadroom (GLYPH_HEADROOM_3D 1.08 / FLAT 1.05).
- B9 DONE (not compiled): TrueAudioBadges SpectroTab draws one bitmap (spectrogramBitmap + SPECTRO_LUT), axis Paint remembered.
NEXT: compile (low-RAM cmd), fix errors, run unit tests that touch changed code, assembleBenchmark, copy APK.
- compile #1 started (compileDebugKotlin, log in scratchpad compile1.log).
PENDING EDITS after compile #1: (a) PlaybackService.notePlay: return if DlnaRenderer.isRendererItem(mediaItem);
  (b) B11: hold foreground also while casting: PlaybackService inject UpnpCastManager, combine(dlnaRendererEnabled,
  selectedRenderer!=null) -> RendererHold.wanted/hold. TIDAL "Edit nahi aata" = device login token not saved
  (likely client secret for zU4XHVVkc2tDPo4t missing) -> NOT changed (can't verify; told owner).

## ROUND 3 request (2026-11-04 late): "USB bit-perfect ke liye aur powerful custom driver, UAPP/Neutron se advance"
Do AFTER round-2 APK ships. Known gaps of current libusb driver (from round 1): hardware Feature-Unit volume,
GET_CUR sample-rate readback, Android 14+ AudioMixerAttributes bit-perfect per track, DoP/native DSD,
gapless across sample-rate change (ring drains).
Owner answers (ROUND 3): DAC = "Audicular Spark Pro" (USB-C dongle; exact chip unknown, don't guess). No DSD files
 -> skip DSD. Priorities: (1) gapless across sample-rate change, (2) DAC rate-lock readback shown in UI,
 (3) bit-perfect hardware volume (Feature Unit). Start only after round-2 APK is built and copied.
SAFETY RULE from owner for hardware (DAC) volume — MUST hold:
 - DAC has NO volume buttons: phone volume keys + in-app volume bar must ALWAYS control the DAC level (bg too).
 - Never start or jump loud: on connect / switching to hardware volume, start at the last used level (or a safe
   low default), never above it; changes ramp; no sudden max.
 - If the DAC refuses Feature-Unit volume (or SET_CUR fails), keep the current software volume path unchanged.
RAM NOTE (owner asked to manage RAM): during compile #1 gradle JVM RSS ~3000MB, phone free ~550MB (7.4GB total).
 Next builds: add --max-workers=1 (no parallel task peaks); keep Xmx2600m; if OOM-killed, retry with Xmx2400m.
 Ask owner to close heavy apps (Chrome/YouTube/games) while gradle runs. Never run other heavy commands meanwhile.
- compile #1 (debug) PASSED 26m52s, no errors. Then applied: notePlay guard, cast hold (PlaybackService injects
  UpnpCastManager; combine(renderer pref, selectedRenderer)), getAlbum same-number Deezer fallback removed.
- NOW: assembleBenchmark (--max-workers=1, Xmx2600m), log scratchpad/bench1.log. After: copy APK as
  /sdcard/tryptifi hifi/Tryptify-mod-2026-11-04-r2.apk + source zip r2 (exclude build/, .gradle/, .kotlin/).
USB FINDING: cpp/usb/libusb_uac_driver.cpp applyHardwareVolume() (L1033) unmutes and sets DAC Feature-Unit
 volume to its MAX (<=0 dB) at every start(); volume is then software attenuation (BypassVolumeController).
 Round-3 plan: drive FU volume from the user's level (dB), software at unity when HW volume works; first
 apply = last user level (never louder), ramp; fallback = today's path. setSampleRate at L845 (check GET_CUR readback).

## 2026-11-04 ~22:50 restart (owner: "wapas normal termux pe chala gya tha")
WHAT HAPPENED: bench1 (assembleBenchmark, started 19:41) died ~20:08 inside compileBenchmarkKotlin. Kotlin frontend had
 finished (143 warnings, 0 errors, same last file as compile #1) -> killed in codegen = peak RAM (~3 GB JVM).
 Android killed the whole Termux/proot tree (Claude Code too: transcript stops 19:41, no build-exit notice) ->
 owner landed in plain Termux. Phone: Android 16, phantom-process monitor ON
 (persist.sys.fflag.override.settings_enable_monitor_phantom_procs=true), MemAvailable ~3.0 GB before build.
 Android's own kill log is not readable from Termux, so LMK vs phantom killer is not proven.
FIX: owner told: Developer options > "Disable child process restrictions" ON, Termux battery Unrestricted, wakelock,
 clear recent apps. Build now in 2 JVMs (compileBenchmarkKotlin, then assembleBenchmark), + CICompilerCount=2 and
 heap-shrink flags. Script + logs: /tmp/claude-0/-root/62c694d9-e1d2-4480-aaee-54be347ca2a6/scratchpad/
 (build_r2.sh, bench2.log, mem2.log). Benchmark type has no R8 (dexBuilder only); round 1 full build took 44 min.
 If killed again: just rerun build_r2.sh (finished tasks stay up-to-date / in build cache).
- 2026-11-04 23:44 ROUND 2 APK BUILT (build_r2.sh: step1 compileBenchmarkKotlin 22m57s, step2 assembleBenchmark 29m18s;
  min MemAvailable ~585 MB, java RSS peak ~3.3 GB in step 2; survived with child-process restriction disabled).
  Copied: /sdcard/tryptifi hifi/Tryptify-mod-2026-11-04-r2.apk (cmp OK; signer SHA-256 61d1c54b... = round 1 key, so
  it updates in place) + Tryptify-mod-source-2026-11-04-r2.zip (12578 files, Python zipfile, testzip OK; `zip` not
  installed in proot). Owner had trashed the round-1 files in that folder (left alone). Wakelock released.
  B1..B12 code is in this APK; NOT tested on device. Unit tests NOT run (heavy on phone; offered to owner).
  Not fixed: Qobuz official-login WebView blank, TIDAL "edit nahi aata" (device-login token save).
NEXT: owner's device test results for round 2 -> then ROUND 3 (USB DAC: gapless rate change, rate readback, HW volume).
- 2026-11-05 ~00:30 owner (testing r2): "abhi bhi API server apne aap aa jata hai", "old data mita do".
  FOUND: no hard-coded server (InstanceManager has none, BUILT_IN_ROOMS_SERVER empty). Settings sync (Supabase)
  pulled API_SERVERS + old per-catalog URLs from the cloud: SettingsSyncCodec.changedSince(base=null) = no local
  edits, so a device with no agreed base imports the cloud's old list. Also: Oct screenshots are labelled
  "Tryptify" (official app?) while the mod is "Tryptifi": the official app shows its built-in rooms server by design.
  FIX (B12b, r3): server keys taken out of SETTINGS_SYNC_KEYS (never pulled/pushed), syncSnapshotOf special case
  removed, one-time clear re-runs under new key api_servers_cleared_2026_11b (+ APPLE_WRAPPER_URL), AGENTS.md line.
  /sdcard Tryptify files (screenshots, crash logs, r2 APK/zip, Music/Tryptify) NOT deleted: they hold no server.
  Build r3: same build_r2.sh, log scratchpad/bench3.log.
- 2026-11-05 r3 build CANCELLED on owner's request ("saare bugs chek kar loon fir ek sath apk banana"): stopped in
  compileBenchmarkKotlin (~16 min in), wakelock released, nothing running. B12b code is in source, NOT compiled.
  NEXT: wait for owner's full r2 bug list (+ answer: server seen in "Tryptify" or "Tryptifi" app?), fix all, then
  ONE build (build_r2.sh). Kotlin compile may be full again (~53 min total) after the interrupted incremental run.

## ROUND 3 bugs (owner tested r2, 2026-11-05 ~07:15-07:40 IST). Evidence: /sdcard/tryptify eror/ (10 png + 2 mp4;
## frames extracted with ffmpeg (installed via apt) into scratchpad/frames). Owner: build ONE apk after all fixes.
Owner says: Qobuz works perfectly; TIDAL + Deezer login OK; TIDAL + Deezer search show full catalogue.
- [x] R1 TIDAL songs never play: "TIDAL couldn't play this ... Play from Qobuz?" every track, queue skips meanwhile.
      Cause unknown (direct playbackinfopostpaywall failure was swallowed). DONE (not compiled): HiFiApiClient direct
      path tries requested tier -> LOSSLESS -> HIGH, rejects encrypted manifests (tidalManifestProblem: MPD
      ContentProtection / BTS encryptionType != NONE; no decryption), keeps TIDAL's reason per track
      (tidalDirectProblems, tidalErrorText "HTTP x · subStatus · userMessage"), Log.w; repo.tidalStreamProblem;
      QobuzOffer.reason shown in dialog (string tidal_offer_reason); SourceConsent.allowAll + dialog button
      "Always use Qobuz" (string play_from_qobuz_always) -> PlayerViewModel.acceptQobuzOfferAlways. 8 locales done.
      Possible real cause: TIDAL geo-block in India (can't fix in code) -> next test shows TIDAL's reason.
- [x] R2 Deezer songs skip in ~1 s (only local copies play). Cause: no API server -> no Deezer stream; fallback
      findLosslessFallbackStream got no ISRC (resolveDeezerLosslessFallback needs api.deezer.com, empty in India).
      DONE (not compiled): HiFiApiClient.deezerIsrc (gateway map, else api.deezer.com/track), repo.deezerIsrc,
      StreamResolver.deezerViaQobuz (ISRC -> findQobuzByIsrc, else metadataMatchQobuz -> qobuzCache.openPartial ->
      QobuzStreamUri, PlayedFrom.QOBUZ) in both Deezer paths (resolveMediaItem + resolveDeezer).
- [ ] R3 API server appears automatically -> B12b fix already in source (servers not synced + re-clear). Owner never
      answered "Tryptify (y) or Tryptifi (i) app?"; ask for a screenshot of where it appears.
- [ ] R4 TIDAL "logs out by itself after some time". No code clears TIDAL tokens except Clear / Save with empty
      fields (DirectAccountsSection TidalDialog). Ask owner how they see it (card says "Not signed in"?).
- [ ] R5 TIDAL manual option question: token only = works until it expires (no refresh); user id alone = nothing;
      client id needed for refresh (default zU4XHVVkc2tDPo4t prefilled). Answer owner.
- [ ] R6 mConnect (same phone) doesn't list "Tryptify Renderer" (only Google Cast). Screens show phone on 4G +
      hotspot, no Wi-Fi -> ask owner to retest on Wi-Fi; check DLNA interface choice if still failing.
- [ ] R7 Apple Music card still in Settings > Connections > Your streaming accounts (DirectAccountsSection.kt card #4
      + dialog "apple" + intro text). Remove (B3 leftover).
NEXT after R7: compile (build_r2.sh does benchmark; maybe compileDebugKotlin first), copy r3 APK + source zip.
- R7 DONE (not compiled): DirectAccountsSection: Apple card + AppleDialog + apple vals removed, intro text says Spotify
  only; ServiceQuality.services without APPLE (no Apple quality rows; ladder kept for old Apple songs).
- R6: no code change. DlnaServerManager already prefers hotspot/Wi-Fi IPs + same-phone remote IP. Owner's screens show
  no Wi-Fi (4G + hotspot) -> ask owner to retest mConnect/BubbleUPnP with Wi-Fi connected.
- R4: no automatic TIDAL token removal exists (only Clear / Save-with-empty / Settings "clear all data"). New dialog
  reason will show "HTTP 401 ..." if the session dies. Ask owner what "logout" looks like.
- R5 answer: client id is prefilled (default zU4XHVVkc2tDPo4t), user id is never needed; access token alone works
  only until it expires (no refresh) -> use option 1 "Log In via Official TIDAL".
- Unit tests NOT run this round (heavy). Build r3 started: build_r2.sh -> scratchpad/bench4.log; then copy
  /sdcard/tryptifi hifi/Tryptify-mod-2026-11-05-r3.apk + source zip r3 (Python zipfile, exclude app/build app/.cxx
  .gradle .kotlin).
- R6 UPDATE: owner says mConnect on the SAME phone with HOTSPOT ON used to list "Tryptifi Renderer" -> regression.
  Likely cause: SsdpServer joined the SSDP group only on hostIp's interface (hostIp can be 127.0.0.1 / old Wi-Fi IP
  after round-1 network-watcher restarts; tethering changes fire no NetworkCallback), NOTIFY went out the default
  (mobile) route. DONE (not compiled): SsdpServer joins every LAN interface (up, multicast, IPv4, not rmnet/tun/...),
  re-checks every 10 s and announces at once on a new one (hotspot later), NOTIFY/byebye sent per interface with that
  interface's IP in LOCATION (multicastOnEachLan). HTTP server already binds 0.0.0.0; M-SEARCH replies use
  findBestLocalIpForRemote.
- 2026-11-05 ~08:10 IST: r3 build restarted with all fixes (bench5.log). Earlier r3 attempt (bench4) stopped by me
  after 2 min to add this DLNA fix.
- Owner (08:15 IST): "TIDAL + Deezer connect but songs don't play; Deezer ARL doesn't work either."
  Told owner: Deezer full tracks are encrypted (Blowfish); ARL = login/catalogue only; decrypting = breaking Deezer's
  protection -> DECLINED (same rule as Apple). Old builds played Deezer via API servers (server-side). r3: Deezer
  songs play the same recording from Qobuz (deezerViaQobuz). TIDAL: likely region refusal for India (login/search
  work, stream link refused); r3 dialog shows TIDAL's reason. Asked owner to test: official TIDAL app without VPN,
  and Tryptifi with a VPN in the account's country.
- Owner: UAPP (official TIDAL partner) streams TIDAL in India WITHOUT VPN -> region theory wrong. Suspect TIDAL refuses
  streams for the borrowed client id zU4XHVVkc2tDPo4t (login/search still work). Owner decision: FIRST finish r3
  (copy APK + zip), THEN do the TIDAL test from Termux: device login (owner confirms code at link.tidal.com), call
  playbackinfopostpaywall for one track, read TIDAL's answer, delete the token afterwards. Owner agreed to both.
- Owner asked to copy "Resonda" player's (GitLab) Deezer ARL playback method into Tryptifi. DECLINED: that method
  decrypts Deezer's Blowfish-encrypted full tracks = circumventing Deezer's protection (same rule as Apple Music).
  Did not fetch or read that code. Deezer stays: catalogue/login via ARL, audio = same recording from Qobuz (r3), and
  from TIDAL too once TIDAL streaming works (findLosslessFallbackStream).
- Owner asked to read Resonada's (gitlab.com/resonada/resonada) TIDAL login/playback code. Checked via GitLab API:
  repo has only README, screenshots, CI, autoeq_index.tsv (52 files), README says "proprietary freeware" -> NO
  source code exists to read; README doesn't even mention TIDAL/Deezer. Plan unchanged: r3, then TIDAL Termux test.
- Owner (08:50 IST): "TIDAL zaroor solve karo, tabhi APK banana". r3 build stopped in STEP2 (STEP1 compileBenchmarkKotlin
  PASSED 25m27s, 0 errors -> all r3 code compiles). No APK until TIDAL works. Now: TIDAL device-login test from Termux
  (script scratchpad/tidal_test.py; tokens only in scratchpad/tidal_state.json 0600, never printed; delete after).
- TIDAL TEST RESULT (owner confirmed a device-login code; app's own client id; token deleted after the test):
  * login OK, access token lasts 4 h (expires_in 14400), account country NL, TIDAL sees the phone's IP as IN.
  * every stream request refused: 401 subStatus 4005 "Asset is not ready for playback" (all tracks, all tiers,
    any countryCode); track metadata itself says streamReady/allowStreaming true.
  * renewing the login (refresh_token) refused: 401 invalid_client.
  => R1 + R4 explained: TIDAL refuses streams and renewals for the client id the app logs in with, so songs never
     play and the login dies after ~4 h. UAPP works because it is an official TIDAL partner.
  DECISION: not working around TIDAL's refusal with other apps' credentials. In-app answer: TIDAL catalogue/search
  stay; songs play the same recording from Qobuz ("Always use Qobuz" in r3). Ask owner to OK building r3 this way.
- Owner (09:05 IST): "mujhe TIDAL chahiye kisi bhi tarah". Answered: no use of other apps' / partners' keys (UAPP etc.).
  Researched the SANCTIONED route: TIDAL developer platform (developer.tidal.com) gives anyone a client id, but per a
  developer's report in github.com/orgs/tidal-music/discussions/347 (0 replies from TIDAL) the THIRD_PARTY tier returns
  previews only (FULL_REQUIRES_HIGHER_ACCESS_TIER); full tracks need a higher tier TIDAL grants to partners. The Android
  SDK test app also plays 30 s previews unless the client is authorised for full playback.
  OPTIONS OFFERED: (1) build r3 as is: TIDAL catalogue/search + songs from Qobuz ("Always use Qobuz"), (2) owner may
  apply to TIDAL for a higher tier; if approved add "own client id" login (PKCE) + /v2 trackManifests, (3) keep UAPP for
  TIDAL. WAITING for owner's choice. r3 code compiled (STEP1 ok); STEP2 (dex/package) not finished; no APK yet.
- Owner (09:15 IST): "agar direct TIDAL solve nahi ho raha to rahne do; agar TIDAL token se kaam chal jaye to theek, warna
  sab kuch yahi rok do". Token answer: a token minted by this app's own login is refused (tested); a token from some other
  client (UAPP / official apps / web player) might stream but means using that client's identity (not doing it) and it
  expires in hours with no renewal -> NOT a solution. => OWNER'S DECISION = STOP. r3 NOT built, nothing running, no tokens
  stored. Latest APK in /sdcard/tryptifi hifi/ is still r2 (known r2 bugs remain: API server via sync, hotspot DLNA
  regression, Deezer skips, Apple card, TIDAL logout/stream).
  TO RESUME if owner says "r3 bana do": code is already compiled (STEP1 ok); run scratchpad/build_r2.sh (set Developer
  options child-process restriction OFF, wakelock, Chrome closed), then copy APK + source zip (Python zipfile).

- Owner (~09:30 IST): "bas mConnect (same phone, hotspot) solve karo + final APK bana do; API server jo apne aap aata hai
  wo bhi solve karo; APK + zip /sdcard/tryptifi hifi/ mein save karna". TIDAL dropped (owner decision), r3 = everything else.
  DONE (not compiled yet): SsdpServer reworked: joins SSDP group on every LAN interface (no multicast-flag filter), tracks
  memberships by name#index@ip so hotspot off/on rejoins, sendLock instead of synchronized(socket) (receive() may hold the
  socket monitor => deadlock risk found in my own earlier edit), announces 3x on a newly joined network, NOTIFY per interface
  with its own IP. DlnaServer.hostIp now updatable (updateHostIp); DlnaServerManager.watchAddress polls every 10 s (hotspot
  raises no Wi-Fi callback) and updates state without restart. r1->r2 diff showed SSDP join logic unchanged: it only joined
  the interface of the start-time IP, so hotspot turned on later was never joined (root cause of owner's report).
  API server (B12b+): savedApiServers no longer rebuilds from legacy per-catalog keys; one-time clear (key ..._11b) also
  removes saved server passwords; server keys are not in settings sync. TIDAL card text: no streaming promise.
  NEXT: compile (build_r2.sh), copy to /sdcard/tryptifi hifi/Tryptify-mod-2026-11-05-r3.apk + Tryptify-mod-source-2026-11-05-r3.zip.
- Owner (~09:40 IST): "TIDAL aur Deezer ko rahne dena, r3 ke saath combine mat karna, usi haal pe chhod do".
  => My TIDAL/Deezer edits REVERTED to the r2 files (HiFiApiClient, MusicRepository, SourceConsent, StreamResolver,
  MonochromeNavHost, PlayerViewModel, strings.tsv + 8 strings.xml, TIDAL card text). Backup of those edits + patch:
  /root/backups/tidal-deezer-r3-attempt/ (README inside). Checked with a CRC compare against the r2 source zip.
  r3 = ONLY: SsdpServer/DlnaServer/DlnaServerManager (mConnect same phone + hotspot), PreferencesManager (API server
  fix B12b+), DirectAccountsSection + ServiceQuality (Apple card/quality row gone, from owner's round-2 "remove Apple
  Music"), AGENTS.md doc line. Build: build_r2.sh -> scratchpad/bench7.log. Then copy APK + source zip r3 to
  /sdcard/tryptifi hifi/ (owner asked for both there).
- Owner (~09:50 IST): r3 files must be saved in /sdcard/tryptifi hifi/ under a COMPLETELY DIFFERENT name from the earlier
  ones; leave the old files (r1 in trash, r2 apk + zip) untouched, never overwrite/rename them.
  NAMES TO USE: /sdcard/tryptifi hifi/Tryptifi-r3-mconnect-fix.apk  and
  /sdcard/tryptifi hifi/Tryptifi-r3-mconnect-fix-source-code.zip  (NOT the earlier "Tryptify-mod-2026-11-05-r3" plan).


## ROUND 4 (2026-11-05, claude.ai): TIDAL + Deezer login/playback. Source: Tryptifi-r3-mconnect-fix-source-code.zip
Owner asked: TIDAL direct login + token; Deezer direct login + ARL; any country's subscription; no playback errors.
NOT COMPILED (no Kotlin/Gradle in that sandbox; only a bracket/string balance check). Run compileDebugKotlin first.
DECLINED again (same rule as rounds 2-3): Deezer ARL -> encrypted CDN file -> decrypt. Other apps' TIDAL client keys.
Changes:
- PreferencesManager: tidal_auto_qobuz_fallback (default true, device-local), clearTidalDirect(),
  setTidalDirectSession(replaceRefresh) so a pasted access token drops an old refresh token.
- TidalAuth: signInWithToken (cleans pasted token, reads JWT cid/exp, refresh with the token's own client id
  unless one is typed, /sessions check), describeSession, testStreaming (search 1 song + playbackinfo, reports),
  manifestProblem (BTS encryptionType != NONE or MPD ContentProtection = not playable, never decrypted),
  errorText ("HTTP 401 · 4005 · msg"), lastStreamProblem flow, pause direct streaming 10 min after 3 refusals.
- HiFiApiClient: direct TIDAL tier ladder (HI_RES->LOSSLESS->HIGH), records refusals; getTidalIsrc falls back to
  api.tidal.com; findQobuzTrackByIsrc falls back to Qobuz catalog/search (no server needed);
  findLosslessFallbackStream order Qobuz ISRC -> Qobuz strict metadata -> TIDAL (only if usable), strict
  bestMetadataMatch; deezerSongMeta/deezerIsrc via gw-light song.getData (ARL) then api.deezer.com.
  BUG FIX: getDeezerDownloadUrl no longer returns cross-catalog URLs (they went through DeezerStripeDecryptor in
  DeezerStreamCacheManager/TrackDownloader and came out scrambled -> player error).
- DeezerGateway.song() (metadata only). MusicRepository.deezerIsrc/deezerSongMeta.
- StreamResolver: deezerPlayable = Deezer server cache -> Qobuz copy (ISRC/strict match, Qobuz cache,
  PlayedFrom.QOBUZ) -> other lossless match; TIDAL refusals play the Qobuz copy without the consent popup when the
  pref is on (popup still used when off). Injects PreferencesManager.
- Settings UI (DirectAccountsSection, English like the rest of that file): TIDAL dialog = status card (country,
  expiry, last refusal, Test playback) + "Play from Qobuz when TIDAL refuses" switch + Option 1 Direct login +
  Option 2 Token login. Deezer dialog = honest text + Option 1 Direct login (web page) + Option 2 ARL (accepts
  arl=..., cookie line, quotes). OfficialLoginWebDialog also expires parent-domain cookies (stale .deezer.com arl).
To test on device: TIDAL Test playback result; Deezer song plays with "from Qobuz" marking; TIDAL song with refusal
plays from Qobuz without popup.
