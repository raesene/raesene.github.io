---
layout: post
title: "Fun with Containers - Adding tracking to your images" 
date: 2023-02-11 09:27:00 +0100
comments: false
categories: Containers
---

Last year I was taking a look at the OCI Image specification and I came across something kind of interesting, which is how you can get a container image to ping a URL when it's `pulled` to a host almost like a tracking cookie. Needless to say this isn't me dropping 0-day, I reported this to the containerd and podman security addresses back in October 2022 and the consensus appears to be that whilst this may have some security implications it's not dreadfully serious in most cases and it's part of the spec, so unlikely to change. 

With that said, it's an interesting way to investigate a bit about how the OCI spec works and some of the tooling that goes with it, so lets dive in.

## OCI Image Specification

Reading through the [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md) like any specification can be a bit hard to reason about as there's a lot of text there, which you'd expect in a document which has to be precise about what it's defining. So to help me parse out what was happening I tried to create a mermaid.js diagram of the different elements, which ended up looking like this :-

![OCI Image Mermaid diagram]({{site.url }}/assets/media/oci-image-spec.png)

Looking at the graph, I noticed that several of the sections, including config, manifest, and layers are of type `descriptor`. Looking at the [type definition](https://github.com/opencontainers/image-spec/blob/main/descriptor.md) I saw that there was a `urls` field which is listed as OPTIONAL. Seeing this made me wonder, what happens if you specify that in a container image?

At this point I had an experiment to try out, so the next step was to create an image that specifies a `url` parameter...

## Setting up the base image

The first thing to note here is a bit of container image history. Usually if you use [Docker](https://www.docker.com/) to create images you don't actually get an OCI specification image, you get a docker format image. Whilst these are generally interoperable, there are som differences which matter for the purpose of this experiment, so we can't just do what we might usually and use a docker image as a starting point.

To demonstrate this I created a very simple Dockerfile

```dockerfile
FROM busybox


CMD ["/bin/sh"]
```

Then built it first with standard `docker build` and then with `docker buildx build -o type=oci` to get an OCI image. Extracting the tarballs for both these shows how the two formats differ

**Docker image**
```bash
383fca235ae5442f82b630fed2d6ed8306cc4f6f490e41f56c7fddbdf5f795be.json
5e558067e843bda80c685292e5fcc6d3a1c01cea5f13896cc891d70bc4067e07/
5e558067e843bda80c685292e5fcc6d3a1c01cea5f13896cc891d70bc4067e07/VERSION
5e558067e843bda80c685292e5fcc6d3a1c01cea5f13896cc891d70bc4067e07/json
5e558067e843bda80c685292e5fcc6d3a1c01cea5f13896cc891d70bc4067e07/layer.tar
manifest.json
repositories
```

**OCI Image**
```bash
blobs/
blobs/sha256/
blobs/sha256/245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9
blobs/sha256/c5b5e79770f0f14d204f1bfdda52533a39b140eec9d39c15d165b41af3972feb
blobs/sha256/f5b7ce95afea5d39690afc4c206ee1bf3e3e956dcc8d1ccd05c6613a39c4e4f8
index.json
manifest.json
oci-layout
```

Now we know how to create an OCI image to use we need to modify one of the sections to include our `url` parameter.

## Modifying the image

One of the feature of OCI images is that they use SHA-256 hashes to identify the different elements, so if we modify the contents of a file, we then need to re-compute the hash of the file and update the references to it. What I found was that the easiest section to modify, which has the `url` parameter set, is the manifest layer. So if we get the manifest from our OCI image above we can add a `url` parameter to it like this

```json
{
   "mediaType": "application/vnd.oci.image.manifest.v1+json",
   "schemaVersion": 2,
   "config": {
      "mediaType": "application/vnd.oci.image.config.v1+json",
      "digest": "sha256:245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9",
      "size": 625,
      "urls": [
         "http://logs.pwndland.uk/245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9"
       ]
   },
   "layers": [
      {
         "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
         "digest": "sha256:f5b7ce95afea5d39690afc4c206ee1bf3e3e956dcc8d1ccd05c6613a39c4e4f8",
         "size": 772998
      }
   ]
}
```

I've added the URL `http://logs.pwndland.uk/245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9` to the `config` section of the manfiest. This URL includes the SHA-256 hash of the config file, which is important as it's checked by the OCI runtime when it's pulled (you can use different urls but you'll get more errors and less successful pulls that way).

When we add this change to the manifest we've modified the hash of the manifest file, so we need to update this anywhere it appears in the image to make things work. There's a couple of steps here :-

- Run `sha256sum` on the modified manifest file to get the new hash
- run `mv [old hash] [new hash]` to rename the file
- run `stat -c%s [new_manifest_hash_value]` to get the size of the file
- In `index.json` update the `manifests` section to use the new hash and size

At this point you should have a valid image, so now we just need to upload to a registry to test it.

## Uploading to a registry

The obvious way to do this might be to `tar` up the image and then use `docker load` and `docker push` to upload to a registry, however that doesn't work as the process will modify the image and remove the `url` parameter. So we need to use a different method.

The best tool I found for doing this without mangling our image, is [crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md). Using `crane push` we can upload the directory to our registry without having to create a tarball first.

In terms of which registries this will work with, I've tested Docker Hub, GitHub and Quay.io and they all work (it also works with harbor run locally).

### Running our webserver

To receive the pings from our images being pulled, we need a webserver to receive them. I used Caddy for this as it works pretty well. You can see some general notes about using Caddy that I made [here](https://raesene.github.io/blog/2023/01/21/Fun-with-Caddy-SSRF-Testing/). In this case our Caddyfile can be pretty simple as we just need it to serve files from a directory and log the requests to a file.

```caddyfile
logs.pwndland.uk:80 {
  root * /home/ubuntu/bad_images/hashes
  file_server
  log {
    output file bad_image_access_log.log
  }
}
```

### Testing it out

So now we have our modified image hosted in a registry, the question is.... does it work? The answer to this turned out to be a little bit varied depending on the tool used to pull the image.

If we use Docker to pull the image, the pull works and we get no ping back to our webserver.

If we use `nercdctl` and `containerd` we get something like this. You can see the `User-agent` header gives us some information about the tool used, you also get the source IP address that pulled it (I've redacted in this case as the test's run from my home network :) )

```bash
{"level":"info","ts":1676109995.5399737,"logger":"http.log.access.log6","msg":"handled request","request":{"remote_ip":"[REDACTED]","remote_port":"50758","proto":"HTTP/1.1","method":"GET","host":"logs.pwndland.uk","uri":"/245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9","headers":{"User-Agent":["containerd/1.6.0+unknown"],"Accept":["application/vnd.oci.image.config.v1+json, */*"],"Accept-Encoding":["gzip"]}},"user_id":"","duration":0.000304641,"size":625,"status":200,"resp_headers":{"Etag":["\"ro9zwzhd\""],"Content-Type":[],"Last-Modified":["Tue, 10 Jan 2023 15:19:47 GMT"],"Accept-Ranges":["bytes"],"Content-Length":["625"],"Server":["Caddy"]}}
```

Pulling the image with `podman` will ping as well with a different user agent string showing a different library used for the interaction with registries.

```bash
{"level":"info","ts":1676109882.1299412,"logger":"http.log.access.log6","msg":"handled request","request":{"remote_ip":"[REDACTED]","remote_port":"38378","proto":"HTTP/1.1","method":"GET","host":"logs.pwndland.uk","uri":"/245c6832cd7a449df9ce7b95d94569329c13fb05ccb38f58f537191b75d258b9","headers":{"User-Agent":["containers/5.16.0 (github.com/containers/image)"],"Docker-Distribution-Api-Version":["registry/2.0"],"Accept-Encoding":["gzip"],"Connection":["close"]}},"user_id":"","duration":0.000237844,"size":625,"status":200,"resp_headers":{"Content-Length":["625"],"Server":["Caddy"],"Etag":["\"ro9zwzhd\""],"Content-Type":[],"Last-Modified":["Tue, 10 Jan 2023 15:19:47 GMT"],"Accept-Ranges":["bytes"]}}
```

Other tools which work with OCI images, such as `skopeo` will also trigger the ping, each with their own User-Agent.

## Avoiding this

If someone tracking your image pulls is a concern, then avoiding this is generally a matter of ensuring that you pull trusted images from trusted registries. If you're using a public registry (e.g. Docker hub or ghcr) then you're already disclosing your IP address and user agent to a third party...

## Conclusion

As with the general tenor of my "fun with" series there's no earth shattering payoff here, but I found this an interesting way to run through a specification and find possibly unintended behaviour. Also doing this helped me learn quite a bit about the OCI image specification and how it's used.