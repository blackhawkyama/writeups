---
layout: post
title: "Reading Authorization Out of IP Camera Firmware (And Being Wrong About It)"
date: 2026-11-13 11:30:00 -0800
categories: [methodology, firmware, vulnerability-research]
tags: [firmware, squashfs, aarch64, reverse-engineering, authorization, access-control, apache, suexec, iot, bug-bounty, responsible-disclosure]
---

I spent a week reading a production IP camera operating system to answer one question:
**does every API endpoint actually enforce the privilege level the vendor documents for it?**

This post is the method, not the findings. I filed two reports through the vendor's public
bug bounty programme. One was a duplicate of a report another researcher had filed about
four and a half days earlier. The other was closed pending a working proof of concept,
which I could not produce without hardware. Because one of those issues is somebody
else's open report, **every technical specific of both findings is withheld here**: no
endpoint names, no parameters, no request bodies, no diffs. What is left is the part that
transfers to the next target anyway.

> **Scope note:** the firmware image analysed here is published by the vendor for public
> download and was verified against the vendor's own published hash. The programme is a
> public bug bounty with full safe harbour. All analysis was static and local. No device,
> network or service belonging to anyone else was touched.

## The structural question worth asking first

Before unpacking anything, the useful question is **where authorization lives**. There are
roughly two shapes:

1. **One table.** The web server, a gateway, or a policy engine holds the rules, and every
   request passes through it. Get the table right once and the whole surface is right.
2. **Many decisions.** Each handler decides for itself. There is no table, so there is no
   single place to be right.

Shape two is where access control bugs live, because consistency across a hundred
independent decisions is a thing nobody achieves. So the first job is not finding a bug.
It is finding out which shape you are in.

## Getting the filesystem out

The vendor publishes firmware behind a free account. The image was 152,154,845 bytes.
**Before opening anything, verify the hash the vendor published next to it.** This takes
ten seconds and it is the difference between analysing the vendor's code and analysing
whatever a mirror decided to hand you.

The container chain was ordinary once you follow the magic bytes rather than the extension:

1. **gzip**, which decompresses to a tar archive
2. **tar**, six members, one of which is the root filesystem image
3. **SquashFS 4.0, zstd compressed**, 8,660 inodes

That came out as 7,571 files and 1,097 folders, about 397 MB, a complete Linux root. A
build manifest inside the image named the release, the SoC (an Ambarella CV25) and
therefore the architecture, **aarch64**. Knowing the architecture before you open a binary
saves you guessing at a disassembler.

### ⚠️ The extraction mistake that cost me a wrong conclusion

I pulled the filesystem out with a general purpose archive tool. It worked. Every file was
there, the directory tree was correct, and the contents were byte exact.

**It silently discarded every file mode and owner.**

I then spent two days reasoning about privilege using that tree, and reached a conclusion
that was flatly wrong, because the single most important bit on the single most important
binary was a setuid bit I could not see. The tell was sitting in front of me the whole
time: `/etc/shadow` showed up owned by my own macOS user with mode `-rw-r--r--`, which is
not a thing that ships on any Linux system. I read right past it.

The fix is one flag:

```
unsquashfs -lls rootfs.img
```

**Never reason about permissions from an extraction you have not confirmed preserves
them.** If the modes look like your own umask, they are your own umask.

## Where the authorization actually was

The web server was Apache 2 with a **vendor-written authorization provider** and four
role groups. The server config mapped a handful of coarse directory paths to required
groups, denied everything by default, and granted a couple of media paths outright.

Then the interesting part. **The CGI directory that serves the entire device API carried no
group requirement at all.** It was exposed by a `ScriptAlias` with no gate in front of it.
And the 67 machine readable API manifests shipped alongside it carried no privilege field
either, so there was nothing declarative to compare against.

That answers the structural question. This is shape two: authorization is roughly a hundred
independent decisions made inside individual CGI programs, not one table. Which is exactly
the shape that produces inconsistencies, and exactly why it was worth auditing.

## Building the matrix

With no privilege field in the firmware, the ground truth for "what privilege *should* this
need" has to come from somewhere else. It came from the vendor's own public API
documentation, which states a required role for each endpoint.

So the audit is a join:

