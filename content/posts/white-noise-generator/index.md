+++
title = 'The Non-(White) Noise Generator'
date = 2026-02-18T12:53:52+01:00
draft = false
categories = ['Dev']
tags = ['javascript', 'web-dev', 'app']
cover = { image = "cover.png", relative = true, alt = "The Non-(White) Noise Generator" }
+++

## The Problem

Anyone who is a parent knows that all those "humming bears" and shushing toys can be a lifesaver when facing the task of putting a child to sleep. The physical shushing mascot is becoming a bit of a thing of the past these days. Firstly, it’s not necessarily the favorite plushie your little one wants to sleep with, and secondly, there are limitations you have to deal with in that situation. I used to use an app, the name of which I can’t even remember anymore. As long as it worked, I didn't touch it. But... an unnecessary **Android** phone suddenly changed its status to "needed," and its place was taken by an old **iPhone 6** running on **iOS 12**.

Nothing is quite as surprising as the lack of a **free and sensible** alternative for this type of app in the AppStore. Never mind a symbolic dollar or two. But a subscription model for **several** dollars a month?!
There is one more factor contributing to this problem: the SDK version.

| Aspect              | iOS 12 SDK (Xcode 10)          | iOS 26 SDK (Xcode 26)              |
|---------------------|--------------------------------|------------------------------------|
| Base Version        | iOS 12.0 (2018)                | iOS 26.0 (2025)            |
| Deployment Target   | iOS 12.0 and up                | iOS 12.0+ (build with Xcode 26+) |
| New Frameworks      | ARKit 2, GroupFaceTime         | SwiftUI 6+, Apple Intelligence, VisionKit |
| Deprecated API      | -                              | UIWebView, NSURLConnection |
| App Store Reqs      | Outdated since 2026    | Mandatory from April 2026 |
| Supported Devices   | iPhone 5s+                     | iPhone 12+ (AI features)  |
| Key APIs (13+)      | Unavailable (Sign in, Dark Mode)| Available via @available checks |

**So, let's summarize my needs:**
1. The app must work on iOS 12.
2. We don't want a subscription; a symbolic, one-time fee is acceptable.
3. The app must be ad-free—we don't want to wake the heir with a lady waving a Revolut card around.
4. A programmable timer with a smooth fade-out.
5. The ability to choose sound characteristics—I learned about this need a bit later, but we'll get to that.

## Let's write the app ourselves!

However, I don't have an Apple Developer account. I've had one on Google Play for ages. I don't have anything with Android on hand—infinite loop.
But here comes an iOS feature that we almost lost due to absurd EU regulations—**PWA**.

>**Progressive Web App (PWA)** is a web application that combines the advantages of websites with the functionality of native apps. It runs in the browser (HTML, CSS, JavaScript) but installs like an app on the home screen. Service Workers enable offline work and fast loading via caching. Push notifications, access to camera/GPS, and an icon right next to native apps.

And the best part is that PWAs are available from iOS 11.3 upwards!

### GUI

You can see the result right now in the screenshot below. The assumptions were simple—dark mode, because the app is mainly intended for use after dark, and simplicity. No unnecessary selection from dozens of sounds or mixing them together. No settings menu whatsoever—just one screen containing what is needed.

