---
title: "Make Firestore awesome with generics"
# date: 2023-07-30T14:03:42+02:0
# TODO: 
author: "Vebjørn Daniloff"
description: "This article shows you how you can get rid of boilerplate code in Firestore with Swift"
draft: true
tags: ["UIKit", "Advanced", "Firestore", "Firebase", "Generics"]
categories: ["UIKit"]
series: ["messenger-clone"]
lightgallery: true
---

<!--more-->

## Introduction/Problem

In this episode I want to show you how you can optimize your interaction with Firestore without a service class. So what do I mean by a service class, and why would I take this approach? 

When I started out programming with Firestore a lot of tutorials was focused around creating separate classes for each collection that I could implement as a dependency into my ViewModel. Dependency injection in itself is a really good approach.
*show old code*

### Creating the service class
Like in the last video, we start with the bad approach and then we will go over the limitations before we move on to the solution.

### Validation
But before we start, we're going to use the Validation repository from the previous video as a starting point, since I don't want to implement a custom text input class from scratch. If you haven't watch that one, we made a custom text input class that can create different instances of itself (name, email, password) and it gives a validation feedback based on the current text input. I'll leave a link to it in the description below.

## UserService
Okay, cool. Let's create the service class I promised to create. We start with a function to create User documents 
