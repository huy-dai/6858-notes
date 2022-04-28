## RLboxing

Link: <https://css.csail.mit.edu/6.858/2022/readings/rlbox.pdf>

## Introduction

Firefox and many other major browsers rely on dozens of third-party libraries to render audio, video, images, and other content. However, these libraries are a frequent source of vulnerabilities, which, if exploited, can lead to a compromise of the browser or web service it is running.

Today, web browsers uses *coarse grain privilege separation* to limit the impact of vulnerabilities, typically through **renderers**, which is a separate portion of the browser that handles anything from untrusted user content to HTML parsing to JS execution to image decoding & rendering, in separate sand-boxed processes.

RLBoxing is a framework that makes it easier to change the browser to use sandboxing and have greater security between library & renderer (and avoid libraries from breaking out the sandbox) with minimal required changes to libraries.

RLBox uses type information to identify where security checks are needed, automatically insert dynamic checks when possible, and force compiler errors for any security checks that require user intervention. It also uses this info to better support secure and efficient sharing of data structure between renderer and library.

They uses two main isolation mechanics for library sandboxing: A software-based **fault isolation** from Google's Native Client and a multi-care process-based approach.

## Fine grain sandboxing

Sandboxing of the renderer is important because most sites requires parsing and rendering of cross-site media. Also many third-party libraries have vulnerabilities and that we want to contain from affecting rest of browser. 

Currently browsers like Firefox are just using one sandbox for all libraries, but this is not enough (your security of all libraries are only good as the weakest one)

## Threat Model

Assume web attacker that serves malicious but passive content from an origin they control that leads to code execution (e.g. via memory safety vulnerability). Side channels and transient execution attacks are out of scope.


## Design Goals

1. Automate security check: Doing static checks sand dynamic checks and sanitizations (e.g., pointer swizzling and bounds check)
2. No library changes
3. Simplify migration: Minimize the per-library effort of RLBox, and minimize changes to the Firefox renderer source

## Design

RLBox employs unique sandbox per `renderer`, `library`, `content-origin`, and `content-type`

RLBox uses a simple typing system of `tainted<T>`. All data originating from a sandbox begins with the tainted designation, and that can only be removed through an explicit validation process. 

RLBox mediates control flow by preventing branching on *tainted* values, and its API design restricts control transfers between renderer and sandbox. Data flows out of the sandboxes primarily though two interfaces: 

* `sandbox_involve()` - taints function return value
* `sandbox_callback()` - permits callbacks into the renderer from the sandbox, and statically forces the parameters of the callbacks to be tainted.

Data coming into the sandbox also needs to have the tainted type *or* ot be a simple numeric type. This means untainted pointers are not permitted.

Validation can either be done through *verify(verify_fn)* to validate simple tainted value types, which, for purpose of safety handling, is first copied to renderer memory [which is considered safe] rather than looked at directly in sandbox memory. Additionally, developers can also call *unsafeUnverified* to remove tainting without ajy checks. This mainly used for dev testing and is sometimes necessary for performance when doing things like passing buffer or decoded pixel data to Firefox renderer without copying it out of sandbox memory.

Validator functions primarily use to ensure application and also library invariants.

To ensure security against double fetches, RLbox() provides a freeze() method on tainted variable and struct fields to ensure that their original value when used, is not changed. This is done by copying value into renderer memory and making check at runtime.

In addition to these constructs, RLBox API also provides plugin to allow for two different low-level sandboxing mechanisms:

1. SFI (software-based fault isolation) using NaCL (Google's Native Client) - Uses inline dynamic checks to restrict memory address by a library to just a subset of the address space.
2. Process sandboxing - Restrict each library in a separate sandbox process whose access to system call is restricted using seccomp-bpf. 

**Drawbacks:** RLBox protections are only valid if the Firefox code that interfaces with the sandboxed library code is retrofitted to account for untrusted code running in the sandbox library, which is usually very hard to get right.

In general, RLBox cannot prevent developers from delibrately abusing unsafe C++ constructs like `reinterpret_cast` to circumvent the wrapper.

## Performance

Overhead time for sandbox creation is small (1-2 ms), and the overhead of RLBox dynamic checks is also small, approximately 20% for sandboxed libjpeg

They find the average peak renderer memory overhead to be 25% and 18% for SFI and process sandboxes, respectively, for image processing.

I didn't look into performance for the rest.