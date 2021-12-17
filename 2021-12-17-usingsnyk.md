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

![Snyk + TCI](https://user-images.githubusercontent.com/20936398/146601257-d681f9a4-ad1a-463f-a998-1000421e38e7.png)

Snyk is a developer security platform. Integrating directly into development tools, which means it can integrate with Travis. In this example, I'll show you how Snyk is going to scan a `HCL` file I have, a container, and other components in my Travis configuration before we deploy our newest idea to the world. 

<!-- more --> 

## Usage

Let's define the language in the `.travis.yml` as `node`. You'll notice we are also grabbing `pipenv`, because in the repository I'll be giving you at the end of this, I will have an example for Python as well. Let's take a look at my `.travis.yml` config:

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

## Things that we will be testing 

So, in this particular use case of Travis and Snyk, I decided to grab Palantir's Dockerfile of Apache Cassandra, the `Dockerfile` is as follows:

```Dockerfile
FROM cassandra:2.2

ARG BUILD_VERSION

ENV CASSANDRA_VERSION $BUILD_VERSION
ENV CASSANDRA_DIR palantir-cassandra-$BUILD_VERSION
ENV PATH $CASSANDRA_DIR/bin:$PATH

# Untar the Palantir distribution

ADD palantir-cassandra-*tgz /

# We give the cassandra user permissions to create things in the Cassandra dir

RUN ["sh", "-c", "chmod +rwx ${CASSANDRA_DIR}"]
RUN ["sh", "-c", "chown cassandra:cassandra ${CASSANDRA_DIR}"]

# Configs set up for Travis/Snyk scans via yaml, and some bash scripting

ENV CASSANDRA_CONFIG $CASSANDRA_DIR/conf/
ENV CASSANDRA_YAML $CASSANDRA_CONFIG/cassandra.yaml
ENV CASSANDRA_ENV $CASSANDRA_CONFIG/cassandra-env.sh
COPY cassandra.yaml $CASSANDRA_YAML
COPY cassandra-env.sh $CASSANDRA_ENV

ENV _JAVA_OPTIONS=-Dcassandra.skip_wait_for_gossip_to_settle=0

# Enable wide JMX access for tests, copied from https://stackoverflow.com/questions/48007037/run-remote-nodetool-commands-on-cassandra-3-without-authentication

RUN sed -i s/'jmxremote.authenticate=true'/'jmxremote.authenticate=false'/g $CASSANDRA_ENV
RUN sed -i s/'LOCAL_JMX=yes'/'LOCAL_JMX=no'/g $CASSANDRA_ENV
RUN sed -i s/'JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.password.file'/'#JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.password.file'/g $CASSANDRA_ENV
RUN sed -i s/'JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.access.file'/'#JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.access.file'/g $CASSANDRA_ENV

# Entry set up

COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN ["chmod", "+x", "/docker-entrypoint.sh"]

EXPOSE 7000 7001 7199 9042 9160
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["cassandra", "-f"]`
```
Okay, so you have your `Dockerfile` the `.travis.yml` file I made, time to get Snyk up and running, open up your CLI and run the following in your project directory:

```bash
snyk auth
```

<img width="569" alt="screen" src="https://user-images.githubusercontent.com/20936398/146600633-549dd143-f900-4acd-8f6b-84c9c903b390.png">

Follow the prompts, and now maybe do a trial run in your CLI, run the following command: 

```bash
snyk test --docker debian --file=Dockerfile --exclude-base-image-vulns
```
In this case I used the `--exclude-base-image-vulns` flag when it's running the checks on the Palantir Cassandra Dockerfile, this is something similar you'll see when you run it locally:

<img width="991" alt="snykvuln" src="https://user-images.githubusercontent.com/20936398/146601206-91ca3318-b68c-4ef6-a2c4-cc607c5e2622.png">

## Testing IaC's With Snyk and Travis CI 

Now let's say you have a Terraform config file, named the classic `main.tf`, and want to check that, pretty simple to do that as well. So let's make a test Terraform config file in `HCL`, here's what I coded out for this example: 

```hcl
provider "local" {
  version = "~> 1.4"
}
resource "local_file" "hello snyk, montana, palantir, cassandra and travis" {
  content = "Hello, user"
  filename = "foobar.txt"
}
```
Now that we have `main.tf` saved in the directory, just run the following in your CLI while you're doing the local tests:

```bash
snyk iac test main.tf
```

You should see something that resembles the following: 

<img width="689" alt="terraform" src="https://user-images.githubusercontent.com/20936398/146601912-907bec06-5de1-48fd-9aa7-9a339fe77564.png">

## Travis CI 

Alright, well now that you have a basic understand of what Snyk does and how we are going to integrate it into our Travis project, let's first trigger a build, you'll want to remember those `env vars`, when it starts building, you'll see that Snyk gets pulled via `npm install -g snyk@latest`, then I have it run `snyk -v` which tells you what version of Snyk you're running inside of Travis, then it will start going on ahead with the testing: 


<img width="687" alt="tester" src="https://user-images.githubusercontent.com/20936398/146602355-e068d537-255b-419e-af1b-0a4e246f588c.png">

You'll start seeing Snyk doing these to any of the files you declared in your `.travis.yml`, making sure there are close to no vulnerabilities. You can also test containers, so for example you can run in your CLI: 

```bash
snyk container monitor <your-image>
```

You can also add this to your `.travis.yml`. so let's say I had a container over at Docker entitled `container.tar`, we could add this line in our `script:` hook in our Travis config file:

```yaml
- snyk container test docker-archive:container.tar
```

There you have it, making things secure is always important when it comes to software development, end to end and up and down the stack. You now know how to integrate Snyk with your Travis CI builds, just another presentation of how flexible Travis CI is using integrations. 
