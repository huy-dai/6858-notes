# 6.858

# Midterm Review

- iOS security
    - 2020 Question 1
    - 2019 Question 12,13
    - 2018 Question 10
- Android Security
    - 2020 Question 2
    - 2018 Question 11
    - 2015 Question 18
- Google Infrastructure
    - 2020 Question 3
    - 2018 Question 1
    - 2017 Question 1
- U2F
    - 2020 Question 4, 5
    - 2019 Question 1
    - 2018 Question 2
- Baggy Bounds
    - 2020 Question 6, **7**
    - 2019 Question 5,6,7,8,9
    - 2018 Question 6,7
    - 2017 Question 6, **7**
    - 2014 Question 1
    - 2013 Question 2
    - 2011 Question 9,10,11,12,13
    - 2010 Section I
- OKWS
    - 2015 Question 6, 7
    - 2014 Question 5,6
    - 2010 Question 12,13
    - 2009 Question 8,9,10
- Symbolic Execution
    - 2015 Question 15,16,17
    - 2014 Question 9
    - 2020 Question 10
    - 2019 Question 2
- Lab 1 (Buffer Overflows)
    - 2020 Section VII
    - 2019 Section II
    - 2018 Section II
    - 2017 Section II
    - 2015 Section I
    - 2014 Question 3, 4
    - 2013 Section I
    - 2012 Section I
    - 2009 Section I
- Lab 2/OKWS
    - 2020 Section VIII
    - 2019 Section III
    - 2017 Section IV
    - 2015 Section IV
    - 2013 Section III

# Labs

