---
title: "TryHackMe: Cache Me Outside"
author: '0x6D4E6D'
categories: [TryHackMe]
tags: [osint, github, komoot]
render_with_liquid: false
media_subpath: /assets/images/tryhackme_cache_me_outside
image:
  path: room-img.png
---
## Overview

In this room, the goal was to investigate a retired hacker by following small pieces of public information left across the internet.
The investigation started from a leaked conversation screenshot and continued through public profiles, GitHub metadata, and social media clues.

Room Link: [Cache Me Outside](https://tryhackme.com/room/cachemeoutside)
---

## 1. Starting Point: Conversation Screenshot

The first clue was found in the provided conversation screenshot.

The conversation mentioned that the person had moved away from hacking and started spending more time outdoors. Inside the screenshot, there was a Komoot profile link.

```text
https://www.komoot.com/user/5667624959835
```

![Conversation screenshot](conversation.png)

This gave the first pivot point for the investigation.

---

## 2. Komoot Profile

The Komoot profile belonged to **Jim Lee**.

The profile description mentioned that he was an ex-hacker trying to turn his life around. The profile also exposed another useful clue: a GitHub profile.

```text
github.com/jiml33t
```

![Komoot profile](01-komoot.png)

At this point, the full name was confirmed:

```text
Jim Lee
```

---

## 3. Pivoting to GitHub and Email Address

The GitHub profile was located at:

```text
https://github.com/jiml33t
```

The next step was to inspect the profile repository and its commit history.

![GitHub profile](02-github.png)

The initial commit was especially interesting because Git commits can expose metadata such as usernames and email addresses.

The commit hash was:

```text
7b2c8e0a540c36f2e09da5945066020621d6a059
```

The patch file could be opened by adding `.patch` to the commit URL:

```text
https://github.com/jiml33t/jiml33t/commit/7b2c8e0a540c36f2e09da5945066020621d6a059.patch
```
Inside the patch file, the email address was exposed in the commit metadata.

```text
From: jimleepro1-cell <jimleepro1@gmail.com>
```

![GitHub expose](03-github.png)

This confirmed the leaked email address:

```text
jimleepro1@gmail.com
```

It is important to note that the email was not inside the README content itself. It appeared in the Git commit metadata.

---

## 5. Phone number

After finding the exposed email address in the GitHub commit metadata, I wanted to verify that this email was actually part of the challenge trail.

The leaked address was found in the `.patch` file of the initial GitHub commit:

```text
From: jimleepro1-cell <jimleepro1@gmail.com>
```

At this point, I sent a direct email to that address with a simple random message.

![Email sent to Jim Lee](mailprompt.png)

Shortly after, I received an automatic reply.

The auto-reply confirmed that the email address was active in the challenge context and also contained the phone number in the email signature.

![Auto-reply from Jim Lee](answer.png)

The signature contained:

```text
Jim Lee
Cybersecurity Consultant
jimleepro1@gmail.com
+40 743 321 239
L33T Security
Pentesting · Red Team · Consulting
```

This gave the phone number:

```text
+40 743 321 239
```


---

## 6. Pivoting to Threads

After I searched for related accounts and references.
This led to a Threads account:

```text
@jiml33t
```

![Threads profile](threads.png)

The Threads account became the next source for location and activity clues.

---

## 7. Finding the City

The next location clue came from the Threads post.

Jim posted that he had just finished his last run before the big day and was getting on the tram for a coffee at his favourite French supermarket.

![Threads post](threads.png)

The attached image was the important part. In the photo, I noticed a visible sign on the left side of the road:

```text
IRIGATII.RO
```

![Threads photo](threads2.png)

This looked like a business or shop name, so I used it as a geolocation clue. I searched for:

```text
irigatii.ro Romania
```

After that, I opened the result in Google Maps / Street View and compared the location with the photo from Threads.

![Google Maps match](shop.png)

The Street View location matched the Threads image:

- the `IRIGATII.RO` sign
- the yellow/orange building
- the road layout
- the tram wires
- the tram infrastructure nearby
- the same general camera angle and surroundings

Google Maps showed this location on **DJ592** in **Timișoara, Timiș County, Romania**.

This confirmed the city:

```text
Timișoara
```


## 8. Finding the Tram Station

The next question asked for the tram station where he got off on **7 May 2026**.

Since the city was already known to be **Timișoara**, I used the activity/photo context and searched for nearby tram stations.

After comparing the location with nearby tram stations, the matching station was:

```text
Piața Gheorghe Domășneanu
```

![Tram station map](nearest-tram.png)