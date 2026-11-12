# Attach Doctor

A small tool that reads the signalling from a failed 4G/5G attach and tells you,
in plain words, why the device didn't connect and what to check. It shows the
evidence it used and how sure it is.

It's built for small private 4G/5G cores like AttoNGC, not as a general packet
analyser.

## The problem

On a private 4G/5G network, when a device won't connect, the reason is hidden in
the messages between the device, the core, and the subscriber database. It might
be a failed login, a missing subscription, or the wrong APN. Reading those
messages needs someone who knows mobile core networks, and most teams running
these networks don't have that person. So the device just doesn't work, and
nobody can say why without a lot of manual digging.

## What it does

You give it the signalling from an attach attempt and it tells you what happened.

A SIM that logged in fine but has no data subscription:

```
Status: FAILED  (confidence: confirmed)
  Failure stage : Subscription (location update)
  Diagnosis     : The SIM is known and logged in, but has no data (EPS)
                  subscription, so the core won't let it on.
  Check         : Add an EPS/APN subscription for this SIM in the HSS/UDM
  Evidence      : Update-Location-Answer -> UNKNOWN_EPS_SUBSCRIPTION (result 5420)
```

A capture with nothing to go on:

```
Status: UNKNOWN  (confidence: insufficient)
  Diagnosis : The start of the attach wasn't in the capture, so we can't say
              why it failed.
  Missing   : the first request message
```

## What makes it useful

- One common way of reading every message, so 4G and later 5G can be handled the
  same way without losing detail.
- It tells you how sure it is: confirmed (it saw a clear "no"), possible (like a
  timeout, where a request got no reply), or not enough evidence. If it can't
  tell, it says so instead of guessing.
- It handles the tricky cases that trip up most tools. A timeout is not a
  rejection. Repeated messages are counted once. If several things fail, it
  reports the first real one and lists the rest. A reason code it doesn't know is
  shown as it is, not guessed. A failure that works on retry is reported as
  recovered.
- Privacy: the subscriber's identity is hidden, and secret keys are never read.

## Why it fits AttoNGC

AttoNGC is a combined 4G and 5G core with one shared subscriber database. It runs
on normal servers, small cells, and Kubernetes, supports VoLTE and VoNR, and is
used in private, often offline networks. That shaped the tool:

- The main daily problem for teams running these networks is "a device won't
  connect and I can't tell why." That is what this explains.
- A shared subscriber database makes one thing possible that only fits a combined
  core: checking across generations, like a SIM that works on 4G but fails on 5G.
  (A planned direction, not a claim that the product has this problem.)
- It runs locally and offline with a small footprint, to suit these deployments.

## What's real now, and what's next

Real now: the engine, the "how sure" levels, and all the tricky-case handling
above. The subscriber-database (Diameter S6a) messages are decoded from real
bytes, so those result codes are genuine. There are tests.

Simulated: the example cases are made up, built on real 3GPP reason codes (the
S6a bytes are real). No production data.

Next, planned and not faked: reading real captures (pcap or Open5GS), decoding
the phone-side and radio messages in full, 5G, and an adapter for a real core
built around what the core can actually share.

This is an early prototype. It shows the design works and that the 4G path gives
a real, evidence-based answer. It is not production-ready.

## A few questions for your team

1. On a running core, what signalling can be seen: a packet capture, structured
   logs, or an event feed?
2. Which interfaces are visible in a typical private deployment?
3. For checks across generations, can per-generation subscriber config be read,
   or only worked out from signalling?
4. What limits should it be designed for: offline networks, small footprint on
   embedded deployments, data retention?
5. Which case would be most useful to diagnose first: 4G IoT, 5G private, or
   VoLTE/VoNR?

*(The working code is kept separately and will be shared on request.)*
