# 🧾 License Guide: A Beginner-Friendly Overview

A historical and practical guide to understanding software licenses — especially useful for smart contract developers and auditors.

---

## 🧠 Introduction: Code as a Commons, Then Commodity

In the 1960s–1970s, most software was developed in universities and government labs. Code was freely shared like academic research — often bundled with hardware (e.g., UNIX from Bell Labs).
But with the rise of personal computing in the 1980s, software became a commodity. Corporations locked down their code, triggering resistance from figures like **Richard Stallman**, who launched the **Free Software Foundation** and the **GPL license**.

By the 1990s, the **Open Source** movement emerged as a more pragmatic alternative — balancing freedom and commercial utility. Permissive licenses like **MIT** and **BSD** became dominant.

---

## 🏛️ 1. The Early Days: Science and Sharing

* Code developed with public funding (e.g., ARPA in the US)
* Distributed via tapes, books, and in-person networks
* Ideology: software as knowledge → collaboration was key
* Example: UNIX source code was originally distributed with full rights to modify and learn

---

## 💸 2. The 1980s: Capitalism Meets Code

* Rise of Microsoft, Oracle, Adobe
* Source code became proprietary
* Reverse engineering banned, licenses enforced
* In response, **Free Software** (FSF, GNU) emerged to protect users’ freedoms

**Notable event:**
🧠 *1984*: Richard Stallman creates the **GPL (General Public License)** — a “copyleft” license that requires all modified versions to also remain free.

---

## ⚖️ Free vs Proprietary (Then and Now)

| Feature                 | Free Software (FSF/GNU)   | Proprietary Software (Microsoft, Oracle) |
| ----------------------- | ------------------------- | ---------------------------------------- |
| Access to source code   | ✅ Yes                     | ❌ No                                     |
| Modify and redistribute | ✅ Yes (required with GPL) | ❌ No                                     |
| License philosophy      | Copyleft, anti-corporate  | Profit-driven, restrictive               |
| Protection mechanism    | Copyleft (e.g., GPL)      | Patents, EULAs                           |

---

## 🔄 3. Open Source as a Compromise

* Term coined in **1998**
* Companies wanted reusable, *but not necessarily shareable* code
* **Permissive licenses** (MIT, BSD) enabled commercialization
* Many giants (Apple, Amazon, Google) use open code in closed products

🧠 **Example**:
Apple’s macOS is based on **BSD**, an open-source OS — yet macOS itself is proprietary.

---

## 🧩 4. Where Are We Now?

Open-source code powers:

* Servers
* Internet infrastructure
* Blockchains
* Dev tools and SDKs

But most profit comes from:

* SaaS platforms (e.g., GitHub, AWS)
* Paid upgrades/extensions
* Consulting and enterprise services

---

## 🔍 Code Licensing in Numbers

| Category             | Approx. Share | Notes                                                                 |
| -------------------- | ------------- | --------------------------------------------------------------------- |
| **Proprietary**      | \~70–80%      | Closed source, licensed per user/device; typical in enterprise        |
| **Open Source**      | \~20–30%      | Publicly available, often under MIT/BSD/Apache                        |
| └── *Copyleft (GPL)* | \~10–12%      | Requires sharing of modifications; less common in commercial settings |

---

## 📚 Popular License Types Compared

| Category        | License      | Key Origin              | Philosophy                      | Must Disclose Code | Commercial Use | Used in Smart Contracts? |
| --------------- | ------------ | ----------------------- | ------------------------------- | ------------------ | -------------- | ------------------------ |
| **Permissive**  | MIT          | MIT, 1980s              | “Use it freely, just credit me” | ❌ No               | ✅ Yes          | ✅ Yes                    |
|                 | BSD-2-Clause | UC Berkeley, 1980s      | Similar to MIT                  | ❌ No               | ✅ Yes          | ✅ Yes                    |
| **Copyleft**    | GPL          | FSF, 1989               | “Freedom must be shared”        | ✅ Yes              | ⚠️ Yes\*       | ⚠️ Rarely                |
| **Proprietary** | Custom/EULA  | Microsoft, Oracle, etc. | Corporate control and profit    | ✅ Yes              | ✅ Yes          | ⚠️ Rare (private chains) |

---

## ⚙️ Smart Contracts and Licenses

| Feature                   | Description                                                                                |
| ------------------------- | ------------------------------------------------------------------------------------------ |
| **Source Transparency**   | Smart contracts are often fully visible; bytecode is always accessible                     |
| **SPDX Tag**              | Declares license at the top of the Solidity file (e.g., `// SPDX-License-Identifier: MIT`) |
| **GPL/Copyleft**          | Difficult to enforce in permissionless systems                                             |
| **Preferred Licenses**    | MIT and BSD (simple, commercial-friendly, flexible)                                        |
| **Proprietary Contracts** | Rare in Ethereum; mostly in enterprise/private chains                                      |

---

## 🏷️ SPDX: What Is It?

**SPDX** stands for **Software Package Data Exchange**, an open standard created by the Linux Foundation.

It helps:

* Standardize how license info is written
* Automate license audits in large codebases
* Avoid ambiguity and legal risks

**Example:**

```solidity
// SPDX-License-Identifier: MIT
```

---

## 📝 BSD-2-Clause — Explained Simply

You **can**:

* Use and modify the code
* Commercialize it
* Re-license it

You **must**:

* Keep the original copyright
* Not use the author’s name to promote your project without permission

**Plain-English summary:**

> “Do whatever you want with this — just don’t pretend I endorsed it, and keep my name in the credits.”

---

## 📝 MIT License — Explained Simply

You **can**:

* Use, copy, modify, and distribute the code
* Use it for commercial purposes

You **must**:

* Include the license text and copyright notice

You **don’t** have to:

* Disclose your changes
* Open your own source code
* Ask for permission

**Plain-English summary:**

> “Use this code however you like — just keep my name and don’t blame me if it breaks.”
