---
title: "Amadey Lab - Cyberdefenders"
date: 2025-09-07
weight: 13
draft: false
description: "My writeup of Amadey Lab"
slug: "amadayeLab-cyberdefenders"
tags: ["cyberdefenders", "forensics", "memory-dump", "volatility3"]
---

In this analysis, I will walk through the process of identifying and understanding the behavior of the Amadey Trojan Stealer by examining a Windows memory dump. The analysis will cover identifying the malicious processes, locating malware, and tracing its communication with external servers

## Scenario

An after-hours alert from the Endpoint Detection and Response (EDR) system flags suspicious activity on a Windows workstation. The flagged malware aligns with the Amadey Trojan Stealer. Your job is to analyze the presented memory dump and create a detailed report for actions taken by the malware.

## Solution

> 1. In the memory dump analysis, determining the root of the malicious activity is essential for comprehending the extent of the intrusion. What is the name of the parent process that triggered this malicious behavior?

First, I used **windows.pslist** and **windows.pstree** to list all processes and to visualize the parent-child process relationships: 

```bash
vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.pslist # all processes
vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.pstree # parent child processes relations
```

![image1](https://miro.medium.com/v2/resize:fit:720/format:webp/1*v6piMO8ZHL71YZW6BuI4sA.png)

![image2](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*scaeevBF_tU6lWVVGpEn0w.png)


I found **lssass.exe** to be associated with the suspicious parent process **rundll32**.exe, indicating this as the root of the malicious behavior..

**lssass.exe**

---

> 2.Once the rogue process is identified, its exact location on the device can reveal more about its nature and source. Where is this process housed on the workstation?

Next, I used **windows.filescan** to find the location of the malware:

```bash
/vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.filescan | grep "lssass.exe"
```

**C:\Users\0XSH3R~1\AppData\Local\Temp\925e7e99c5\lssass.exe**

---

> 3. Persistent external communications suggest the malware’s attempts to reach out C2C server. Can you identify the Command and Control (C2C) server IP that the process interacts with?

So, i used **windows.netscan**

```bash
vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.netscan | grep "lssass.exe"
```

**41.75.84.12**

---

> 4. Following the malware link with the C2C, the malware is likely fetching additional tools or modules. How many distinct files is it trying to bring onto the compromised workstation?

I tried to dump malicious Process:

```bash
vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem  windows.memmap.Memmap --pid 2748 --dump
```

So after dumping the process i used strings command to extract HTTP GET Requests:

```bash
strings pid.2748.dmp | grep -A 5 -i "^get /"
```

![image3](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*18PVWoG6KW2Jt-J3Ycvd9w.png)

The answer is: **2**

---

> 5. Identifying the storage points of these additional components is critical for containment and cleanup. What is the full path of the file downloaded and used by the malware in its malicious activity?

From Q4 we known that the attacker downloaded another 2 malicious files named :

- cred64.dll
- clip64.dll

I used **windows.filescan** to find the location of the second malware:

```bash
/vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.filescan | grep "clip64.dll"
```

**c:\Users\0xSh3rl0ck\AppData\Roaming\116711e5a2ab05\clip64.dll**

---

> 6. Once retrieved, the malware aims to activate its additional components. Which child process is initiated by the malware to execute these files?

From question n°5: 

**RUNDLL32.EXE**

---

> 7.Understanding the full range of Amadey’s persistence mechanisms can help in an effective mitigation. Apart from the locations already spotlighted, where else might the malware be ensuring its consistent presence?

```bash
vol.py -f ../../Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.filescan | grep "lssass.exe"
```

**c:\Windows\System32\Tasks\lssass.exe**

---

~ Carbo