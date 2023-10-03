---
layout: post
title: "OAuth clickjacking attack (with the example)"
categories: misc
tags:
- OAuth2.0
- vulnerability
- hacking
- OAuth
cover-img: /assets/img/article9/token.png
---

OAuth clickjacking attack tricks the user into authorizing malicious application to access user's resources. Clickjacking attack can be very hard to detect, but very harmful for the user. 

## Introduction
Clickjacking, in general, is not OAuth specific attack. Clickjacking is performed by tricking the user to click invisible/transparent element, which performs malicious action, like downloading malware or performing CRSF attack. Website is designed to trick the user to click into the malicious element, by displaying other content under malicious, transparent element, which is fabricated to convince user to take an action.

![clickjacking](/assets/img/article10/clickjacking-img.png)
