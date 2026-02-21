# RepMentor — Case Study

**Full-Stack Mobile App | Solo Developer | React Native, Expo, Supabase, OpenAI**

---
<img width="1242" height="960" alt="Screenshot 2026-02-17 at 12 05 21 PM" src="https://github.com/user-attachments/assets/e497ddee-3c22-4e98-b242-ab5e434b7b0f" />

---

## The Problem

I was already using ChatGPT to help with my training. With things like building programs, reviewing my progress, asking questions about recovery and volume, it was genuinely useful. But every single session involved the same friction: copy my workout log from one app, paste it into a chat window, explain the context, ask my question, then go back and manually apply whatever the coach suggested.

The data existed. The intelligence existed. They just lived in separate places with me as the human bridge between them.

The obvious solution was to unify them. But that created a harder problem: **to get useful data into an AI coach, you need a workout tracker people actually use.** Not just functional but genuinely enjoyable. If logging a set feels like admin work, people skip it, the data degrades, and the AI has nothing meaningful to work with. The tracker had to earn its place in the routine before the coach could earn its.

So I built both.

---

## The Goal

A single app where:
1. Logging a workout is fast and frictionless enough that you never skip it
2. That data flows directly into an AI coach that already knows your history, your goals, and your recent sessions
3. No copy-pasting. No context-switching. No manually explaining what you've been doing.

---

## The Tracker Had to Come First

Before any AI work, I had to solve the UX problem. A coach is only as good as the data feeding it, so the tracker needed to be something people would actually use consistently.

Key decisions that shaped the experience:

**Previous session comparison** — Every exercise shows your last logged weight and reps inline while you're logging. You don't have to remember what you did. You just beat it or don't.

**PR notifications** — The app detects personal records in real time and fires a haptic + toast. A small thing, but it makes logging feel rewarding rather than administrative.

**Frictionless set entry** — Minimal taps to log a set. The session screen is designed around the reality of being in a gym: one hand on your phone, limited attention, time pressure between sets.

**Health sync** — Weight and waist measurements pull automatically from Apple Health or Health Connect. The data you already have doesn't need to be re-entered.

The tracker needed to be good enough to use even if the AI coach didn't exist. That was the bar.

---

## The Watch Was Non-Negotiable

A gym session has a rhythm. You finish a set, you're resting, your phone is in your bag or across the room. Pulling it out to log every set breaks that rhythm. If logging feels like an interruption, people stop doing it. The data degrades, and the AI coach has nothing real to work with.

An Apple Watch companion wasn't a nice-to-have. It was essential to the core promise of frictionless tracking.

Building it, however, meant taking on one of the more genuinely difficult problems in the project: keeping two independent devices in sync during a live workout, with no guaranteed network, across a communication layer (WatchConnectivity) that has its own rules about when and how messages are delivered.

A few of the challenges worth calling out:

**Reliable message delivery** — `WCSession.sendMessage` only works when the phone is reachable and in the foreground. For set completions — the most critical data in the system — the Watch uses a dual-send strategy: `transferUserInfo` queues the payload and guarantees delivery when the app next runs, while `sendMessage` attempts immediate delivery in parallel. No completed set gets lost.

**Timer sync without drift** — Rest timers need to read identically on both devices. Rather than each device counting down independently from when it received a message, both derive the countdown from a shared `endTime` Unix timestamp. Every tick recalculates from that absolute reference, so clock skew and delivery delays don't accumulate into visible drift.

**Stale state and race conditions** — When the Watch completes a set and starts a rest timer locally, the iPhone will shortly send a `timer_update` reflecting its own view of the world — which may not yet include that completion. Without careful flagging, the Watch could double-navigate or show stale "next exercise" data. This required tracking which device initiated each state transition and ignoring updates that arrived out of sequence.

**NSNull and property list compliance** — `updateApplicationContext` requires strict property-list types. Swift's WCSession rejects payloads containing `NSNull`, which JSON serialisation produces freely for optional fields. A recursive `stripNulls` pass over every outgoing payload was the unglamorous but necessary fix.

The Watch app itself is a full SwiftUI app built as a native Xcode target, communicating with the React Native layer through a custom Expo native module wrapping `WCSession`. The result is that a user can go an entire workout touching only their Watch — logging sets, viewing rest timers, seeing what's next — with everything reflected on their phone (and vice verca) in real time.

---

## Then the Coach

With reliable, structured data in a local SQLite database, the AI layer became straightforward to make genuinely useful.

When you open the coach, it isn't starting from zero. It has access to your actual routines, your recent sessions, your body measurements, and your stated goals. This is huge when looking for targed, actionable feedback.

The coach uses OpenAI's tool-calling API, meaning before it responds it can programmatically look up your routines and recent workout history. The response isn't based not on what you described to it, but  what you actually did.

Responses stream in real time, so the interaction feels like a conversation rather than a form submission with a loading spinner.

The API key never touches the client. Requests are proxied through a Supabase Edge Function that validates the user's session JWT before forwarding to OpenAI. This is standard practice for keeping model access gated behind real authentication.

---

## Technical Stack

| Layer | Technology |
|---|---|
| Mobile | React Native + Expo (SDK 52) |
| Local DB | SQLite via expo-sqlite |
| Backend | Supabase (Auth + Edge Functions) |
| AI | OpenAI GPT via streaming Edge Function |
| Health | Apple HealthKit, Google Health Connect |
| Watch | Apple Watch — SwiftUI + WatchConnectivity |
| Analytics | PostHog |
| CI/CD | EAS Build + EAS Submit |

---

## Engineering Challenges Worth Noting

**Local-first with versioned migrations** — All workout data lives on device. No internet required to track a session. The schema is versioned with a hand-rolled migration system (`SCHEMA_VERSION: 14`) that safely applies deltas on startup, which became non-trivial when adding FK relationships mid-development with existing user data.

**Streaming + tool use UX** — Displaying tool call results ("Looked up your routines...") mid-stream while the model is still generating requires careful state management to avoid flickering or out-of-order renders.

**Cross-platform health integration** — Apple Health and Google Health Connect have meaningfully different APIs and permission models. A shared abstraction layer normalises them so the UI never has to think about the platform.

**React Native New Architecture** — Required by `react-native-reanimated` v3 on Android, which surfaced a `minSdkVersion` conflict with the Health Connect library and required debugging how Expo's Gradle plugin sets root project ext values.

**Custom Expo native module** — WatchConnectivity isn't exposed by Expo out of the box. The Watch sync layer is built on a custom native module written in Swift that wraps `WCSession` and exposes it to the React Native JS layer with a typed TypeScript API.

---

## What This Demonstrates

- Identifying a real personal problem and following it to a full product
- End-to-end ownership: schema design, mobile UI, native Watch app, backend, AI integration, and CI/CD
- UX thinking as a prerequisite to AI usefulness — the model is only as good as the data
- Native platform development across React Native, SwiftUI, and Kotlin/Gradle
- Secure AI integration with streaming and tool use across a real authentication boundary
- Production delivery on both iOS App Store and Google Play