![GUI screen](/posts/white-noise-generator/maingui.png#center#50)

Finally, I even display the time remaining until the noise stops in MM:SS format, even if I have 176:33. It doesn't bother me. I won't fix it anymore—if it works, there's no point in touching it.

It is worth remembering the appropriate tags in the *head*:

```html {linenos=true}
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <link rel="apple-touch-icon" href="icon.png">
</head>
```

### Backend

And here is where much more is happening. First challenge—iOS cannot kill JavaScript after the screen goes dark. Neither immediately, nor after an hour or longer.
Browsers drastically limit resources for inactive tabs. Classic `setInterval` can start to "lag," making the on-screen timer unreliable.

We solve this in two ways:
- Audio plays on: The Web Audio API runs on a separate thread, so the sound won't be interrupted.
- Timestamps: Instead of counting seconds, we save the end time (`Date.now() + duration`). Every time we return to the tab (`visibilitychange`), we refresh the UI based on the current system time.

```ts {linenos=true}
document.addEventListener("visibilitychange", () => {
    if (!document.hidden && globalEndTime > 0) {
        refreshTimerUI();
    }
});
```
### Math

The most interesting element is generating the noise itself. Initially, I wanted to spend a few moments looking for a high-quality few-second loop of white noise. But sound is just pure mathematics. Especially something as simple as randomly occurring values.

**White noise** is a set of randomly generated values ranging from -1 to 1. This range represents two states of the speaker membrane—maximally retracted and maximally extended. Everyone has surely had the opportunity to observe membrane movement, especially visible during a heavy bass hit (which is often accompanied by a strong blast of air).

Before we generate our array, we must define the buffer size. I decided to *reserve* space in the RAM for 2 seconds of sound. This is the optimal value between wastefulness and tricking the brain so it doesn't catch a repeating sound pattern.

```ts {linenos=true}
const bufferSize = 2 * audioCtx.sampleRate;
```

2 * 44100 = 88,200 Hz
That’s exactly how many samples we will generate when the sound starts, save them in RAM, and then play them in a loop.

Let's proceed to generate the samples. We'll do this in the simplest loop, which will repeat until we fill the entire array of our buffer with values.

```ts {linenos=true}
for (let i = 0; i < bufferSize; i++) {
    data[i] = Math.random() * 2 - 1;
}
```

`Math.random()` itself provides values from the range 0.0 - 1.0. We expand this range twofold by multiplying `Math.random()` **x 2**, which gives an array of random values with a range of 0.0 - 2.0. In the last step, we step back by 1: `Math.random() * 2` **- 1**, and we have a set of values ranging from -1.0 to 1.0 :)

**You might not realize what we achieved here, so let's stop for a moment to summarize. The moment the user clicks *START*, the iPhone processor generates 88,200 random values, which takes about 2-3 ms, and saves them in RAM.**

**88,200 samples x 4 bytes (Web Audio API standard) is merely 344.5 KB of data!**

If, instead of a 2-second loop that we play over and over, we generated data on the fly, an hour of synthesizing such noise would lead to generating approx. **600 MB** of data. That's almost as much as a CD.

White noise, however, isn't quite what I expected. The sound is not very pleasant due to the excessive randomness of consecutive generated samples. The solution is...

**Brown noise**, which differs from white noise in that it applies a *random walk* instead of full randomness.
Brown noise *remembers* the previous sample. Therefore, it has an influence on the next one, which in turn influences the next... and so on until the end. The jumps are much smaller, which makes the synthesized sound more pleasant.

```ts {linenos=true}
let lastOut = 0.0;
for (let i = 0; i < bufferSize; i++) {
    let white = Math.random() * 2 - 1;
    lastOut = (lastOut + (0.02 * white)) / 1.02;
    data[i] = lastOut * 3.5;
}
```
After initializing the *lastOut* variable, during each iteration, we assign the value of the expression *lastOut + (0.02 * white) / 1.02*.
In short, we add only a small fragment of the new one to the previously generated sample. White noise simply substitutes a whole new value.

### What else?

I won't hide that the biggest challenge was preventing the app from sleeping, while the aspects of mathematical sound generation are interesting enough to me that they pushed me to write this article.
The app wouldn't be complete without details. After all, while the sound is playing, we might want to increase its duration because the baby is waking up. The sound should start smoothly and fade out just as smoothly so as not to wake our sleepyhead. This, in turn, can lead to a *collision* if the fade-out time and volume ramp-up time overlap.
This is already simple mathematics with dynamic calculation of current variable values. It's something to think about, but nothing to debate.

```ts {linenos=true}
function scheduleAudioFade(durationSec, startTime) {
    let fadeDuration = Math.min(durationSec, 180);
    let fadeStart = startTime + (durationSec - fadeDuration);
    
    const fadeInEndTime = startTime + 4;
    if (fadeStart < fadeInEndTime) {
        fadeStart = fadeInEndTime; 
        fadeDuration = (startTime + durationSec) - fadeStart;
    }

    if (fadeDuration > 0) {
        gainNode.gain.setValueAtTime(1, fadeStart);
        gainNode.gain.linearRampToValueAtTime(0.0001, fadeStart + fadeDuration);
    }
}
```

## Summary

iOS 12 is, on one hand, a stumbling block—old APIs, outdated requirements—but on the other hand, Apple was less restrictive back then, which turned out to be somewhat of a salvation.
The solution, although tailor-made and unsupported by newer devices, fulfills its purpose. Stability was wildly important, and that was also achieved. Additionally, the adventure with mathematical sound generation provided a large dose of knowledge and exploration of a new area. A fascinating area, because who knows what other programming experiments could be performed during future home projects?