---
title: "Why bcrypt Can Be Unsafe for Password Hashing ?"
date: 2025-11-04T11:18:39+01:00
author: "Aymane BOUMAAZA"
draft: false
description: ""
tags:
 - TIL
 - cryptography
 - bcrypt
 - security
 - password-hashing
---

> **TL;DR**: bcrypt ignores any bytes after the first 72 bytes, this is due to bcrypt being based on the Blowfish cipher which has this limitation.

[bcrypt](https://en.wikipedia.org/wiki/Bcrypt) has been a commonly used password hashing algorithm for decades, it's slow by design, includes built-in salting, and has protected countless systems from brute-force attacks.

But despite its solid reputation, it also has a few hidden limitations worth knowing about.

Let's take a look at this code:
```python
import bcrypt

password_1 = b"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa1"
password_2 = b"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa2"
hashed_password1 = bcrypt.hashpw(password_1, bcrypt.gensalt())
if bcrypt.checkpw(password_2, hashed_password1):
    print("Good password")
else:
    print("Bad password")
```
The code takes a string (as bytes) starting with 72 a's and **ending with 1**, hashes it using `bcrypt.hashpw`, and then checks the same string **ending with 2 this time** against the hashed password.

The output should be *`Bad password`*, right ?

Let's run the code, and see the output:
{{< figure src="/img/bcrypt_old_version.png" align="center" alt="bcrypt.checkpw returns True" caption="bcrypt.checkpw returns True" border="#f8f4f0" width="100%" >}}

The code shows *`Good password`*, but why ????

Turns out that bcrypt is [based on the Blowfish cipher](https://www.openbsd.org/papers/bcrypt-paper.pdf), which only encrypts the first 72 bytes, this limitation is therefore inherited by bcrypt.

> The [Blowfish algorithm](https://en.wikipedia.org/wiki/Blowfish_(cipher)) uses an 18-element [P-box](https://en.wikipedia.org/wiki/Permutation_box), where each element is a 32-bit (4-byte) subkey. Therefore, the total P-box size is 18 * 4 bytes (72 bytes).

This means that if the password is longer than 72 bytes, the bcrypt algorithm will **only encrypt the first 72 bytes**, and the rest will be **ignored**.

> bcrypt's 72-byte limit applies to **bytes**, not **characters**. This means that passwords with non-ASCII characters (like emojis or accented letters) may reach the limit even sooner, since UTF-8 encoding can use more than 1 byte per character.

If you don't want to worry about this limitation, you have several alternatives:

- Use a different algorithm, like [Argon2](https://en.wikipedia.org/wiki/Argon2), which does not have this limitation (Awarded the [Password Hashing Competition](https://password-hashing.net/) in 2015).

- Hash the password into a digest -**less than 72 bytes**- first (SHA-256, SHA-512, etc.), and then hash the digest using bcrypt. See the following example:

{{< figure src="/img/bcrypt_sha512.png" align="center" alt="Hash with SHA-512 before bcrypt" caption="Hash with SHA-512 before bcrypt" border="#f8f4f0" width="100%" >}}

- You can always enforce the password length to be less than or equal to 72 bytes by yourself, but it's not recommended to do so.

However, [since version 5.0.0](https://github.com/pyca/bcrypt/blob/443366fcdd06c56c825ba7ac58cb8d7630cea1b2/CHANGELOG.rst?plain=1#L10C1-L12C62), Python's `bcrypt` package began raising errors when hashing passwords longer than 72 bytes, this commit introduces the change: [Raise `ValueError` if password is longer than 72 bytes](https://github.com/pyca/bcrypt/commit/d50ab05b2bece07d5c8d6a4179064fc714fd9126)

{{< figure src="/img/bcrypt_v5_error.png" align="center" alt="bcrypt.hashpw raises an error since version 5.0.0" caption="bcrypt.hashpw raises an error since version 5.0.0" border="#f8f4f0" width="100%" >}}

This limitation is handled in different ways in other languages and libraries.

- [Go raises an error](https://pkg.go.dev/golang.org/x/crypto/bcrypt#GenerateFromPassword) when the password is longer than 72 bytes
- OpenBSD's bcrypt implementation [truncates the password](https://github.com/openbsd/src/blob/be88feada38d370546228da28b63a078a676d0ca/lib/libc/crypt/bcrypt.c#L127) if it's longer than 72 bytes
- [Rust's bcrypt](https://docs.rs/bcrypt/latest/bcrypt/#functions) truncates the password by default, but offers `non_truncating` methods to raise `BcryptError::Truncation` error if the password is longer than 72 bytes.
- Spring Security's base class `BCrypt` offers the method `hashpw` with `for_check` flag (weird name, right ?) to [raise `IllegalArgumentException`](https://github.com/spring-projects/spring-security/blob/9dde69746f7512f3b6df3641afa10b71db51ad4c/crypto/src/main/java/org/springframework/security/crypto/bcrypt/BCrypt.java#L616) if `for_check = false`, while I havent found a way to pass a similar flag to the `BCryptPasswordEncoder` class.


To summarize, bcrypt is still fine for typical passwords **<72 bytes**, but consider other options for future-proof security.

---

I discovered this limitation in [@devhammed](https://x.com/devhammed)'s [tweet](https://x.com/devhammed/status/1985284158390739204), thanks to him for sharing this information.
