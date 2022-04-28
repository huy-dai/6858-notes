# Android Security

## Context 

When Android was designed 15 years ago we were already looking at allowing installing arbitrary apps on mobile devices. Thus the developers of Android want to leave more flexibility for users to install broad range of applications on the phone while still maintaining security. For our class we're focusing the security of app-vs-app security controls, and preventing one specific app from getting unauthorized access to the filesystem and sensors.

The challenge will be to balance **isolation** with **sharing**.

## Models of App Isolation

1. **Desktop OS**

```text
| App1 | App2 |
|      OS     |
|   Storage   |
```
On the Desktop there isn't a lot of isolation between applications. Applications also operate and share the same OS and storage device with each other. In this case applications can easily interact with each other, and every app has a user's full privileges, so in the case of a malicious app it will affect the rest of the system.

2. **Strong Isolation**

On server applications like Google Drive, the applications are highly separated, where each is run under its own Virtual Machine, and their storage is separated. This is seen in things like VMs in AWS Lambda and WebAssembly. In addition, this model is not a great fit for an end-user device (for example, you might end up downloading photos/videos into one VM, but how do you get it to the other VM with Photoshop, and what about uploading to Youtube app?)

```text
| App1  |   |  App2 |
|  Qubes / VMM`     |
| Disk1 |   | Disk2 |
```

## Android Applications

Each Android Application is packaged within an .apk file with native code and also Java code. Each app process run with its own userID, data directory (e.g. /data/app1, /data/app2). Applications can't touch each other's directory directly. The **manifest** file of an application declares its permissions (things it is able to do within the process), along with APIs endpoints to be used by other applications. In addition, the entire application is signed by the developer, which helps devices identify the source of the app.

Each app is made of a number of **components**. In Android there are four types of **components**:
 1. Activity: UI component of app
 2. Service: background processing, RPC service
 3. Content provider: A database 
 4. Broadcast receiver: Gets broadcast announcements from other components

App-to-app communication is limited to **Android messaging**. Messaging is implemented by a specific Java component (e.g an RPC function) within each application. Thus there is no sharing of files. Isolation is implemented via traditional Linux kernel facilities (originally was just UNIX UID per application, but now incorporates SELinux and seccomp-bpf). SELinux in particular protects app files on SD card even when using FAT w/o ACLs (enforces an ACL on traditionally non-ACL fs).

Within the Android OS there is also a **Reference Monitor** that intercepts messages and decide where to send and on which port. If a message is intended for an application that hasn't been started yet, the Reference Monitor will start that application. The Reference Monitor will also enforce the policy of each app to decide if a message is allowed to go through or not.

## Message Sending - "Intent Messages"

### Intent 

Intent is basic messaging primitive in android. It has:
 * An (optional) targeted component, e.g. `com.android.phone/Dialer`
 * An action to be executed, e.g. one chosen from the set of `{ MAIN, DIAL, SHARE.. }`
 * Data / type, e.g. `tel:16172530005`

Often in Android there is the use of **implicit intent**, where applications can ask for a certain action to be done without having to know exactly who to address it to (so other apps don't know if you are using Skype or Zoom dialer, etc.). Java bindings are used for sending/ receiving intents. 

### Broadcast Intents

For some apps they want to be able to broadcast intents to all components with a specific permission. This is implemented by allowing a separate field within the intent for **receiving permissions**, which is specified by the sender of the apps which are allowed to see its messages. This functionality is useful for authenticator or 2FA apps. However, the required broadcast permissions are nested in app code and is not visible from Manifest page.


### Permission Labels

For the Reference Monitor to decide which intent it lets through, it will look at **permissions labels**. Permission labels, which are free-form string usually denoted as Java-style package names (ex. `com.android.phone.DIALPERM`), denotes individual privileges (e.g. actions). An application must define the full set of labels which it is authorized to use in the Manifest file. In addition, within each app, each of its component must have a single label that identifies it. Thus to reach that component we will need to use its label. Note: For content providers, the component is given two labels: one for read, one for write. 

### Permission Types

As mentioned, each permission label has a classifying type. In Android there are three:
* Normal
  * Use of permission is unlikely to be a security problem, and system don't sk the user about these permissions.
  * However, these labels are still listed because user can review them if they are interested, and it also establishes the idea of least-privilege.
  * Examples include: Changing wallpaper, change audio volume
* Dangerous
  * Dangerous permissions allows an app to do something potentially dangerous
  * System ask users about dangerous permissions
  * Examples include: Send SMS, access contact permission
* Signature
  * Only to be used apps signed by the same developer
  * This is useful if developers want to privilege-separate their own applications and don't want other applications to use their components except for themselves.
  * Also it help ensure that upgrades are performed only if they are signed by the same developer.
  * Example: Network configuration app

### Reference Monitor Security

The nice thing about the Reference Monitor is that instead of making developers implicitly worry about managing permissions or doing security checks on permission within their own application, through the Manifest + Reference Monitor we are making enforcement default. In addition, by explicitly spelling out the permissions we can also provide more transparency with the user (and also better allow for user choice) and also Google Play (which oversees and analyzes applications). Overall, this style of security is called **Mandatory Access Control (MAC)**, where the permissions are specified separately from code, thus making it easier to analyze without understanding the app code, and also to set up "default nothing" policies. MAC is in contrast to **Discretionary Access Control (DAC)** like those in Unix, where each UNIX app can set its own permissions on files, and can be changed by the app over time (thus it's not easier for us to tell directly how the film perms are managed).

### Manifest

The **Manifest** is written by the developer, and is allowed to ask for any permissions (!). Each permission labels an app provides needs to have a type classification and also a description of what it is providing (this is useful when displaying to the user). As mentioned, Manifest must also declare a label to protect each component. An app cannot change its manifest without reinstallation or re-asking user.

Example of a Manifest:
* Needed performs: DIALPERM, CONTACTS
* Components:
  * A -> Private
  * B -> Public, DIALPERM
  * C -> Public, CONTACTS
* Intent Filters: B can handle VIEW/pdf 

User approvals for permissions are done:
1. At install
2. Through user prompts made by Reference Monitor

### Temporary Permissions

In some cases you may have a program like Gmail which downloads a pdf from an email, and it sends an intent to the PDF viewer on the system to view that file. However, in this case rather than sending a copy of the file to the PDF viewer over Intent, the PDF viewer will need to be granted temporary access to the file on the Gmail process' storage. 

We can do this using **URI delegation**, e.g. setting up an URI for a resource and specifying which app is allowed to view/edit/modify it, or with **pending intents**, which allows for implementation of callbacks into the sender application (also useful for things like event notification). Both of these are registered to and tracked by the Reference Monitor.

## Off Device Security

Google Play Protect is a service by Google which checks, audits, and verify APK files. That way certain applications can get a check marked (approval) that designates trusted apps. By default on Android phones it will only install Google Play Store apps, but this setting can be changed.


## Overall Thoughts

In terms of evaluation security of the Android platform, while kernel and setuid-root bugs do occur, they are much less common and hard to find. However, most of the time the security issues lies with user installing malware with dangerous permissions. For example, actual common malware have occurred where it send SMS messages to premium numbers, and attackers directly get money by deploying such malware. Constantly asking for permissions will over time desensitize users to permission asks, which can dangerous. Overall though, it is reasonable to download and run unknown apps on Android, unlike on desktop.