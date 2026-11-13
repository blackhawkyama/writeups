---
layout: page
title: About
permalink: /about/
---

## Scott Egan

Offensive security: penetration testing, AI and ML red teaming, vulnerability research.
Based in Rancho Santa Fe, CA. **BlackhawkYama** is the handle I publish lab and research
work under.

My day job is owning security, risk and continuity for a production AI platform, and doing
the hands on offensive work alongside it. Outside that I do independent vulnerability
research on public bug bounty programmes, strictly within scope and safe harbour, and I
write up the method here.

### What I actually work on

- **Web and API penetration testing.** Access control, IDOR and BOLA, authorization and
  authentication bypass, SSRF, SQLi, GraphQL. Burp Suite, Nmap, ffuf, sqlmap.
- **AI and ML security.** Indirect prompt injection, jailbreak resistance, and adversarial
  testing of agentic systems. I have run measured prompt injection studies against
  anonymised frontier models across two competition waves, documenting each attack class,
  its result, and which defence actually held.
- **Firmware and binary analysis.** Unpacking vendor firmware, aarch64 disassembly, and
  auditing endpoint authorization against the privilege a vendor documents. The write up
  of that method is [here]({{ '/posts/reading-authorization-out-of-camera-firmware/' | relative_url }}).
- **Security tooling.** An LLM evaluation framework that scores output against versioned
  test cases and blocks regressions before release, a manipulation resistance suite, an
  autonomous reconnaissance agent for authorized targets, and a differential tester for
  path authorization bypass in API gateways. I design and direct these; the
  implementations are AI assisted and I say so.

### Selected work

- Found and reproduced a memory safety flaw (**CWE-125**, out of bounds read) in a widely
  used open source AI runtime, with a working proof of concept and a patch. Since fixed
  upstream.
- Authored reconnaissance extensions to AI security testing frameworks maintained by
  **NVIDIA** and **Microsoft**, written to each project's own architecture, and proposed
  upstream.
- Reverse engineered a production IP camera operating system and mapped 245 documented API
  endpoints against the privilege level the vendor publishes for each. Two reports filed;
  one was independently corroborated by another researcher who filed the same issue days
  earlier. Authorized testing, no claimed bounty payouts.
- As security owner for a production AI platform: found three live gaps, confirmed each one
  safely rather than assuming it, and stayed on them until fixed and verified.

### Training and certifications

- **OSCP**, in progress
- **eJPT**, studying
- Hack The Box Academy, Penetration Tester path
- PortSwigger Web Security Academy
- TryHackMe, Penetration Tester path
- United States Marine Corps

### Elsewhere

- **LinkedIn:** [linkedin.com/in/egan-scott](https://www.linkedin.com/in/egan-scott)
- **GitHub:** [@blackhawkyama](https://github.com/blackhawkyama)
- **Email:** scott@blackhawklabs.dev

> All testing performed in legal, authorized environments: systems I am responsible for,
> personal labs, sanctioned exercises, and in scope bug bounty programmes. Flags are
> redacted and live target addresses are masked.
