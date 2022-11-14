---
title: CSAW Finals 2022 - smoothie operator (pwn)
layout: post
permalink: /2022/csaw-finals
description: ""
---

This was a C++ heap note I solved during CSAW finals, which I played with PPP.

Exploit summary
========

Uninitalized memory in `Monster` can be used to get a heap leak. Off-by-one validaton error in `Monster::edit_params` allows a 4-byte write at a specific out-of-bounds offset from a `Monster` heap chunk. Align heap chunks so that the OOB write occurs into a `std::vector<std::string*>` of Smoothie 'ingredients'. This can be used as an arbitrary-free primative, since these `std::string`s and their backing memory will be free'd when a Smoothie order is cancelled. Craft fake chunks in a large `Complaint`, and use the arbitrary-free primative to free a fake unsorted-bin sized chunk, getting a libc address written into the middle of a `Complaint`.  Print out the complaint to get a libc leak. Lastly, construct a fake std::string pointing to 



