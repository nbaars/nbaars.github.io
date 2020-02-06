---
layout: post
title: Using Libsodium in your Java project
categories: [Java, Security]
---

Sodium is a modern, easy-to-use software library for encryption, decryption, signatures, password hashing and more.
It is a portable, cross-compilable, installable, packageable fork of NaCl, with a compatible API, and an extended API to improve usability even further.
Its goal is to provide all of the core operations needed to build higher-level cryptographic tools.

That is the introduction given on the web page of [Libsodium](https://download.libsodium.org/doc/) let's look
at how we can integrate this into your Java application.

Libsodium is a written in C and is cross compiled to different platforms, however in Java we need to have 
a JNI wrapper which actually does a call to the low level Libsodium code. 




