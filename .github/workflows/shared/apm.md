---
name: APM Package Manager
description: Shared workflow for installing APM packages

inputs:
  packages:
    description: List of APM package references to install
    required: true
    type: array

---

# APM Package Installation

This workflow handles installation of APM packages for agent primitives.
