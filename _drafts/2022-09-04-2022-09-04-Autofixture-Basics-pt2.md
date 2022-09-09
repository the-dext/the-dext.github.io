---
layout: post
title: AutoFixture Basics Part 2 - Using Anonymous Data
draft: true
publish: true
tags: C# dotNet TDD Test-Driven-Development
date: 2022-09-09 20:41 +0000
excerpt_separator: <!--more-->

---

# Writing Unit Tests With Anonymous Data

In my previous article I setup a fictitious project to demonstrate how brittle unit tests can create a burden on developers as they implement applications and take time to refactor their codebase.
This is the complete opposite of what you want and need from unit tests, it's vital that unit tests are there to help you refactor with confidence, rather than discourage you from refactoring.

In this second article I am going to demonstrate how the tests we wrote can be rewritten using AutoFixture and Anonymous data.
<!--more-->