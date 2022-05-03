# Day 1

A system is considered 'secure' if it works despites adversary.

Goal: Make sure that only TAs can access grades.txt

* Positive Goal Check: TAs check if they can access it
* Negative Cons: bugs; bribe TA; steal credentials; steal laptop; sniff wireless network

### Systematic Plan

Thus, we need some **systematic plan** to work with security systems. For example,

- Plan
    * Goal: Only Alice can access F.
    * Threat model: Set of assumptions that you are making about who you are up against (who wants to foil your security system)
    * Policy: Define a plan / set of configurations you are going to follow to meet goal.
    * Mechanism: Everything that implements the policy, e.g. filesystem with access control, logging system, 2FA 

The policy and mechanism is working to achieve the goal whilst fighting against the threat model.

Observation: Security is _hard_ to get right. It's hard to know what to even look for from your threat model or what the best way is to implement it.

Thus you need to __iterate__:

- Monitor your system for attacks
- Build on well-understood components from other people / solutions (e.g. libraries, design principles)
- Learn from your post-mortems -> Why did the breach happened, how can we learn from it.

As a defender, you have a harder job than attacker. Defenders need to guard against all attacks, while attacker only need to find one successful attack.

That said, we still want to make:

* Attack cost > value
* And convince attackers to want to attack other systems

We should also sought to use techniques that have proven to payoff for building secure systems. For example, you want to enable security features like VPN, Sandboxing


## Policy

When designing your policy or making a policy changes, you have to consider all of the different scenarios that can occur and decide how you will handle them accordingly.

Ex. Flight company allows business class members to change their tickets at any time. This opened for a loophole for where a member can change their ticket to another flight location once they've boarded, which effectively allows them to have infinite amounts of flights.

- Fix requires for boarding systems to be connected to the reservation system / network so that they update the system to disallow ticket change once customer has boarded.

- Indicates a flawed policy of allowing ticket change at **any** time.

The place where most policy problems happen in _corner cases_ related to administration and maintainance. 

  - Ex. What happens if you change a password, who has access to backups, who gets to configure the systems, who controls the audit logs, and who gets to update

Considering security of interconnected components: Making things explict, communicating to each other about your own policies, etc.

## Insecure Defaults

Many routers have default credentials that are often unchanged by the users, or permissions in Amazon S3 are not properly set. 

## Threat Models

* DON'T:
  * Use security by Obsecurity: Don't assume or depend on secret designs continuing to be secret. Many designs can be reverse-engineered or be leaked over time. Also, once breached the damage is severe (you no longer have any security)
  * Depend on users: Users are vulnerable to phishing attacks - it's actually how many attackers are getting through 2FA
  * Depend on assumptions about how attackers will attack you - For captchas, some malicious entities have gotten through by hiring workers in 3rd-world nations to solve it for them.

* NEED TO CONSIDER:
  * Where your binaries come from: Attackers not need to exploit system but simply attack the codebase (where source code is stored) to inject malware.
    * Updates may actually introduce threats if the compiled software is not what's expected


## Bugs

One exists in average 1,000 lines of code on average. Bugs in any component can often lead to exploits.


## What are the best practices to secure design?

* Be explicity: Clarify weaknesses
* Simple and general design
* Defense in depth: Consider many threat models and use multiple layers to defend against them
