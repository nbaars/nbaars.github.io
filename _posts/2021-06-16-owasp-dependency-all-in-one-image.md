---
layout: post
title: All in one image for OWASP Dependency Check
categories: [Java, Security]
---

Running OWASP Dependency Check locally is quite easy, wait once a long time then run it over and over. But once you are trying to setup a CI pipeline with Dependency Check a lot of things get complicated quite easily.

## Introduction

As explained locally it works fine, but when you want to use it in a CI environment things become a bit more complicated, you don't want to download the database each time. Of course most CI pipelines you have the option to share for example the `.m2/repository` directory as a cache and hope that one of the runners already downloaded the complete NIST database from the internet. If not available, several runners will probably start downloading the database at the exact same time which could result in `HTTP/429 - too many requests`. There are solutions available that include a database but then you still need to set up the central database, a quote from the [website](https://jeremylong.github.io/DependencyCheck/data/database.html):

> WARNING: This discusses an advanced setup, and you may run into issues.

Even though there are Docker images that take care of setting up the database etc you would still need to setup a volume for the database, run it in your infrastructure as the CI runner need to be able to connect to the database for the analysis.

### Basic idea

What about having an image that is created every day at 0:00 with a local data file so OWASP Dependency Check is ready to analyze your project. This image is self-contained, it contains the H2 database that can run as a Docker image from the pipeline. No need to setup a database maintain it etc.

### Benefits

- Easy to use
- Every day there is a new image waiting to be used in your CI environment
- No need to mount the directory which contains a copy of the database to run the Docker image it is all available in the image
- Scanner runs offline (database is contained in the image)
- Fast (no need to wait to download the CVEs)
- No need to setup a database with persistent storage etc to hold the configuration.
- No need to configure a central database as described [here](https://jeremylong.github.io/DependencyCheck/data/database.html).

This image was created based on a personal itch, setting it up in a pipeline took too much time, and running a Docker image on for example a Kubernetes cluster to which the client connects feels like too much effort to me.


## Implementation

The complete Docker can be found [here](https://github.com/nbaars/owasp-dependency-check-as-one). Let's focus on the main parts:

```dockerfile
FROM azul/zulu-openjdk-alpine:16.0.1-16.30.15-jre

WORKDIR /dependency-check
RUN apk add curl && \
    curl -L -o dependency-check-6.1.6-release.zip https://github.com/jeremylong/DependencyCheck/releases/download/v6.1.6/dependency-check-6.1.6-release.zip && \
    unzip dependency-check-6.1.6-release.zip && \
    dependency-check/bin/dependency-check.sh --data data -s .
```

In short, we download the latest version, unzip it and run the `dependency-check` command to let it initialize itself, this will download all the NIST databases, and it will create the local `H2` database.

Next up is creating our final image:

```dockerfile
FROM azul/zulu-openjdk-alpine:16.0.1-16.30.15-jre

RUN addgroup -S owasp && adduser -S owasp -G owasp
USER owasp

WORKDIR /dependency-check
COPY --chown=owasp:owasp --from=builder /dependency-check/data data
COPY --chown=owasp:owasp --from=builder /dependency-check/dependency-check .
```

we copy the `data` directory from the builder and the contents of the directory `dependency-check` which we extracted in the builder. This way we can run the command line directly.

## Running

Now that we have the container setup we can run it in different ways. The Github repository contains a demo project which has a Jackson library that contains a vulnerability. To scan it we use:

```shell
docker pull nbaars/owasp-dependency-check-as-one:latest

docker run -v ${HOME}/.m2:/home/owasp/.m2 -v ${PWD}/demo-project:/workspace nbaars/owasp-dependency-check-as-one:latest ./mvnw dependency:copy-dependencies && dependency-check --data /data --scan /workspace --noupdate
```

When you run this command you can see it runs fast, and indeed finds the Jackson vulnerability as expected.

### Github actions

Now let's show how to run it as part of your Github actions:

```shell
- name: "Run tests"
  uses: nbaars/owasp-dependency-check-as-one:latest
  with:
    image: test-image:latest
    options: "--user root -v ${{ github.workspace }}/demo-project:/workspace:rw"
    run: |
      ./test.sh
```

Where the `test.sh` file is a simple test we use during the build of a branch to verify the image still finds the mentioned Jackson vulnerability:

```shell
testAnalyzingDemoProjectShouldFindViolation() {
  ./mvnw org.owasp:dependency-check-maven:6.1.6:aggregate -DautoUpdate=false -DdataDirectory=/data > output.txt
  grep "jackson-databind-2.9.1.jar" output.txt
  assertEquals "0" "$?"
  grep "CVE-2017-17485" output.txt
  assertEquals "0" "$?"
}
```

## Conclusion

I hope this image helps you to get OWASP Dependency Check up and running in your pipeline without running into the obvious downsides when using it in a CI pipeline. 