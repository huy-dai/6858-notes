# Client Device Security

During the creation of the iOS, we're seeing a shift from desktop/laptop to smartphones, which introduces a number of new security concerns and considerations. Today we'll focus on the device security, and on Thursday we'll discuss application security.

## Threat Model

* Threat: Stolen device, with the intentional of extracting data
  * Achieved through: guessing PIN, taking flash memory, installing buggy or malicious OS + app, exploit existing apps or kernel bugs, social engineer Apple for bypassing device security, taking advantage of unlocked phone, use fake fingerprint / face

## iOS hardware architecture 


We see that Apple's approach to device security involves a good deal of integration with hardware, which contrasts to many of the software-centric approaches we've seen so far.

At the hardware level, data goes through the following path:

`CPU --> DRAM --> AES Engine --> Flash`. 


Thus all data that gets written to the flash is automatically encrypted by the AES Engine. In addition we have a separate **Secure Enclave** coprocessor that is responsible for many of the secure operations for the device, including managing the AES engine. The Secure Enclave also contains the UID tag burned into it, which it uses as a secret key for key wrapping and secure long-term storage options. In this system the individual keys are never exposed to the CPU, only the results of the cryptographic operations after they are completed. It should be noted that the Secure Enclave stores its data in the DRAM, along with the other coprocessors, but it is encrypted carefully using their secret keys. Also the MMU also does work to make sure memory stored by these secure coprocessors and also the kernel isn't accessed or written to by OS + applications

In addition there are other coprocessors module like for the Touch ID and Face ID / Camera. Both of these modules also have a unique secret key burned into them in the manufacturing stage, which is also burned into the Secure Enclave (!). 
 - Question: Why are these keys secure? Can't we physically open the chip and read into them?
   - There are ways to design the coprocessors such that it is tamper resistant (e.g. changing capacitance by opening top plate destroys the data). Also this type of attack can be very difficult and meticulous, and at the end of the day it's not scalable since you only exploit per device.

## Design Focus

* Secure Boot
  * We start with a burned-in Boot ROM, which will load a bootloader called iBoot, which then loads the OS kernel. To make sure that we are loading the right components, we'll be using cryptographic signatures.
  * At the top-level, Apple has a public-secret key pair which it uses to sign all trusted softwares. Thus when it comes time to load the iBoot and OS kernel, the device will run code on Boot ROM to verify that the software to be loaded has a signature was indeed signed with Apple secret key. It does so using the Apple public key which is inherently burned into the chip e.g. `Verify(PK_Apple, target_software, signature) -> OK?`
  * Even though Boot ROM and iBoot are theoretically single point of attack, Apple has also put a lot of time into make sure these small code sources have been made securely. Also even if bugs are found there isn't a lot an attacker can do to affect them (they aren't taking in user input, for example).
  * To fight against **downgrade attacks**, which would still pass the signature check but have have serious bugs that an attacker will want to exploit, we need to add an addition check in our Verify function. It will look like `Verify(PK_Apple, iBoot || ECID, sig)`, which associates the action of upgrading (during software update) to this specific version of iBoot for this specific device (denoted by the ECID). Thus the device would send a request to Apple asking to upgrade to this version for this ECID. Apple would then check if this version is indeed the latest version and it then sign the result and send it back. The device can then verify the response as to be coming from Apple and use the result.
* Secure Enclave
  * Responsible for managing all of the cryptographic keys, and also to authenticate the user (whether that is done through passcode, Face ID, or Touch ID). For authentication the Secure Enclave also performs policy enforcing like rate-limiting and also checking if the sensors it is working with is trustworthy (ensure secure Sensor I/O through known share secret key)
  * By having all these important security functions done in the Secure Enclave we are able to separate ourselves from the main processor. Thus if the OS is exploited - which for many of the previous security systems would leave the system totally vulnerable - here the Secure Enclave is still unaffected.
  * When transmitting data between the Secure Enclave and Touch ID component, for example, they use encryption with the shared key. In practice there is a little more to the process than just a single symmetric key authentication, but intuitively this is the process. 
* Data Protection / Data encryption
  * All data is encrypted in storage on iOS. When it comes to data encryption, a crucial part is managing the keys used for encryption.
  * For iOS devices, the **root key** `K_PIN` is derived from two things: The passcode and the UID. They take the PIN and repeatedly encrypt it with the UID: `E_UID(E_UID(PIN))`. Enough iterations is done such that the total computation takes 80 milliseconds, which help reduce the speed of brute force attacks (Note for 4 digit passcodes it's still weak, but longer and complex passcodes are good).
  * Also, because the encryption key is partially derived from UID, it means that the flash chips are useless if moved to another device, even if the passcode is known.
  * This encryption key also go away if you restart or automatically after a period of time, so it's not stored for long in memory.
  * iOS also widely use **key wrapping**, which is a technique that allows you to work with multiple keys. Why do you need many keys?
    * Because in practice only using one key for encryption is not very flexible. If you change your passcode you will need to re-encrypt the entire drive, which takes time. Also for certain apps you may want for them to run in the background while the phone is locked.
    * Instead, you maintain a primary key `K_PIN`, and then for files encryption you use various keys `K_f1, K_f2, ...`. You then encrypt ("wrap") the individual file keys with `K_PIN`, and then store the wrapped version in storage. As long as `K_PIN` is safely maintained, you can always use to allow you to unlock everything, even if the actual file encryption key changes.
    * Note: There are also more levels to their design than just primary and child keys (like keys on classes of data).
  * Keys are stored on Effeacable Storage which are raw flash blocks (addressed directly).
    * The reason why it's addressed directly because we don't want multiple versions of the key stored at different parts of the storage, so that if we erase a key at that location it will be effectively forever lost. This is useful for Secure Wipes.
  * **Filesystem Key Breakdown**:
    * Filesystem specific key `KFS` - Global key used to encrypt all directories and file metadata
      * Needs to be a principal since you need to be able to unlock FS even before login
      * Metadata also stores the wrapped version of the individual file keys inside it
    * Individual-file keys `KF1, KF2`
      * Encrypted with key of the corresponding data class protecting the file
    * Data-protection class keys, of which there are four.
      * They are encrypted with `K_PIN`, stored in Effaceable Memory