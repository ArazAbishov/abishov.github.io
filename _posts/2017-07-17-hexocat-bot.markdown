---
layout: post
title: >
  Building a simple chatbot for slack in Rust using Rocket and Anterofit frameworks
date: 2017-07-17 22:00:00 +0000
---

## Introduction (refactor)
Rust is a systems programming language which enables developers to write safe and fast code without sacrificing high-level language constructs. At first it seems that Rust is targeting only performance critical use cases, but original intention is far more ambitious. Frameworks like Rocket, Serde and Anterofit make Rust a good fit for web application development as well.

This blogpost is all about building a simple slack bot, which allows to search github repositories. Obviously, you don't need to develop a simple chatbot in systems programming language, but intention of this blogpost is to show how powerful Rust can be. The whole implementation is less 150 lines code, which is quite amazing.

## Structure   
 - Anterofit (write a simple query to the GitHub APIs)
 - Rocket (use for exposing to slack)
 - Using ngrok for development with slack

## Future
 - Deploying and managing the bot on Ubuntu under nginx
 - Deploying the bot on RaspberryPi
