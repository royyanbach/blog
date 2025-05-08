---
title: 'When Popups Pop Too Late'
description: 'A rant about popups that appear too late, disrupting the user’s flow and intent'
pubDate: 2025-05-07
category: 'tech'
---

I opened a local Indonesian banking app the other day. I was in a rush—just wanted to check my balance. The app loaded quickly, and I tapped into the “Accounts” tab. Everything seemed smooth... until *bam*—a popup slid into view **right after** I had already started navigating.

I didn’t mis-tap. I didn’t trigger anything. It just popped up, uninvited, 2–3 seconds after I had landed.

Here’s a screen recording of what I’m talking about:

<video controls class="w-full rounded-lg shadow-lg">
  <source src="/kai-popup.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## The Problem with Async Popups

The popup itself isn’t the problem. Promos and announcements are fine. But *timing* matters. The issue is that the popup comes in **too late**, disrupting the user’s intent. It breaks flow. Worse, sometimes it overlaps with the screen I’m already on, causing a jarring visual glitch or dismissing my keyboard input.

It’s likely caused by some asynchronous logic—maybe the app fetches popup content from the backend and renders it as soon as it’s available, regardless of current context.

## How This Could Be Better

There are better UX patterns:

* **Delay the popup until the user returns to the home screen**, or at least…
* **Don’t interrupt an ongoing interaction** (taps, scrolls, keyboard input).
* **Cache the popup early** during the splash screen and decide later whether to show it at all.

This isn’t a hard problem—it’s just a matter of respecting the user’s journey. Not every backend call needs to show a result the moment it resolves.

## Final Thought

In product teams, these decisions often go unnoticed—"just show the promo when it's ready." But these tiny frictions add up. If your users groan while using your app, you've already lost.

Has anyone else seen similar patterns in local apps lately? Would love to compile more cases.
