# Interview walkthrough

This is what I show when I screen-share Connect, and the questions I expect it to raise. Everything runs in demo mode on a phone or an Android emulator, with no hardware attached.

## The 10-minute demo

| # | Time | What I open | What I show | Question it answers |
|---|---|---|---|---|
| 1 | 0:00 | The app, cold start | Launch screen, then My Boat with every status labelled "Simulated" | Can I tell at a glance what's real and what's simulated? |
| 2 | 0:45 | My Boat → Pair a device | QR scanner, the manual-code fallback, and the note that pairing needs a window opened on the unit | What stops a stranger from pairing with your boat? |
| 3 | 1:45 | Simulate pairing to an unclaimed unit | The "this unit had no previous owner" warning, which has to be ticked before continuing | How does the app handle a unit that might be stolen or already owned? |
| 4 | 2:45 | Live tab | Link status, sequence number, speed telemetry sparkline, the demo-feed label | How does the app show link health honestly? |
| 5 | 3:45 | Live tab controls | Digital zoom, Snapshot, Save Highlight; the locked Safety card | Which actions are local, which would be device commands, and why can't safety be switched off? |
| 6 | 4:45 | Library | Filters, favourites, playback of the highlight I just saved, share and download | How does captured media flow back to the user? |
| 7 | 5:45 | Session and Community | The day's timeline and the leaderboard hub | What does the product look like beyond the core loop? |
| 8 | 6:30 | The code editor: the test folder | Pairing transport, secure key and credential-store tests; the control-channel tests for ack, reject, timeout and reconnect | How do you test network code without hardware? |
| 9 | 8:00 | A terminal | `flutter test` running green, then the database test suite | Is the test suite real, and how long does it take? |
| 10 | 9:00 | GitHub Actions | The CI workflows on a recent change | What stops a bad change, from a human or an AI agent, from merging? |

## The 8 hardest questions I expect, with honest answers

**1. Is the pairing key really hardware-protected?**
Not yet completely. The pairing key is a real P-256 key generated in the app and encrypted at rest by the platform keystore, but it isn't a non-exportable key living inside the secure element. The account-level device key already uses the Android Keystore's non-exportable path, and the signing interface never exposes the private key, so moving the pairing key the same way doesn't touch any caller.

**2. Has this ever talked to a real Core device?**
No. The phone side is implemented and tested against fakes, loopback sockets and an emulator, but the wire contract hasn't been verified against production hardware yet. I wrote down the physical acceptance sequence in advance (pair, authenticate, live telemetry and video, capture, playback, plus link loss and expired credentials), and no simulated result counts toward it.

**3. What happens when the boat's Wi-Fi drops mid-command?**
Outstanding commands are failed and reported as "disconnected", never silently retried, because replaying something like "save highlight" after a reconnect could do the wrong thing twice. The client reconnects with backoff from 1 s up to 32 s, and a generation counter makes sure callbacks from the old socket are ignored.

**4. How do you know the data on screen is current?**
Each state message has a sequence number, and anything that doesn't advance it is dropped. Staleness is measured on a monotonic stopwatch that restarts each time a valid state message arrives, not from the device's own clock or the phone's wall clock: 15 seconds of silence shows "not responding", and 45 seconds drops the link. I originally used wall-clock time and switched to a monotonic stopwatch, so that a phone clock change can't make a dead link look live.

**5. Why send the bearer token in the first WebSocket frame instead of a header or the URL?**
A token in the URL ends up in proxy logs, server access logs and browser history. The browser WebSocket API can't set custom headers, and the web build matters to me, so the token goes in the first frame after the TLS handshake, before any command is accepted.

**6. Where is authorization actually enforced?**
On the device and in the database, not in the app. The app's role checks only grey out buttons a spectator can't use; the Core re-checks every command, and the cloud uses row-level security, tested in CI from the point of view of different users, so a user can only ever touch their own rows.

**7. Why does demo mode exist, and couldn't it leak into production?**
It lets me demo, test the UI and onboard people without hardware, which matters while the hardware is still in build. It is fixed at compile time: a demo build can't open a connection to a device, and a device build refuses to start without a configured address instead of falling back to fake data.

**8. How do you use AI coding agents without losing control of the codebase?**
I use them for implementation, tests and review passes, and I review every change myself. CI gates every merge with static analysis, the test suites and a static policy audit, and I keep the architecture decisions and the hard boundaries (keys, auth, safety commands) for myself.
