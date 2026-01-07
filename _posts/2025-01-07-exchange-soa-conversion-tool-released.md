---
title: "Exchange SOA Conversion Tool - Now Available"
date: 2025-01-07 06:54:00 +0100
categories: [Microsoft, Exchange Online]
tags: [exchange, exchangeonline, exchangehybrid, cloudmigration, powershell, automation, entraId]
---

Following up on my [previous post](/posts/exchange-soa-conversion-tool-teaser/) about cloud Exchange recipient management, I'm releasing the first version of a tool that makes this transition much easier.

![Exchange SOA Conversion Tool](/assets/img/posts/exchange-soa-tool-gui.png)
_The Exchange SOA Conversion Tool_

## What it does

The Exchange SOA Conversion Tool is a PowerShell GUI that simplifies converting directory-synced mailboxes between on-premises and cloud-managed Exchange attributes. Instead of running individual PowerShell commands for each user, you get a visual interface to manage conversions at scale.

## Key features

- Clean, modern GUI interface
- Batch conversion support (multi-select users)
- Pagination for large environments
- Automatic Exchange Online Management module installation
- Built-in logging and Exchange Online connection management

## What's next

I'm working on detailed blog posts that will dive deeper into:

- Why this tool matters for hybrid environments
- Real-world use cases and scenarios
- Step-by-step walkthrough and demo
- The path to retiring your last Exchange server

## This is v1.0, and I'd love to hear from you

- What features would be most valuable to you?
- What challenges are you facing in your hybrid environment?
- Ideas for improvements or additional functionality?

Feel free to try it out, open issues on GitHub, or reach out directly.

## Get the tool

You can find the Exchange SOA Conversion Tool on my GitHub:
[https://github.com/codeandersen/Exchange-SOA-Conversion-tool](https://github.com/codeandersen/Exchange-SOA-Conversion-tool)

---

This is a piece of the puzzle on the path to minimizing the need for on-premises Exchange Server. I will be creating more tools and blog posts about this subject.

*Questions or feedback? Reach out on [Twitter](https://x.com/dk_hcandersen) or [LinkedIn](https://www.linkedin.com/in/hanschrandersen).*