| left side | right side |
|---|---|
| 245 endpoints with a documented required role, scraped from public docs | what the shipped binary for that endpoint actually checks |

Any row where those two disagree is a candidate. That is the whole idea, and it is boring
on purpose. The work is in populating the right hand column.

The CGI directory held 233 entries, a mix of shell scripts and ELF binaries. Shell scripts
read directly. For the binaries, roughly a hundred aarch64 objects, the loop was:

1. Parse the ELF sections in Python to find where the code and the string tables live.
2. Disassemble with `objdump`, which on an Apple Silicon Mac already speaks aarch64.
3. Reconstruct the string references. On aarch64 an address is usually built from an
   **`adrp` plus `add` pair**, a page address and then an offset, so a naive string search
   will not connect a constant to the instruction that uses it. Pairing those two
   instructions back together is what turns a disassembly into something you can read for
   authorization logic.
4. Look for the actual decision: the identity check, the group comparison, and the branch
   that returns 401.

What you are hunting for is the shape of a decision, not a keyword. A CGI that checks
nothing looks very different from one that checks and fails closed.

### The control case is the most valuable row in the table

One endpoint family denied unprivileged callers at its service policy layer. Its direct
sibling, with a near identical policy file, did not.

**That pair is worth more than either half.** A single endpoint that fails to check
something could always be explained away: maybe the check is somewhere you have not looked,
maybe the capability is harmless by design. But when the platform demonstrably knows how to
express the restriction, and applies it to one service and not its neighbour, the gap stops
looking like a design decision. Always go find the control case before you write anything
up. If you cannot find one, you probably do not understand the mechanism yet.

## 🪤 The part where I was wrong, in writing, in a filed report

My first report rested on a premise about process identity: that every CGI ran as the same
fixed user, that the suEXEC mechanism was not in play, and therefore that nothing
downstream of the web server could possibly know which HTTP user was calling.

**All three clauses were false.** The platform runs each CGI under a **per role uid**,
selected from the authenticated user's groups. The firmware carries sixteen dedicated
Apache users, one per combination of the four roles, each with the role group as its
primary group. The patched Apache binary computes a **four bit role mask** and uses it to
index that table of sixteen. suEXEC was enabled, and the suEXEC wrapper was setuid root.

Here is why I missed it, because the failure mode is reusable:

1. I checked for a `mod_suexec.so` file in the modules directory. Nothing there.
2. I grepped the Apache configuration directory for `suexec`. Nothing there either.
3. I treated two negative results as proof.

**mod_suexec was statically compiled into the server binary**, and the identity assignment
came from a vendor patch rather than a standard configuration directive. Neither of my
checks could ever have seen it, no matter how many times I ran them. And the setuid bit
that would have given it away was invisible because of the extraction mistake above. One
error hid the other.

Two negative checks on the wrong surface are not evidence. They feel exactly like evidence.
The only defence I know is to ask what a positive result would have looked like, and whether
my check could have produced one at all.

I found this while auditing something unrelated, after the report was already filed. I wrote
the correction up and self reported it rather than leave it for triage to find.

**The conclusion survived.** In fact it got stronger, because the corrected mechanism showed
the platform assigning a distinct identity per role, one sibling service using that identity
to deny unprivileged callers, and the other never consulting it at any layer. The premise
was wrong and the finding was more interesting than I had argued.

## What the week was actually worth

The honest scoreboard: two reports, one duplicate, one closed for want of a proof of
concept I cannot build without buying the hardware. Zero dollars.

What I would take to the next one:

1. **Settle the shape before you hunt.** One table or many decisions. It determines whether
   you are reading config or reading a hundred binaries.
2. **Verify the vendor's hash, then verify your extraction.** Contents are not the only
   thing in a filesystem.
3. **The documentation is the oracle.** When the firmware carries no privilege metadata, the
   vendor's public docs are the only statement of what *should* be required, and the audit
   is a join against them.
4. **Find the control case.** A sibling that does enforce the thing turns an anomaly into an
   argument.
5. **Being beaten by four days is confirmation, not failure.** An independent researcher
   reaching the same conclusion is the strongest external check on an analysis you are ever
   going to get for free.
6. **Report your own errors before triage finds them.** It costs one page and it is the only
   way the next report you file means anything.

A duplicate means somebody else got there first. It does not mean you were wrong.
