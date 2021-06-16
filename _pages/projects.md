---
layout: page
title: Projects
permalink: /projects/
---

Overview of the projects I'm currently working on or actively maintain.

## WebGoat

WebGoat is a deliberately insecure web application maintained by OWASP designed to teach web application security lessons.
This program is a demonstration of common server-side application flaws. The exercises are intended to be used by people 
to learn about application security and penetration testing techniques.

This project is under the OWASP flag and I'm one of the project leaders.


[[Github]](https://github.com/WebGoat/WebGoat) [[Homepage]](https://webgoat.github.io/WebGoat/) [[OWASP Page]](https://www2.owasp.org/www-project-webgoat/)

## OWASP Dependency-Check all in one image

A Docker image that is created every day at 0:00 with a local data file so OWASP Dependency-Check is ready to analyze your project. This image is self-contained, it contains the H2 database that can run as a Docker image from the pipeline. No need to setup a database maintain it etc.

[[Github]](https://github.com/nbaars/owasp-dependency-check-as-one) [[OWASP Dependency-Check]](https://owasp.org/www-project-dependency-check/)


## paseto4j

Implementation of PASETO library written in Java. This library is focused on taking part of the encryption/decryption 
part of the tokens it has a little dependencies as possible. 

What is Paseto?

Paseto is everything you love about JOSE (JWT, JWE, JWS) without any of the many design deficits that plague the JOSE 
standards. Paseto (Platform-Agnostic SEcurity TOkens) is a specification and reference implementation for secure 
stateless tokens.

[[Github]](https://github.com/nbaars/paseto4j) [[Paseto.io]](https://paseto.io/)

## pwnedpasswords4j

A Java client for checking a password against pwnedpasswords.com using the `Searching by range` API, see [here](https://haveibeenpwned.com/API/v2#SearchingPwnedPasswordsByRange
) for more details.

[[Github]](https://github.com/nbaars/pwnedpasswords4j) [[Homepage]](https://haveibeenpwned.com/API/v2#SearchingPwnedPasswordsByRange)