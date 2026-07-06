---
title: "TryHackMe: Fools Mate"
author: '0x6D4E6D'
categories: [TryHackMe]
tags: [web, api, client-side-validation, logic-flaw, chess]
render_with_liquid: false
media_subpath: /assets/images/tryhackme_foolsmate
image:
  path: room-img.png
---
## Overview

It's mate in one. You know it, the engine knows it, my grandma knows it. The board says checkmate is one click away. The engine says no. Settle the argument.
Room Link: [Fools Mate](https://tryhackme.com/room/foolsmate)

## Recon

The target web application was available at:

```text
http://10.114.157.19/
```

I started with basic enumeration. A quick Nmap scan did not reveal anything useful beyond the web service, so I moved on to manual web inspection.

When opening the page in the browser, the application displayed a chess board game.

![Landing page showing the Endgame Trainer chess board](01-landing-page.png)

The position was a simple rook endgame puzzle. White had a rook on `a1`, and the black king was trapped on `g8` by its own pawns.

The intended winning move was:

```text
from: a1
to: a8
```

---

## Testing the UI

I tried to play the checkmate move through the web interface.

Instead of allowing the move, the application displayed a fake system-style error message:

```text
I'll shut down your PC if you play that.
```

![Client-side warning shown after attempting checkmate](02-client-side-warning.png)

This showed that the application was intentionally blocking the checkmate move from the UI.

At this point, the important question was whether the check was enforced by the server or only by JavaScript in the browser.

---

## Inspecting the Frontend Code

I inspected the page source and JavaScript files. The main frontend logic was loaded from:

```text
/js/app.js
```

Inside `app.js`, I found the following function:

```js
function preMoveCheck(from, to, promotion) {
  const probe = new Chess(game.fen());
  let result;
  try {
    result = probe.move({ from, to, promotion: promotion || undefined });
  } catch (e) {
    result = null;
  }
  if (result && probe.isCheckmate()) {
    showSystemNotice("I'll shut down your PC if you play that.");
    return false;
  }
  return true;
}
```

This function creates a local copy of the chess position in the browser and tests the move before it is sent to the server.

If the tested move results in checkmate, the function displays the warning message and returns `false`.

That means the restriction is happening on the client side. The browser blocks the UI action before the request is sent, but this does not necessarily mean the backend rejects the same move.

Client-side code can be inspected, modified, or bypassed. If the backend endpoint accepts direct requests, the frontend protection does not provide real security.

---

## Finding the API Endpoint

Continuing through the JavaScript file, I found that moves were submitted to the backend using a `fetch()` request to:

```text
POST /api/move
```

The request body used JSON with `from` and `to` fields:

```json
{
  "from": "a1",
  "to": "a8"
}
```

So the plan was simple:

1. Get the current session cookie from the browser.
2. Send the checkmate move directly to `/api/move` with `curl`.
3. Bypass the JavaScript `preMoveCheck()` function entirely.

---

## Getting the Session Cookie

The application used a session cookie named `sid`.

I opened the browser developer tools and checked the stored cookies for the target host.

![Session cookie in browser storage](03-session-cookie.png)


## Exploitation

Instead of using the board UI, I sent the move directly to the backend with `curl`.

The important part is the JSON body:

```json
{
  "from": "a1",
  "to": "a8"
}
```

Example command:

```bash
curl -X POST http://10.114.157.19/api/move \
  -H "Content-Type: application/json" \
  -H "Cookie: sid=<your_sid_cookie>" \
  -d '{"from":"a1","to":"a8"}'
```

The server accepted the checkmate move and returned the flag:

```json
{
  "ok": true,
  "move": "a6a8",
  "fen": "R5k1/5ppp/8/6K1/8/5P2/6PP/8 b - - 2 9",
  "status": "checkmate",
  "turn": "b",
  "winner": "white",
  "flag": "THM{REDACTED}"
}
```
