# Privilege Separation

With any program, it's very likely that they will have bugs, some of which may lead to security vulnerabilities. Thus the question becomes "how do we build secure systems given these bugs?" The paper we looked at encapsulates a case study of `OKcupid.com` running OKWS.

## Ideal: Prevent bugs

No bugs = no vulnerabilities. However, this approach requires us to understand what bugs are out there and also for us to write out classes of bugs that we are looking for (can't just say "I want to find bugs"). In design phase we also need a **prevention plan** to decrease the chances of bugs existing.

## Threat model

Assume that we are working with a web server which serves a variety of apps. It takes in login, msg, profile, among other requests. The App requires interfacing with a database (accessed with SQL-like queries) and local files (accessed with syscalls). Stored in the database includes info about passwords, messages sent, profile info, and more.

Common classes of web bugs:

* SQL injection
  * Consider the phrase "SELECT * FROM .. WHERE id=" + friend_id;
  * If input not properly sanitized, the user can provide something like "5; DELETE FROM profile" to drop the entire profile table
* Buffer overflow
* Access control bugs
* Denial-of-service bugs
  * User has some way to get the server to run out of memory or disk space or forces the server to crash so often it's unable to recover
* File access bugs
  * E.g. open("/profile/"+path_id) where path_id is provided like "../../etc/passwd".

## Possible mitigation

* Process separation
  * Split up functionality into logically separated processes. 
  * That way, we are limiting the potentially damage that one gets with exploiting one service [**limit damage**].
  * In addition, through this method we could better hide some of the more sensitive components behind other components and only expose the more limited processes (e.g. those tasked at handling / parsing URLs) [**limit attack surface**].
  * _Question_: How do we actually split it up?
    * Service / type
    * Privilege / resource
    * User
    * Buggy
    * Exposure
  * _Implementation_: How to actually isolate?
    * Userid, chroots (jail directory)
    * Limit sharing between processes
    * **Challenge**: Need for isolation to work well (from a security perspective) while maintaining good performance

## Case Study: OKWS

OKWS runs separate service processes, which communicate to a database proxy that then directly interfaces with the database. Above the services is the OKd which takes in user HTTP request and dispatches them correspondingly. oklogd is responsible for taking in logs from other services in batches and write them to log files. At the highest level is the okld daemon which initializes the entire system and run / restart services as needed.

One important thing to note is that OKWS was designed in early 2000s, so the hardware they were working with at the time is that there would physically be one machine responsible for the database, another for the HTTP parsing, etc.

**Why this split?** 

Their goal is to be able to **quickly iterate** on their services, which is mostly done by developers who are not security experts. In that case they figure that most of the bugs will be _coming from the services_ themselves. The DB proxy help limits the data access that each service can do, which prevents from all the data being leaked in the event of a service compromise.

**What is the damage from a compromise?**

| Component | Damage                                                               | Attack Surface                                                                                           |
|-----------|----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| db_proxy  | all DB state                                                         | svc sends malicious RPC request to proxy                                                                 |
| okld      | runs as root -> control all other services & config files & DB state | almost no input except for interface getting info about process crash, leaking data about okld from fork |
| svc       | what dbproxy allows for that service                                 | http request, bad data planted from db to svc                                                            |