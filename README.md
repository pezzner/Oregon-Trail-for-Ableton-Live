An Ableton Live extension that adds a right-click "Play Oregon Trail..." action to any track header. It opens a self-contained, old-school-styled retelling of the classic trail game — hand-drawn pixel-art scenes plus text, not a wall of prose — entirely inside the extension's own dialog webview. No Live Set data is read or written.

**The twist: you always get dysentery.** It's not a one-time cameo — dysentery keeps coming back throughout the trip (party members, more than once each), the oxen can catch it too, everyone you meet on the trail already has it, and anyone currently afflicted forces regular bathroom stops that cost the party a day. And however anyone in the party actually dies — starvation, a river, a stray illness, anything — the record only ever says one thing: dysentery. If the leader dies, the game always ends with **"You have died of dysentery."**

## Using it

Right-click any track's **header** (the name/title strip at the top of the track — not a clip, and not the empty track body) in Session or Arrangement view and choose:

**"Pezzner - Oregon Trail: Play Oregon Trail..."**

That opens the game in a fixed-size dialog (980×720). Play through name/profession/departure-month setup, outfit at the store, then travel — pace and rations, hunting, river crossings, random events — until the party either reaches the Willamette Valley or the leader dies. Closing the dialog early (Escape or the title bar) just cancels; nothing is written anywhere.

Progress and the final outcome are logged to the extension's console output (Live's Log.txt / the terminal running `npm start`), not surfaced anywhere in the Live Set itself.

### Note on `manifest.json`'s `"name"`

For track-header (`MidiTrack`/`AudioTrack`) context-menu scopes, Live prefixes whatever label an extension registers with that extension's `manifest.json` `"name"` field, joined as `"<name>: <label>"`. This project's convention (established with MIDI Exploder, the first extension to use a track-object scope) is to keep `manifest.json`'s `"name"` as a unique `"Pezzner - <Extension Name>"` — so it's identifiable on its own in Live's Settings → Extensions page — and keep the registered label action-only, never repeating the extension name. Here that's `"Play Oregon Trail..."`, producing the final menu text `"Pezzner - Oregon Trail: Play Oregon Trail..."`.

## Development

```
npm install
npm start
```

`npm start` builds the extension and loads it into Live's Extension Host (Developer Mode must be enabled in Live's Settings → Extensions).

