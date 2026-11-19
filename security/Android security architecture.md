# 1. Explain Android's Security Architecture and Its Main Layers

Think of **Android security as a building with five locked doors, one inside the other**.

A thief has to open every door to reach the treasure. Android follows the same **defense-in-depth** approach: if one security layer is compromised, the remaining layers continue protecting the device and user data.

![Android Security Layers](https://github.com/user-attachments/assets/18bc5ac3-7f53-428f-be60-9a6a4ad2e674)

## Quick Overview

| Layer | Main job | Key technologies |
|---|---|---|
| 1. Application | Keep apps apart and control what they can access | UID sandbox, permissions, app signing |
| 2. Framework and services | Control access to system features and hardware | Binder, permission checks, Treble HALs, Play Protect |
| 3. Linux kernel | Enforce the rules, even against a compromised process | SELinux, seccomp, kernel hardening |
| 4. Boot and encryption | Make sure the system is untampered and data is unreadable at rest | Verified Boot, file-based encryption |
| 5. Secure hardware | Protect keys so they cannot be extracted | TEE, KeyMint, StrongBox |

---

## Layer 1: Application Sandbox and Permissions

Every Android app lives in its own **sandbox**, similar to having its own locked room.

Android provides each application with:

- A unique **Linux User ID (UID)**, enforced by the kernel
- Private application storage
- Its own **SELinux domain** (a label that limits what the app process can do)
- A **seccomp** filter that blocks system calls apps should not need
- **Scoped storage**, so apps see only their own files and shared media they are allowed to use, not the whole filesystem

By default, one application cannot directly access another application's private data.

### Permissions

If an application needs access to sensitive resources such as the camera, location, contacts, or microphone, it must request a **runtime permission** from the user. Android also supports:

- "Only this time" and "while using the app" choices
- Auto-reset of permissions for apps that go unused
- Privacy indicators (camera and microphone dots) and a Privacy Dashboard

### Application signing

Every APK must be **digitally signed**. The signature lets Android confirm who published the app and that it has not been modified. Modern signature schemes (v2 and later) cover the entire APK, and an update is accepted only if it is signed with the same key as the installed version.

> **Key idea:** Even if a malicious application is installed, the sandbox limits what it can access.

---

## Layer 2: Android Framework and System Services

This is the layer applications communicate with to access system functionality.

It includes components and services such as:

- Activity Manager
- Package Manager (verifies app signatures at install time)
- Permission checks
- **Binder IPC**, the mechanism apps use to talk to system services
- Keystore service
- Media and hardware services

Applications do not touch most hardware directly. They call framework APIs, and the system services check permissions before acting.

### Isolation of hardware code

Under **Project Treble**, hardware drivers sit behind **Hardware Abstraction Layers (HALs)**, and many HALs run in their own restricted processes. A bug in a camera or media component is therefore contained instead of giving an attacker the whole system. Android also sandboxes media codecs and increasingly writes security-critical code in memory-safe languages such as Rust.

### Ecosystem protections

These are provided by Google's services rather than the core OS, so they depend on the device having Google Mobile Services:

- **Google Play Protect** scans apps on the device and in the cloud, including apps installed from outside the Play Store, and can warn about or remove harmful ones.
- **Play Integrity API** lets apps check that they are running on a genuine, unmodified device.
- **Project Mainline** lets Google update core system components (such as media and networking modules) through Google Play without a full OS update.

> **Key idea:** Applications go through controlled Android framework APIs and system services, and each risky component is kept in its own restricted process.

---

## Layer 3: Linux Kernel Security

The **Linux kernel** forms the core security foundation of Android.

It provides:

- Process isolation and UID-based permissions
- Memory protection
- Networking security
- Device driver security
- Hardening features such as ASLR and control-flow integrity, which make exploits harder to write

Two important mechanisms at this layer:

### SELinux

Android uses **Security-Enhanced Linux (SELinux)** with **Mandatory Access Control (MAC)**, enforced for all processes.

SELinux policy defines:

> Which process can access which resource, and what operation it is allowed to perform.

These restrictions apply even to privileged processes, including root.

### seccomp

**seccomp-BPF (Secure Computing)** restricts the system calls a process is allowed to make. If a process does not need a dangerous system call, Android blocks it, which shrinks the kernel's attack surface.

> **Key idea:** Even if an attacker compromises a process, SELinux and seccomp can significantly restrict what that process can do next.

---

## Layer 4: Verified Boot and Data Encryption

Android protects both the **boot process** and **stored user data**.

### Android Verified Boot (AVB)

When the device starts, each stage verifies the integrity of the next stage (bootloader, kernel, system partitions) before allowing it to run. If something has been tampered with, the device warns the user or refuses to boot. **Rollback protection** also stops an attacker from installing an older, vulnerable version of the system.

### File-Based Encryption (FBE)

Android encrypts user data on the device using **File-Based Encryption**. Different files can use different keys, which allows two kinds of storage:

- **Credential Encrypted (CE)**: available only after the user unlocks the device for the first time
- **Device Encrypted (DE)**: available right after boot, so essential features such as alarms and incoming calls can work before unlock

**Metadata encryption** additionally protects filesystem information. The encryption keys are tied to the user's lock screen credential and protected by secure hardware (see Layer 5).

> **Key idea:** If someone physically steals the device, the stored data stays unreadable without the user's credentials and the device's keys.

---

## Layer 5: Secure Hardware

The deepest security layer is hardware-backed protection.

Key terms and how they relate:

- **Android Keystore** is the API apps use to create and use cryptographic keys.
- **KeyMint** (the successor to Keymaster) is the secure component that actually generates keys and performs operations with them.
- **Trusted Execution Environment (TEE)** is an isolated secure area on the main processor where KeyMint can run.
- **StrongBox** is a KeyMint implementation running on a separate, dedicated secure chip (for example, Google's Titan M on Pixel phones). It offers stronger tamper resistance than a TEE.
- **Biometric authentication** (fingerprint and face) is also verified inside secure hardware. Only "Class 3 / strong" biometrics can unlock keys.

An application asks the secure environment to perform an operation, such as:

- Encrypt or decrypt data
- Create a digital signature
- Verify a signature

The key material is designed to **never leave** the secure environment. Apps can also use **key attestation** to prove that a key really lives in hardware.

> **Key idea:** The application can **use the cryptographic key without ever seeing the key material**.

---

## Protections That Cut Across the Layers

- **Network security:** Apps targeting Android 9 (API 28) or higher block cleartext (non-HTTPS) traffic by default. Developers can configure exceptions and certificate pinning through the Network Security Config. Modern protocols such as TLS 1.3 and Private DNS (DNS over TLS) are supported.
- **Regular security updates:** Monthly security patches fix known vulnerabilities, and Mainline delivers fixes for core modules faster.
- **Memory-safety defenses:** Rust for new low-level code and hardware-assisted Memory Tagging Extension (MTE) on supported chips reduce an entire class of exploits.

---

# Why Do These Security Layers Matter?

Consider a malicious application installed on an Android device.

### Step 1 - App Sandbox and Permissions

The malicious application is initially restricted by the **application sandbox**. It cannot freely access another application's private data, and it needs user approval for sensitive resources.

↓

### Step 2 - Framework Security

Access to sensitive system functionality must go through controlled **Android framework APIs and permission checks**.

↓

### Step 3 - Kernel Security

If an attacker manages to escape the application sandbox, **SELinux** and **seccomp** still restrict what the compromised process is allowed to do.

↓

### Step 4 - Encryption

Even if the attacker obtains access to stored files, the data is still protected by **file-based encryption**. Verified Boot also makes it hard to plant a persistent modification in the system.

↓

### Step 5 - Secure Hardware

The cryptographic keys protecting the data are held inside a **TEE or StrongBox**. The attacker cannot simply extract them from application memory or storage.

---

## Interview Summary

Android follows a **defense-in-depth security architecture**.

The main security layers are:

1. **Application Sandbox and Permissions**
2. **Android Framework and System Services**
3. **Linux Kernel, SELinux, and seccomp**
4. **Verified Boot and File-Based Encryption**
5. **Hardware-Backed Security: TEE, KeyMint, and StrongBox**

A one-line way to remember it: **isolate, verify, encrypt, and keep the keys in hardware.**

The main principle is:

> **No single vulnerability should be enough to compromise the entire Android device.**

If one security layer fails, the remaining layers continue limiting the attacker's capabilities.
