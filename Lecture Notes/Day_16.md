# Information Security in real life

Guest Lecture from Max Burkhardt. His website: <https://maxb.fm/>

In terms of experience Max did pentesting -> application security at AirBnB -> Figma (security in-general). His presentation is revolved around motivating us to go into security.

## Why do cybersecurity need more people

In the last 20 years the number of applications has scaled exponentially, but not so much so in terms of security. This is because modern information systems are built on growth, and if security opposes growth, then it won't happen. Google has one of the best security teams out there, but they are still paying tens of thousands in XSS bugs (they paid out $1.2 million in XSS bug bounties in 2016!).

Current, there's the a hugh opportunity to change how out the industry does security (security work is always at the bleeding edge!). Also security is hugely creative, and you're constantly jumping between different paradigms and technologies. Each organization has its wn tech stacks, threat models, budgets, organization cultures, all of which impacts its security standpoint.

## Definition of Security

Security is a strategy to address risks to your system. To do that you should:

1. Mitigate risks
 -  To mitigate risks, you should:
    -  Build protection close to your assets
    -  Work with the assumption that some defenses *wil* fail. Thus be prepared when those defenses fail, and be ready to respond when that occurs
    -  Self-assess constantly!
 -  Look at common security threats that show up in breaches and orient your approach towards them. 
    -  Examples include ransomware (Colonial Pipeline), compromised insiders (LAPSUS$), phishing (basically everyone).
2. Think from the attacker's perspective
 -  Attackers will most likely start with the basics, e.g. perform phishing attacks (but can be difficult with 2FA).
 -  Deploy watering hole attack which affects a piece of software that many employees in the company use. For example, get an employee to install a malicious Chrome extension
 -  If this fails, perhaps your previous more generic attacks were detected by in-house security tools, then you would move on to designing *custom malware* (no signatures to detect) with security countermeasures (like terminating security software and using advanced exfiltration techniques)
3. Build security defenses that scale and are more centralized / streamlined
4. Incorporate technologies such as sandboxing, 2FA / passwordless authentication, WebAuthnn, etc.

**However, you must avoid putting unreasonable constraints on your employees**. This will mostly lead to employees finding ways around your security systems. Also if you are hurting product efficiency the company will most likely not accept these security changes. 

## Careers in Security

1. Offensive side
   1. Pentesting, security research, red team
   2. These allow you to build a diverse skill set, and it's super satisfying when an exploit land
2. Defensive side
   1. Blue team, security product engineering
   2. A large part of this involves being a **software engineer** or thinking like a SWE. For example, you may need to develop:
      1. Security toolkits / libraries to be incorporated in company's product
      2. Frameworks to eliminate common bugs
      3. Systems to analyze activity for malicious indicators
   3. Other roles includes intrusion detection
      1. This involves being able to extract signals from systems, identify events, and build a process that can respond to security threats with speed.