Requires a `.env` file (not included — the tooling that scaffolded this project can't write one) with:

```
EXTENSION_HOST_PATH=C:\ProgramData\Ableton\Live 12 Beta\Program\ExtensionHost\ExtensionHostNodeModule.node
```

adjusted to wherever your Live install's Extension Host module lives.

`npm install` may warn about `esbuild`'s postinstall script being skipped — this project's `package.json` pre-declares `allowScripts` for the expected resolved `esbuild` versions (`0.25.12`, `0.28.2`, matching every other extension in this project). If `npm install` reports a different resolved version in an `npm warn install-scripts` line, add it to `package.json`'s `allowScripts` and reinstall.

## Structure

- `src/extension.ts` — activation: registers the `oregon-trail.play` command and the two track-header context-menu actions (`MidiTrack`, `AudioTrack`), opens the dialog, and logs the outcome.
- `src/dialog.html` — the entire game. Self-contained HTML/CSS/JS styled to match this project's other dark-themed dialogs (`--ableton-panel` background, monospace type), with Oregon-Trail-flavored accent colors, plus a set of small canvas-drawn pixel-art scenes (wagon and oxen team, hills and sun, a river crossing, a tombstone, a hunting quarry) scattered across the title/travel/hunt/river/end screens. No external assets, fonts, or images anywhere — every scene is drawn with plain canvas rects/arcs at a tiny internal resolution and stretched up with `image-rendering: pixelated`, not a port of any specific existing version of the game.
- `src/html.d.ts` — lets `extension.ts` `import` the dialog HTML as a bundled string via esbuild's `.html` text loader.

## Design notes

- **Guaranteed dysentery, made airtight**: the first trigger is based on travel-leg count (randomized to leg 1–3), not miles, and is checked *before* any other per-leg processing (health drift, illness rolls, random events, landmark/river stops) rather than after. When it fires, that leg's processing stops there — the dysentery prompt is the only thing that happens that turn, and nothing else can race it (kill the leader out from under the prompt, or paper over it with a river-crossing overlay) before the player responds. Deferred events simply resume on the next "Travel" click. This was tuned via fuzz-testing across randomized pace/ration combinations after an earlier version could occasionally have the leader die of ordinary causes before the dysentery trigger was ever reached.
- **Every death is officially dysentery**: `killMember(member, cause)` always records and displays "died of dysentery," regardless of what `cause` actually says (starvation, an ailment, drowning at a river crossing — a wry second log line still names the real cause, "involved, but everyone knows what it really was"). This is what makes "the game always ends with someone dying of dysentery" literally, unconditionally true rather than usually true — there's no race to win against other death causes, because there's nothing left to race.
- **Dysentery keeps coming back, not just once**: the leader has their own guaranteed backstop (a second early trigger, `leaderDysenteryDeadlineLeg`, in case the first guaranteed case landed on a companion instead) so the leader personally catches it too, not just whoever the dice picked first. After the game's one dramatic onset (a modal with a real choice — rest and use medicine, or push on regardless), every further case — companions catching it for the first time, anyone relapsing after recovering — is a quiet log-only event instead of another popup, so the disease stays constantly present without needing constant dismissing. While anyone's currently afflicted, the party automatically loses a day each leg to "needing to stop the wagon," logged by name.
- **Oxen and strangers aren't exempt either**: oxen have their own small per-leg chance of catching it, and once sick can die of it (reducing the team, which naturally feeds into the existing "fewer than 4 oxen slows you down" travel penalty) or recover. Every leg also has a chance of a flavor-only encounter with another traveler on the trail — they always have dysentery too.
- **Still actually winnable**: increasing dysentery's frequency and lethality made the game meaningfully harder, but a careful, well-stocked, steady-paced run can still reach the Willamette Valley — verified with a generously-outfitted playthrough (8 oxen, deep food and medicine reserves, always treating outbreaks rather than pushing through them) reaching "You Made It!" Winning doesn't require anyone to have died of dysentery along the way; only the *losing* ending is unconditionally tied to it.
- **No Live Set interaction by design**: this is a distraction, not a tool — it doesn't read the song, tracks, or clips, and registering on both track scopes just means it doesn't matter which kind of track you right-click.
- **Dialog size is fixed**: the SDK's `showModalDialog` can't be resized once open (confirmed elsewhere in this project), so 980×720 was picked to fit the travel screen's HUD, party list, and event log without internal scrolling at typical display sizes.
- **Pixel art is drawn, not shipped as image files**: a shared handful of canvas primitives (`drawWagon`, `drawOxenTeam`, `drawSkyline`, `drawRiver`, `drawTombstone`, `drawAnimal`, `drawSun`) get reused across the title, travel, hunting, river-crossing, and end scenes, so one wagon/hill/skyline "sprite" appears consistently everywhere rather than five one-off drawings. This keeps the file single and self-contained (this project's dialogs don't bundle external assets) and sidesteps any question of reproducing another game's actual art, since nothing is copied from a reference image — everything is hand-specified shapes. The travel banner also reacts a little to real game state (rain/snow rendered for stormy/cold weather, a hill-skyline seed tied to progress along the trail) rather than being a static decoration.

## Confirmed working in Live

Dave has run this in Live 12 Beta and confirmed the menu, dialog, and full game flow work end to end.

## Not yet done

The pixel-art scenes (still unconfirmed for crisp, non-blurry rendering under Live's actual WebView2 host — depends on it respecting `image-rendering: pixelated`) and this round's much heavier dysentery mechanic haven't been seen by Dave in Live yet. Verified so far only in the build sandbox: headless-Chromium screenshots of all the scene types, a 15-run fuzz test across randomized pace/ration combinations confirming every single run now ends on the "You Have Died of Dysentery" screen, and a dedicated generously-outfitted run confirming a win is still reachable. Worth a real playthrough to confirm the new frequency feels right rather than overwhelming — the log now fills up fast with dysentery-related lines (onsets, recoveries, bathroom stops, traveler encounters), which was the explicit ask ("More dysentery please") but is worth a gut check against the actual pacing of a real game.
