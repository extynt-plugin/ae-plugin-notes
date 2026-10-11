# Contributing

The most useful contribution is a **measurement on a setup we have not
tested**. Everything here was measured on After Effects 26.2.1, macOS, Apple
M4 Pro. We would like to know what changes on:

- other AE versions (25.x, earlier 26.x, betas), especially for the absence of `isHidden`,
  `isModified`, `app.project.revision`, SVG import, and AEGP suite versions
- Windows (idle-hook latency, `evalScript` floor, `$.evalFile` scoping)
- Intel Macs and other Apple Silicon generations
- projects of very different size

## What a good report contains

1. The claim you are confirming, refuting or adding.
2. Environment: AE version and build, OS version, CPU, CEP version if relevant.
3. Method: how you timed it (`$.hiresTimer`, `performance.now()`, native
   clock), number of repetitions, warm-up, whether you report median or mean.
4. The numbers, ideally with p50 and p95.
5. A short script or snippet if it is under about 20 lines (MIT, see LICENSE-MIT).

## Rules

- Measured facts and techniques only. Say what you observed, not what you
  expect.
- Do not paste Adobe SDK files or large SDK excerpts. Name functions and
  suites instead.
- No third-party plugin data, and no personal data (hostnames, usernames,
  machine paths).
- Keep each note focused; add a "Measured on" line for every new result.

Open a pull request or an issue. By contributing you agree your text is
released under CC BY 4.0 and your snippets under MIT.
