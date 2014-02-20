---
title: Dynamic language bridges using remote procedure call
author: Benjamin Pringle; Mukkai Krishnamoorthy
numbersections: true
bibliography: references.bib
---

#### Abstract {-}

A simple strategy is presented for dynamically interpreting remote procedure
calls in scripting languages, resulting in the ability to transparently use
libraries from both languages in a single program. The protocol described
handles complex, nested arguments and object-oriented libraries.

Example implementations show two useful bridges: one between Ruby and Python,
and another between two different Python runtimes.

# Introduction

A **language bridge** allows a program written in one language to use functions
or libraries from a different language.

## Existing techniques

### Common Intermediate Language

### Foreign Function Interface

### Remote Procedure Call

## Desired features

-   cross-runtime (mix Java and C libraries)
-   dynamic (no tedious "glue code")

# Protocol

## JSON over IPC

## Dynamic introspection

## Object Proxies

# Implementation

## Ruby to Python

## Jython to CPython

# Analysis

## Edge cases

## Speed

# Conclusion