[Lab 1](https://www.notion.so/Lab-1-cab4555c074f4e2fa073193f0b75f6bd)

# Lectures

### Lecture 2: Google Infrastructure Security

- What are the goals of a security architecture?
    - Protect user data
        - Confidentiality and Integrity
    - Maintain availability of services
    - Accountability
- What are possible threats that this paper is concerned about?
    - Insider Threats
    - Supply Chains
    - Physical attacks
    - Bugs
    - DoS
- What is the GFE?
    - It accepts incoming requests from the internet at large, and then translates it into an RPC call which routes to an internal service. Stands for Google Front End
- What components do they try to isolate?
    - VM’s
    - Linux users/containers
    - Language level sandboxes
    - Kernel sandboxes
    - Physical isolation
- What is a reference monitor and what steps does it take?
    - It takes a  request and sends it to a specific service
    1. Authenticate
        1. Determine who is making the request
    2. Authorize
        1. Is the person allowed to make the request?
    3. Audit
        1. Log the event so you can look at it later if needed
- What is the principal WRT to the reference monitor?
    
    It is the thing making the request. Could be a person, another service, or a computer
    
- What is the resource WRT to the reference monitor?
    
    The thing that is being requested. It could be a service, a cluster manager, or even just a single email message
    
- What are the steps to authenticate a person?
    - Use a password as well as 2FA
- What are the steps to authenticate a computer?
    
    Use a cryptographic key/signature which will authenticate the principal.
    
- How is authorization handled in GFE?
    
    **STORING BY ROW**
    
    Use a table which contains the Access Control Lists (ACL’s). Shows up in places like storage (who has access to a file) and RPC (who is authorized to talk to this service). It will take a particular resource (like a file), check who is allowed to access it, and see if you are allowed
    
    |  | Alice | Bob |
    | --- | --- | --- |
    | foo.txt | RW | — |
    | bar.txt | W | RW |
    
    **STORING BY COLUMN**
    
    Present a list of your permissions, like an End User Permission Ticket, which tells you what you are allowed to access. If someone steals your ticket, they can impersonate you and access things that you are allowed to access. Makes delegation really easy, because it doesn’t have to update an entire table. It can just delegate you a ticket. Great for short lived access control.
    
- How does Google prevent DDoS attacks?
    - Overprovision
    
    - Authenticate users before making requests
- How do you trust the servers you are running?
    
    Server’s have a security chip called the Titan Security Chip which hashes the BiOS and the OS. This hash is signed by the chip, which is validated by the cluster manager when that server is registered. The cluster manager has a list of all the valid Titan Security Chips so it can know if the server is valid.
    

### Lecture 3: User authentication

- What are the parts for authentication?
    1. Register → Add user
    2. Authenticate → Check request
    3. Recovery → Make sure you can recover the account if the credentials are lost
- Why is an intermediate principal a challenge?
    
    Because then you need to trust the intermediate principals during authentication to correctly relay the original user request
    
- How do you identify users?
    
    This is tricky. For example, for a bank you use a name and DOB. MIT you have something generated during admissions. This is important to be able to recover accounts.
    
- What are different methods of registering a user?
    - First come first serve: email@gmail
    - Bootstrap from other system: taking a username/registration from another service
    - Admin creation
- What are different methods for account recovery?
    - Security questions
    - Recovery email
    - IS&T
    - New account
    - Multiple ways to log in
- What are some common issues that can happen with passwords?
    - Common passwords can lead to guessing
    - Has to have enough entropy
    - Shared across systems
- How to store passwords?
    - Store a hash of the password so an attacker is not able to get the passwords if he gets access to the system. However, an attacker can precompute these hashes.
        - To remedy this, store a salt with the password.
- What is 2FA?
    - The computer stores the phone number. When a user authenticates, it generates a code, sends the code to the user, and then the user sends back the code to be verified.
- One Time Password?
    - The server has a seed which is hashed with the time. It then sends it to the user through SMS or something who then authenticates it with the server.
- What attackers are U2F worried about?
    - Web attackers?
        - Entities who control malicious websites. Users will enter credentials into attacker-controlled websites.
    - Related-Site Attackers?
        - Attackers will attack other sites in order to steal user credentials
    - Network attackers?
        - Attackers can observe and modify network traffic between the user and the legitimate site
    - Malware attackers?
        - Attackers can install and run arbitrary software on users computers
- What are the potential consequences of a successful attack?
    - Session Duplication?
        - Where an attacker can steal credentials that allow him/her to access user accounts
    - Session Riding
        - If an attacker can access/modify the user account only when the user is actively using the computer
    - 
- What two commands to a security key allow and what do they do?
    - Register:
        - Generate a asymetric key pair and return the public key. The server associates the key with this account
    - Authenticate
        - Tests for user presence, and uses it’s private key to provide a response. The server can then verify this response and authenticate the user.
- What happens during registration?
    - The server produces a challenge. The browser sends the server’s origin and a hash of the client data to the key. The key generates a key pair, associates that with the origin, then returns the public key along with a signature, key handle, attestation certificate. The web browser forwards this, and the website verifies the signature and associates the public key and key handle with the user account.
- What happens during Authentication?
    - The server sends a key handle and challenge to the browser. The browser sends the client data, a hash of the client data, and the key handle and origin to the security key. The security key verifies the key handle and the web origin. The key then verifies the user is present, and a counter. The browser passes the signature, the TUP (Test of User Presence), and the coutner value, which is then verified by the server.
- What is the device attestation certificate used during registration?
    - Each security key has an attestation certificate so that servers can disallow certain keys. For example, if a certain device has a flaw the server can choose not to accept it.
- What is the client data?
    - It binds the server-provided challenge to the connection with it’s server. This prevents MITM attacks.

### Lecture 4: Buffer Overflow Defenses: Baggy Bounds

- What is bounds checking?
    - Just checking if we are writing past a buffer. So this way we can’t overflow.
- What is a fat pointer?
    - Each pointer is now triple the size. It has the pointer, a base, and a limit. The base denotes the base of where the pointer should point to, and the limit is highest address it should point to.
    - malloc(): what’s it do?
        - set base=ptr, limit=ptr + size
    - *ptr: what’s it do?
        - base ≤ ptr ≤ limit
    - q = p+5:
        - ptr changes, but base and limit stay the same
- Where does baggy bounds store the metadata for pointers?
    - It stores a separate table which will store the base and the limit for each pointer.
- What size does baggy bounds allocate for each object?
    - They round up to the nearest power of 2 bytes.
- How does baggy bounds find the base of an object?
    - Because every object is a size of power of 2, if it stores the size of an object, it can round down to find the base.
- What is stored in the baggy bounds table?
    - For every address, store the size of the object that contains that address
- How does Baggy deal with OOB pointers?
    - It flips the highest bit in the address. The upper half of the address is reserved for the kernel, so dereferencing the pointer will crash, but you can still figure out where the pointer came from.
- How does baggy boudns determine if something was an overflow or an underflow?
    - If it’s out of bounds in the lower half of a slot, it’s an overflow. If it’s out of bounds in the upper half of a slot, it’s an underflow.
- What are slots, slot sizes, and how are they used?
    - Memory is partitioned with into slots with slot_size bytes. Every object will be assigned (size_of_object rounded up to nearest power of 2)/slot_size slots.
- When does baggy bounds crash?
    - When OOB pointers are DEREFERENCED. Or if a pointer is OOB by more than half a slot.

### Lecture 5: Privilege Separation (Redo Lab 2)

- How do services communicate to databases in OKWS?
    - There is a Database Proxy which handles all communications between services and databases
- How do HTTP requests get sent to services?
    - OKd handles all HTTP requests, and decides which services to send it to
- How are things logged?
    - oklogd communicates with everything and logs
- How are all these things started?
    - OKLD starts all the services and OKd
- What is the damage for each component being compromised?
    - Database Proxy?
        - Damage: All database data is compromised.
        - Attack Surface: Service sends request over RPC
    - OKLD
        - Damage: Root access!!!
        - Attack Surface: Almost None
    - Service
        - Damage: Whatever the dbproxy allows for
        - Attack Surface :HTTP req
    - OKD
        - Damage: Read all HTTP requests to services
        - Attack surface: Almost none - it’s 1 line of HTTP and forwards the rest
    - oklogd
        - Damage: not that bad, delete/view logs
        - Attack surface: all the services that talk to it

### Linux containers

- What is a container at a super high level?
    - A set or group of processes
- What is a Cgroup?
    - Isolates processes into slices, such as user-defined processes and kernel processes. This lets you define different permissions or resource usage limits to different groups
- What does seccomp-bpf do?
    - It limits the syscalls you can do
- What are namespaces?
    - Still not fully sure on this one

### Lecture 6: OS and VM Isolation (Read Paper). No previous quiz questions

- Why doesn’t Firecracker use normal containers?
    - Exploits in the linux kernel. There’s a large attack surface, 300+ system calls. There’s no isolation inside the linux kernel
- Why is VM security better than container security?
    - There is a virtual machine monitor that handles VM’s, and bugs in the VMM are way more rare.
- Why don’t they use VM’s?
    - It’s too slow. Can’t spin up a VM whenever a customer needs a new operation. Also too expensive.
- Why don’t they use QEMU?
    - QEMU has more functionality than they need, which is more potential attack surface. This way, they can spin up a little container/VM for each app, and then has a kernel with minimal functionality.
- What are the attributes of Firecracker?
    - What language is it written in?
        - Rust
    - How many LOC roughly?
        - 50k
- Why does firecracker boot so fast?
    
     Because it’s tiny
    

### Lecture 7: Software Isolation with WebAssembly

- What are the basic types in webassembly?
    - There are 4. Integers and Floats, 32 and 64 bit. 32 bit integers can be used as addresses
- Why don’t they use containers or others form of isolation?
    - need to run as root, overhead, also platform specific. They don’t know what architecture they’re running on.
- What is software fault isolation?
    - validation at runtime happens on the running machine
- How is memory stored?
    - memory is all linear. Each module can define a single memory which can grow. Memory is accessed with load/stores. Completely disjoint from code space, execution stack, and engine’s data structures
- Why can’t we use an interpretor?
    - It would be safe because everything would be encapsulated inside of the interpreter. It’s almost like a sandbox. But it’s too slow.
- Why can’t we use native x86 code?
    - Too hard to filter everything and secure.
- What do we need to ensure when it comes to security?
    - Sholdn’t be able to access anything outside of the module
    - Should enforce proper control flow. All basic blocks are validated, and one basic block must jump to another valid basic block.
- What can WebAssembly code access?
    - Stack values, Local variables, Global variables, Heap memory. We need to bounds-check each of these.
- How are global variable accesses safe?
    - There are a known number of global variables. Whenever we access a global, we reference it by an index, and we can check that the variable index we are referencing is less than the total number of globals.
- How are local variable accesses safe?
    - Each function has a fixed number of local variables. It’s kind of the same as local variables.
    - But how are they found? Global variables are constant, but local variables are stack offsets.
        - There’s a known stack depth at every WASM opcode regardless of how we got there. So whenever you are at a specific instruction, the types of all the stack variables are defined at that point predetermined. There’s some PL theorem for this
- How are heap memory accesses safe?
    - Whenever we write to memory, add a check to make sure that you aren’t writing past the memory size. Memory size is predetermined
- How is control flow handled?
    - No simple jumps, structured control flow. All branches define new constructs and scoping. Each block is almost like a function call, which is then returned at the end.
- How are function calls handled? How is control flow integrity enforced?
    - Direct calls
        - Functions all stored in a table. A direct call just takes an index which determines which function to call
    - Indirect calls
        - Function pointers are checked against the table of function. They ensure that the types of the functions that are supposed to be called match. Control flow integrity is enforced through these type checks.
- Store (?)
    - A record of module instances, tables, and memories. Basically everything for the program to run?
- **Kind of confused about the Execution section. What’s it mean and what’s it for? Do we need to know this?**
- 

### Lecture 8: Sandboxing Libraries with RLBox

- What is the challenge with apps using libraries?
    - App trusts results from libraries
    - The app might send sensitive data to the library
    - There is already an existing API for the library
    - Shared memory
- What can go wrong with sanitizing data?
    - An app might be written assuming a library behaves a certain way. But if the library is compromised, it might not behave that way. For example, it might have different error codes.
- What can go wrong with sharing pointers?
    - ASLR leaks
- What is a potential problem with checking values in a library?
    - Potential double fetch bugs. If you check twice, it might have changed in the library between checking.
- What is roughly RLBox’s solution?
    - Sandbox all the different libraries into their own sandboxes, and define ways to share data between them. Each library has their own address space
- WHen does RLBox perform the checks needed?
    - At compile-time
- How does the code distinguish between sensitive data?
    - By marking data as tainted with C++ templates whenever it is shared between libraries and application
- What is the issue with callbacks?
    - When the application is calling library code, they often provide the code with callbacks into the original application. We need to explicitly determine what callbacks are valid, and at what times. Also, what arguments should be allowed during a callback?
- Operations on tainted types
    - Int
        - You can directly convert to a tainted int, but need to validate it before you convert it back from a tainted int
    - char*
        - to convert to tainted, need to allocate memory in the library and copy it over. Same with converting it from tainted to untainted
    - Struct
        - To convert to tainted, you have to convert all of the fields. Same with converting back
- How do they deal with double fetch bugs?
    - With a freezable type. You can pass around freezable pointers in the library, but before you actually use it, you must call .freeze() first. This will copy the struct’s contents over to the application memory, so it can then be used without worrying about what the library does.
- How do they handle callbacks?
    - All callbacks must only accept tainted arguments and return tainted arguments. This means it has to validate the data before using.

### Lecture 9: Client Device Security (iOS)

- How is data handled on the flash chip?
    - It is all encrypted with AES, so whenever DRAM talks to it, it’s encrypted.
- What is the Enclave?
    - It is a separate address space from DRAM, and is responsible for several security tasks, such as encrypting the data on the Flash.
- How is the Boot ROM protected and what does it do??
    - It’s burned into the chip so it can’t be updated. It’s real trusted. It loads iBoot which then loads the OS Kernel, which then loads the apps.
    - It also contains the public key of Apple, which checks the signature of the iBoot code
    - This checks that everything comes from Apple. The Boot ROM definitely comes from Apple, and it verifies everything else.
- How does Apple prevent downgrade attacks?
    - The CPU is assigned an ECID which is unique for each phone. The iBoot signature is the signature of the iBoot code concatenated with the ECID. So now, each phone has a unique signature for each iBoot version, which needs to be verified. Apple only signs the latest versions.
- What are the roles of the Secure Enclave?
    - Manage crypto keys
    - Handles user authentication
        - Rate limiting authentication attempts
    - It’s outside of the OS, so even if the OS is compromised these roles are still working.
- What prevents replay attacks in the fingerprint reader?
    - The Touch chip has a key which is used to encrypt/validate the keys which is then sent to the Enclave.
- Where does the key that’s used for encrypting data?
    - There is a root key that is derived from the PIN as well as the UID that is in the Enclave. How’s it derived?
        - E_uid(E_uid(.....E_uid(PIN))): encrypt multiple times to slow down brute force
        - Because the key is derived from the PIN, it doesn’t exist anywhere on the phone when the phone is locked.
- What is the mechanism that allows you to change your PIN without destroying the encryption scheme?
    - Called Key Wrapping. Gives you many keys. How does it work?
        - Allows you to change keys and for background apps to run while the phone is locked.
        - Each file has it’s own key. Those keys are then encrypted with the root key. That way, you can change your PIN without having to decrypt too many things.
- How is the file system encrypted? (Look at this from the lecture video)
    - How are the keys for each file generated?
        - The root key encrypts a key for data protection, which is stored in the enclave. This then encrypts the file specific keys which is stored in the file metadata

### Lecture 10: Mobile App Security (Android)

- What is the primary goal of App security?
    - Sharing data between apps without giving up the security benefits of isolation
- How are apps separated?
    - Each app runs under it’s own UID, and has it’s own directory which is chroot’d. Apps are stored in APK’s
- How is application code verified?
    - Each application’s code is signed with a key from the developer
- Who does an app talk to if it wants to communicate with another app?
    - A reference monitor sits between. The reference monitor communicates with components in the receiving applications. An application can have several components that perform behavior based on what type of message it gets (DB, service, UI, etc.)
- What is a component?
    - A function that is triggered when an application receives data
- What are the parts of an “Intent” message?
    - Component: [com.android.phone/Dialer](http://com.android.phone/Dialer) (the app)
        - Optional
    - Action: MAIN, DIAL, SHARE
    - Data/type: Tel: 19789443686
- What are intent permissions?
    - Each application has intent permissions which say what it can do. Kind of like an entitlement in iOS. For example, an App might be able to DIAL, or list contacts.
    - There are sending intents. For example, an App might try to send a request to another app to dial a number. The receiving app would check that the sending app has the DIAL permission.
- What is in the application manifest file?
    - Needed permissions for the application to run (Dial, contacts, camera, etc.)
    - Components, and which components are public/private
    - Intent filters: which components can handle what functionality. For example, component B in this application can handle viewing PDF’s
    - Describe how the application shares data with other applications
- What are permission labels?
    - A specific permission, such as ADD FRIEND for facebook. They have types and descriptions.
    - Types
        - Normal: pretty benign ones
        - Dangerous: sending messages or something
        - Signature: Only apps from the same developer can ask for this permission. If an application wants to privilege separate several applications, the developer can define this and not the developer. The user can’t accidentally add permissions.
- When are the permissions decided/allowed?
    - At install
    - Refernce monitor prompts again at use
- Why is there a reference monitor?
    - It can help protect buggy apps
    - It helps enforce security with the intent permissions. For example, there could be multiple PDF viewers, and the reference monitor can decide which one is appropriate.
    - It gets the user approval and doesn’t have to rely on the application to do a popup
    - The reference monitor is what enforces all the permissions of the apps.
- Broadcast intents
    - Sometimes, you want to send a message to every application whose intent filter can take it. For example, you might have multiple text messaging apps, and so when you receive a message you want to send the message to all the apps.
    - Issue with this - you want to broadcast while making sure that only appropriate apps can receive it. So when you broadcast, you add a check saying that any receiving apps should have a specific permission.
- Dynamic intents
    - Sometimes, an application needs new permissions it didn’t know at install time. For example, if gmail requests to open a PDF in a PDF viewer, the application needs to be able to view gmail files. The Reference monitor uses what’s called a **pending intent**, which allows the sender to add an intent/permission in the form of a callback, which allows the receiver to call some function to get additional data.
- Off-device security
    - Google Play Protect audits various APK files
- What are some potential attacks?
    - Bugs
    - If a malicious app requests information from a user and a user agrees.

### Lecture 11: Bug finding

- What might count as a bug?
    - Crash, OOB memory accesses
- What are pros and cons of manual testing?
    - Pros
        - Precise specific bugs
        - Regression testing, you can check specific scenarios
    - Cons
        - Only a few inputs
        - Missing corner cases
        - High effort
- What are the pros and cons of fuzzing?
    - Pros
        - A lot of inputs
        - Can find corner cases
        - Can work on a blackbox system
    - Cons
        - High CPU usage
        - Weak coverage
    - What is generator-based fuzzing?
        - Defining a grammar and then making sure all inputs conform to that grammar. For example, fuzzing HTTP
    - What is coverage-guided fuzzing?
        - Have a corpus of inputs, and mutate them before passing them into the code you are fuzzing. Run it and see what the coverage is, and then modify it to increase coverage
- What is symbolic execution at a high level?
    - Finding inputs that cover different execution paths
- What are the high level ideas of EXE?
    - Use symbolic values
    - explore all possible paths
    - use SAT solver to decide what’s possible
- What is a symbolic value?
    - Just an alpha value that is going to be solved later rby Z3
- What is a path condition?
    - When control flow branches on a symbolic value, EXE queries STP on whether that branch is possible. That’s called a path condition.
- What are some potential issues with symbolic execution?
    - Branching get super expensive
    - SAT solver times out → unreachable

e