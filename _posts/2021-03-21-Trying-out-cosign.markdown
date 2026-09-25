---
layout: post
title: "Trying out Cosign" 
date: 2021-03-21 17:00:39 +0100
comments: false
categories: Docker
---

Container image signing has been a bit of a gap in the security landscape, so I'm always interested in seeing new projects starting up which address it. Docker Content Trust/Notary never really gained traction in v1, and whilst v2 looks very interesting, it's still in the design phase (AFAIK).

So seeing the [Cosign project](https://github.com/sigstore/cosign) come along as part of the [Sigstore](https://sigstore.dev/) initiative, I was interested to take a look at it and see how it works. Sigstore has some really interesting ideas about software transparency logs, but for this blog, I'll just be looking at the raw image signing process.

The use case I wanted to look at was the idea of getting a container image that's built as part of a CI pipeline and making a signature available so that people downloading the image from Docker hub could validate that the image they're downloading was built by me (well more precisely that it was built by someone who had access to the private key which was used to sign it).

## Installing cosign

It's a golang project, so it's fairly easy to get started, there's a single binary available from their [release page](https://github.com/sigstore/cosign/releases/tag/v0.1.0) and it has been signed by them.

Once you've got it you can generate a keypair `cosign generate-key-pair`. A very important point here, is to set a good long passphrase to protect your private key! This will provide you with a public and private key.

## Setting up Github Actions to build and sign your image.

With cosign installed and working, we need get our CI setup to sign the image as it's built. Cosign have done this with their setup but I wanted to do things a little differently. They put the private key in the repository and I'd prefer to try and keep it a bit more restricted. I'm sure that they've set a good passphrase on the key but making it available publicly does open you up to brute force attacks on the passphrase used, so I'd say ideally you'd want to avoid that.

The test repo I made for this process is [here](https://github.com/raesene/cosign_test). The image itself is just a super simple Dockerfile based on ubuntu 18.04 which adds the cosign binary to it, the interesting part is the GH actions file [here](https://github.com/raesene/cosign_test/blob/main/.github/workflows/ci-build-docker-image.yml)

Let's break down what we're doing in the action. I'm by no means an expert on these, but this seems to work ok :)

First step is setting up some output with today's date in this ([source](https://github.community/t/how-can-i-set-an-expression-as-an-environment-variable-at-workflow-level/16516/6)). I'm using that to tag the images. 

```yaml
    - name: Get current Date
      id: date
      run: echo "::set-output name=date::$(date +'%Y-%m-%d')"
```

Next we'll log into my account on Docker hub. For this you'll need to set a secret in the Github repository with a docker hub access token in it.

```yaml
    - name: Docker login
      run: echo {% raw %}${{secrets.DOCKER_PASSWORD}}{% endraw %} | docker login -u raesene --password-stdin
```

Then we'll build the Docker image and push it to Docker Hub.

```yaml
    - name: build Docker Image
      run: docker build -t raesene/cosign_test:{% raw %}${{steps.date.outputs.date}}{% endraw %} .
    - name: push to Docker hub
      run: docker push raesene/cosign_test:{% raw %}${{steps.date.outputs.date}}{% endraw %}
```

With our new image on Docker Hub, we need to sign it. One thing I thought about this flow is that it doesn't really protect against an active malicious Registry as they could theoretically modify my image as soon as it's uploaded then the sign commands hit afterwards. Pretty niche attack in most cases but could be worth considering.

What I've done in advance of this is get the passphrase protected Cosign private key and place it into a Github Secret called `COSIGN_KEY` . In order to have it available for cosign to read, I put it into a file (using a method from [here](https://stackoverflow.com/a/59482124/537897))

```yaml
    - name: place the cosign private key in a file
      run: 'echo "$COSIGN_KEY" > /tmp/cosign.key'
      shell: bash
      env:
        COSIGN_KEY: {% raw %}${{secrets.COSIGN_KEY}}{% endraw %}
```

Now with the signing key available inside the Action, I can sign the image

```yaml
    - name: Sign the image pushed
      run: echo -n "{% raw %}${{secrets.COSIGN_KEY_PASSPHRASE}}{% endraw %}" | ./cosign sign -key /tmp/cosign.key raesene/cosign_test:{% raw %}${{steps.date.outputs.date}}{% endraw %}
```

One interesting thing, is how cosign stores signatures. If you look on the Docker Hub page for the image that I'm using in the [tags tab](https://hub.docker.com/repository/docker/raesene/cosign_test/tags?page=1&ordering=last_updated) you'll see that cosign creates tags that it uses inside the repo. I think we'll be seeing more people using OCI registries for things other than pure container images, but this is the first time I've seen a tool take that approach.

## Verifying the image

Once you've got your image uploaded, anyone can use cosign to verify it. For my test image , the public key is

```yaml
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEoeqsxUUhzWrx70u/dCAf1QgBFMVF
eyqWrtbAfwDdjONf9gbhfzURQFyZvcL7ET5PEq36x0OS9enJShKzAJKkEQ==
-----END PUBLIC KEY-----
```

With that saved to a file called `cosign.pub` you should be able to run

```bash
cosign verify -key cosign.pub raesene/cosign_test:2021-03-21
```

and get output that looks like

```bash
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
  - WARNING - THE CERTIFICATE EXPIRY WAS NOT CHECKED. set COSIGN_EXPERIMENTAL=1 to check!
{"Critical":{"Identity":{"docker-reference":""},"Image":{"Docker-manifest-digest":"sha256:9b5a67e1e2e67f60d4e93529ff280f12601586c0c382949f96947001c0c6094f"},"Type":"cosign container signature"},"Optional":null}
```

If you try verifying a tag that doesn't have a signature like the `invalid` tag on that repository using `cosign verify -key cosign.pub raesene/cosign_test:invalid` you'll get an error

```bash
error: GET https://index.docker.io/v2/raesene/cosign_test/manifests/sha256-45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b.cosign: MANIFEST_UNKNOWN: manifest unknown; map[Tag:sha256-45c6f8f1b2fe15adaa72305616d69a6cd641169bc8b16886756919e7c01fa48b.cosign]
```

## Conclusion

It's early days for cosign (they just hit 0.1 2 days back), but it along with sigstore, look like a really promising area of tech. Hopefully we'll see more developments in this area and get some great new ideas for image (and general software) signing.