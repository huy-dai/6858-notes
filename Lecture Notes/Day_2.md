## Day 2 Lectures Notes

The aims of security architecture is to:

- Prevent known attacks
- Prevent against yet / unknown attacks
- Limit the potential damage from attacks

To do that, we need to consider the goals that you want to achieve with your architecture and also the threat landscape.

Concepts to consider:
* Trust
* Isolation
* Authentication,
* Secure Channels,
* etc.

### Case Study: Google (Cloud)

**Goal:** 
  - Protect user data
  - Providing high availability
  - Ensure integrity and confidentiality of the data
  - Establish accountability

**Threats**: 
  - Insider
  - Supply Chain attacks
  - Physical attacks
  - Finding bugs
  - Malicious apps
  - DoS attack

What does the Google environment look like?
  * Consist of a number of servers hosting different services.
  * GFE takes in RPC requests and forward them to proper machine
  * Handles various protocols like HTTP, SMTP, etc.

**Isolation**:

They provide *isolation* through: VMs, Linx users / containers, language-level sandbox (running code for "unsafe" languages like JS and WASM), kernel sandboxes, physical [strongest level of isolation]. These work together to have high assurance, increase cost effectiveness + performance (combine strength of many machines), and allow for high compatibility [though some are better at this than others].

Some components like GFE help fight against attackers getting in (they can't directly attack or read memory of the machines), and in other instances like language-level sandboxes help prevent attackers from getting out.

**Sharing - "Reference monitor"**

Google implements __guards__ (to filter users-direct requests or inter-service requests) that check whether a request is allowed before it allows it to interface with the resource. There is a 3 part process:

1. Authentication - Verify the identify of the user (who did the request came from?)
2. Authorize - Check if the user has the permission to access the resource
3. Audit - Log of security decisions that was made. Used to help you recover in the case of a problem and also for post-mortem investigations

* Discretionary Access Control: Allow each users to set own access permissions
* Mandatory Access Control: Permissions is enforced downwards from administrators

Principals involved: People, service, computer

Resources: Services, email message, cluster manager

**Triple A - More Detailed**

* Authentication
  - User + password combination checked against table
  - 2FA 
  - Public/private key authentication 
    * Keys should be rotated periodically (help fight against data breaches when keys are potentially compromised)
    * In some cases services may generate new keys every it starts up
* Authorization
  - Principle is anything which you want to define an access policy for [ex. Nikolai on his laptop, Nikolai on his phone, Jessica vs Nickolai, etc.] 
  - Imagine a matrix mapping f(principle, resource) to Access Control List for various objects (e.g. things you want to access and their corresponding permissions)
    - When it comes to storing ACLs by files, it becomes a very task to figure out "Who has access to this file?"
  - In addition, another way to do permission is storing **capabilities**, which provides a more general way of access. For example, when working with end-user permission tickets, which are generally shortlived, we can hand them over to services we trust and _delegate_ them to act on our behalf.

ACL vs Capability List:

ACL associates **files** to list of users and permissions that thoses users have to the files. This "row" view is good for long-term storage and easy revoking of access to specific files.

Capability list on the other hand associates **users** with a list of files and permissions they have with those files. These are better for short-term usage when working with induvidual users.

**DoS discussion:**

Challenge: Distinguishing between real and attack requests.

Google's method: Establish a GFE entity. Upon an attack it can handle it with:
  1. Overprovision: Allocate more bandwidth. Uses shear bandwidth so that attackers request won't make a difference
  2. Authenticate: Force users to log in as soon as possible (make it easier to distinguish between potential attackers and users)

**Deploying a new server:**

How do we onboard a new server? The new server will need to *register* with the Cluster Manager. One way it can do that is to use a customized chip (called "Titan security chip") that is integrated as part of the boot process for server. Upon startup, the security chip calculates hash of BIOS, hash of OS, and packages them with the Titan chip key and signed it and send it over to the Cluster Manager. 

Google, as the manufacturer of the security chip, will have a table of trusted security chip keys along with their allowed / associated BIOS and OS. This help ensures that i) servers are trustworthy, and ii) servers are running trustworhty BIOS or OS.

Also, when it comes to running binaries, binaries are actually carefully built on separate, trusted hardware with full backtracking of who, where, when it came from, and then they are shipped to the actual servers that are supposed to run these binaries.
