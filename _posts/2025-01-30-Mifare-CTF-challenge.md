---
layout: post
title: "
<div style='display: flex; align-items: center; justify-content: space-between;'>
    <a href='/' style='font-size: 28px; font-weight: normal; color: #c1c1c1; text-decoration: none; margin-top: -50px;'>Home</a>
    <img src='https://raw.githubusercontent.com/pligonstein/pligonstein.github.io/main/images/logo.gif' alt='Logo' style='height: 48px; width: 48px; border-radius: 50%; object-fit: cover; margin-top: -50px;'>
</div>"
post_title: "Decrypting Mifare Communication"
date: 2025-01-30
categories: blog
---

## Introduction

Last year I participated in Asis CTF Finals and I was pretty interested in a 0 solve hardware challenge - `e-cart`. Today, I'm going to show you my solve and how easy it actually was after you figure out the beginning.

## Description

> We have a file associated with an [e-cart](https://asisctf.com/tasks/e-cart_b367be4e52ea05b975d9189d98f38f8538bef071.txz) that contains the key to the flag. Once you find the flag, ensure that you accurately fulfill all the requirements specified within it.

## Solve

First of all, you had to figure out that this is a trace file for a Mifare Classic nfc card and from there navigate your way to decrypting the communication. Fortunately, there is a tool [mfkey64](https://github.com/Proxmark/proxmark3/tree/master/tools/mfkey) which does just that. Make sure to retrieve the card uid, nt, nr, at and ar fields (think of those as the values used to establish the handshake) in order to be able to decrypt the communication.

> Use proxmark3 in order to analyse the trace file and retrieve the specified fields.

After spinning up proxmark3, load up the trace file and run `trace list -1 -t mf` to specify the type of nfc card that the trace belongs to. We're going to be greeted with some long output, but only the last part counts.


