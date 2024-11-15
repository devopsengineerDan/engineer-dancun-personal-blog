+++
title = "GOAD Part 1: AD Recon, Password Spraying, ASREPRoasting & LLMNR Poisoning"
date = 2024-11-14T01:29:20+03:00
tags = ["cybersecurity"]
#Decide to delete draft or change it to false in order to display post on website
+++

---

![goad_part1_1](https://github.com/user-attachments/assets/03da8b24-ab3c-4264-b6dc-2851c0840b8c)

> ### Table of Contents

- [Introduction](#introduction)
- [Network Diagram](#network-diagram)
- [Reconnaissance and Enumeration](#reconnaissance-and-enumeration)
  - [Hosts file setup](#hosts-file-setup)
  - [Domain User enumeration](#domain-user-enumeration)
  - [Domain Group enumeration](#domain-group-enumeration)
- [Valid Usernames](#valid-usernames)
  - [Password Spraying](#password-spraying)
  - [AS-REPRoasting](#as-reproasting)
- [No Valid Credentials or Usernames](#no-valid-credentials-or-usernames)
  - [LLMNR Poisoning](#llmnr-poisoning)
    - [Listen and capture hashes](#listen-and-capture-hashes)
    - [Crack with hashcat](#crack-with-hashcat)
    - [Find targets for relaying](#find-targets-for-relaying)
- [Appendix](#appendix)
  - [Valid Credentials](#valid-credentials)
  - [Valid Users](#valid-users)

## Introduction

- [Game of Active Directory](https://github.com/Orange-Cyberdefense/GOAD) is a fully functional AD lab environment, misconfigured with several AD issues designed to help understand various AD security concepts.
- In this n-part series, we will explore how we can abuse the misconfigurations. In part 1, we focus on `enumerating` the environment to find `domains, domain controllers, usernames and groups`. We further leverage this to conduct various attacks, showcasing techniques like `password spraying & ASREPRoasting`.
- We then finish off by exploring what attacks can be carried out when we have `no credentials or usernames` to work with such as the `LLMNR Poisoning and NTLM relay`.

## Network Diagram

![goad_part1_2](https://github.com/user-attachments/assets/758bc9f7-7337-45e7-8ab5-3bff783f6d5f)

## Reconnaissance and Enumeration

- From our network diagram, we are dealing with 5 different machines across 3 domains in the network.

1. `north.sevenkingdoms.local`
   - DC01 (kingslanding)
2. `sevenkingdoms.local`
   - DC02 (winterfell)
   - SRV02 (castleback)
3. `essos.local`
   - DC02 (mereen)
   - SRV03 (braavos)

- The creators of GOAD - Orange Cyberdefense - have also come up with [this](https://orange-cyberdefense.github.io/ocd-mindmaps/) mindmap that structures our AD pentesting approach. We will frequently refer to this resource when exploring the lab.

> Example1 of Active Directory Penetration Testing

![pentest_active_directory_1](https://github.com/user-attachments/assets/a2c12381-5182-487a-afb2-04d03e36b796)

> Example2 of Active Directory Penetration Testing

![pentest_active_directory_2](https://github.com/user-attachments/assets/8e8ac296-202b-4a54-be98-d3b600bc4675)
