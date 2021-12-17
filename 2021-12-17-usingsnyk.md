---
title: "Using Snyk with Travis"
created_at: Fri Dec 17 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2021-12-17-usingsnyk
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

<img width="732" alt="Screen Shot 2021-12-10 at 1 33 28 PM" src="https://user-images.githubusercontent.com/20936398/145644269-cf5be5f7-6ee7-4808-b418-de5ff10bf7f0.png">

Snyk is a developer security platform. Integrating directly into development tools, which means it can integrate with Travis. In this example, I'll show you how Snyk is going to scan a `HCL` file I have, a container, and other components in my Travis configuration before we deploy our newest idea to the world. 

<!-- more --> 

## Usage

Let's define the language in the `.travis.yml` as `node`. You'll notice we are also grabbing `pipenv`, because in the repository I'll be giving you at the end of this, will have an example for Python as well. Let's take a look at my `.travis.yml` config:

```yaml
install:
  - pip install pipenv
language: node_js
node_js:
  - lts/*
script:
  - npm install -g snyk@latest # Globally install Snyk via node package manager, using condition `@latest` for latest version.
  - snyk -v # Print out the current version of Snyk symlinked. 
  - snyk code
  - snyk test --docker debian --file=Dockerfile --exclude-base-image-vulns # Scan the Palantir Cassandra container. 
  - snyk iac test variable.tf # Test an IaC method, say in this case Terraform. With simple variables that really equal to moot.
  ```

## Snyk Environment Variables 

It's important to note you'll need your Snyk `env vars`. You'll need to fetch those from Snyk, and then name the `env var` something like `SNYK_TOKEN`.


