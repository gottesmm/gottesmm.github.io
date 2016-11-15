---
layout: proposal
title: SIL Ownership Verifier
categories: proposals
---

# {{ page.title }}

# Summary

This document proposes an ownership verifier to ensure that a SIL program is
ownership correct. This is done by causing all SSA def-use edges to have an
ownership implication associated with them. Once such a
