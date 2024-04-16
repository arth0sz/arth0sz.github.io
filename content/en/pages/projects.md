---
title: GitHub Projects
description: 'Github Projects'
author: arth0s
toc: false
---
---

## [Practice ADCS Domain Escalation](https://github.com/arth0sz/Practice-AD-CS-Domain-Escalation)

Introductory guide where we will configure our own Windows Server with a Certificate Authority manually and with the help of PowerShell, and subsequently exploit Active Directory Certificate Services with Certipy. Based on the white paper [Certified Pre-Owned](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf). Covers ESC1, ESC2, ESC3, ESC4 and ESC7.

---

## [Bash script for AD network scanning with Nmap](https://github.com/arth0sz/AD-network-scanning-script)

This is a script I put together to deal with scanning an Active Directory environment with Nmap after going through the Active Directory Enumeration & Attacks module on HackTheBox.

It takes a network range in CIDR notation as the only command-line argument and goes looking for port 88 and 445 to find active hosts. Then it scans them for the 1000 common ports and performs a more detailed script and version scan on all open ports.

The results are saved for each IP in the three major formats for Nmap along with being output to the terminal.

---

## [Bash script to make your screenshots report-ready](https://github.com/arth0sz/Report-ready-Screenshots)

A simple bash script to convert your dark-mode screenshots into light mode and add a black border around them, so you can happily paste them into your report. It utilises the open-source ImageMagick software suite.

Additionally, if you have screenshots that are in light mode, you can separate them out and use the light-mode script instead to only add a black border to the screenshots.

---