---
title: "DNSSEC Resolver Test"
source: "https://wander.science/projects/dns/dnssec-resolver-test/"
author:
  - "[[Matthäus Wander]]"
published:
created: 2024-11-21
description:
tags:
  - "clippings"
---
**This web-based test checks whether your domain name lookups are protected by DNSSEC.**

![Test image](https://wander.science/projects/dns/dnssec-resolver-test/derp.png)

## How It Works

The test first attempts to load an image from a URL with a broken DNSSEC signature. This is expected to fail, because your resolver should refuse the bogus domain. The test then attempts to load another image from a URL with a valid DNSSEC signature, which is expected to succeed. If it fails too, the test is inconclusive (see below).

## Questions and Answers

### The test is successful. Am I safe?

It depends. Though a DNSSEC-enabled resolver protects you from DNS forgery, it is unclear whether you have a secure communication path with that resolver. Consider the following scenarios:

1. You are running a DNSSEC-enabled resolver on your local machine. You are safe.
2. You are using a DNSSEC-enabled resolver in your local network or you are using your ISP’s resolver. This requires a trusted network communication to the resolver. If there is an attacker in your network or your ISP’s network, you are susceptible to DNS spoofing.
3. You are relying on a public resolver (like Google Public DNS, Quad9, OpenDNS, etc.) for DNSSEC validation. You need a secure connection to the public resolver, for example, with [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS) or [DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS). Alternatively, enable DNSSEC validation on your local machine.

Furthermore, this test only checks the security of DNSSEC name resolution, not of any connections like HTTPS.

### The test failed. Am I at risk?

Your system is susceptible to DNS forgery. This makes you vulnerable to spoofing and phishing attacks, if your applications do not assert the authenticity of the communication peers. Cryptographic network protocols like TLS or SSH provide such an authenticity assertion. In other words: if the website uses HTTPS, your web browser will be able to detect an insecure connection, even though the name resolution may have been misguided.

### I use a validating resolver, but the test fails. Why?

The most common reason is that you use more than one resolver, and one of them does not validate DNSSEC signatures. Consequently, your web browser will be not protected by DNSSEC. The reason lies in how name resolution failures are handled: the DNSSEC resolver returns a SERVFAIL error when the DNSSEC signature fails validation. The SERVFAIL response does not convey the reason of the failure. Thus, the client will typically repeat the name resolution with the next resolver configured, until all of them have been exhausted and failed. In order to protect the client from DNS forgery, all resolvers configured by the client must validate DNSSEC.

### Why is the test inconclusive?

Something prevents your browser from loading the image with the valid DNSSEC signature. Possible reasons include:

- the nameserver might be temporarily unreachable
- there is a network problem
- your web browser refuses to load external images, e.g., due to an adblocker

### How to repeat the test?

Wait one minute to make sure the DNS records have expired from cache. Restart your web browser or do a hard reload (ctrl+F5) to make sure the browser cache is flushed.

### How to enable DNSSEC validation?

- [BIND](https://www.isc.org/dnssec/): Since v9.13.1, DNSSEC validation is enabled by default. You can set it explicitly with: `dnssec-validation auto;`
- [Unbound](https://nlnetlabs.nl/documentation/unbound/howto-anchor/): Obtain the [Root Zone Trust Anchor](https://www.iana.org/dnssec/files) with the tool `unbound-anchor`. Make sure your unbound.conf points to the anchor, e.g.: `auto-trust-anchor-file "/var/lib/unbound/root.key"`

I cannot comment on how to enable DNSSEC on end-user operating systems. If you know a helpful article, please drop me a message.

### How to test DNSSEC validation on the console?

- `dig sigok.ippacket.stream` should return an [A record](https://wander.science/projects/dns/dnssec-resolver-test/dig-sigok.txt). Note the `ad` flag from the resolver (authenticated data = DNSSEC validation was successful).
- `dig sigfail.ippacket.stream` should return a [SERVFAIL error](https://wander.science/projects/dns/dnssec-resolver-test/dig-sigfail.txt).

## Background

This test was created in May 2012 to measure the adoption of DNSSEC validation in the web. It was running on [dnssec.vs.uni-due.de](https://dnssec.vs.uni-due.de/) until the University suffered from a ransomware attack in November 2022. In March 2023, it was relaunched on [wander.science](https://wander.science/projects/dns/dnssec-resolver-test/). Currently, it does not collect any data.

### Papers and Talks

- Measuring Occurrence of DNSSEC Validation (2013)  
[Paper](https://wander.science/paper/2013_Wander_MeasuringDNSSEC.pdf) [BibTeX](https://wander.science/paper/2013_Wander_MeasuringDNSSEC.bib) [Slides (HTML)](https://wander.science/talks/20130319_DNSSEC_Validation) [Slides (PDF)](https://wander.science/talks/20130319_DNSSEC_Validation.pdf)
- Measuring Occurrence of DNSSEC Validation (2012)  
[Slides (HTML)](https://wander.science/talks/20121014_DNSSEC_Validation) [Slides (PDF)](https://wander.science/talks/20121014_DNSSEC_Wander.pdf)

## Other Tests

These tests use slightly different mechanics, which may lead to different results in corner cases.

- [www.dnssec-or-not.com](http://www.dnssec-or-not.com/): online test by @PacketPusher (no JavaScript required)
- [internet.nl/connection](https://internet.nl/connection): online test by Dutch Internet Standards Platform