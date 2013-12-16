# dogestry

Proof of concept for simple image storage for docker.

## prerequisites

* lz4 - https://code.google.com/p/lz4/ compiled and on the path
* go 1.2
* docker

Currently, the user running dogestry needs permissions to access the docker socket. [See here for more info][https://docs.docker.io/en/latest/use/basics/#sudo-and-the-docker-group]

Currently the docker socket's location isn't configurable.

## usage

### push

Push the `redis` image and its current tag to the `central` remote. The `central` remote is an alias to a remote defined in `dogestry.cfg`
```
dogestry push central redis
```

Push the `hipache` image to the s3 bucket `ops-goodies` with the key prefix `docker-repo` located in `us-west-2`:
```
dogestry push s3://ops-goodies/docker-repo/?region=us-west-2 hipache
```

### pull

Pull the `hipache` image and tag from the `central`.
```
dogestry pull central hipache
```

### config

Configure dogestry with `dogestry.cfg`. By default its looked for in `./dogestry.cfg`.

Dogestry can often run without a configuration file, but its there if you need it.

For example, using the config file, you can set up remote aliases for convenience or specifiy s3 credentials.

However, if you're bootstrapping a system, you might rely on IAM instance profiles for credentials and specify the
remote using its full url.

## discussion

In my organisation docker will be the way for us to move away from Capistrano's [`cap deploy`][cap].

Capistrano has a number of problems which docker solves neatly or sidesteps entirely. However to make the investment of
time and energy worthwhile in moving away, docker must solve all of the problems Capistrano presents.

It currently does not do this. Luckily most of these blockers are concentrated in the registry approach.

In capistrano:
* dependencies are not resolved until during deployment. 
  * If the services hosting these dependencies are down, we're unable to deploy.
  * If these services go down half way through a deploy onto multiple boxes: chaos.
  * This is particularly the case on fresh boxes, `bundle install` is a very expensive and coupled to external service uptime.
* Capistrano as software is complex
  * Maintinaing recipes is difficult
  * Debugging recipes is difficult
  * Testing ditto

Docker's registry doesn't solve these problems.

* The official registry (http://index.docker.io) is centralised.
  * Particularly on a fresh machine, if we can't pull the ubuntu image, we're out of luck.
  * So we need to ensure that images are available from somewhere internal
* Docker's interaction with docker-registry is complex and tightly coupled
  * Tied in with the first problem, there are no guarantees that docker won't go out to the official registry for images. This isn't acceptible for production.
  * It might delegate to the index for auth. Again, in production I would want to actively disable this.
* Setting up docker-registry is complex
  * It doesn't support secure setups out of the box. There's a suggestion to use basic auth, but no documentation on how to set it up.
  * I've spent a long time trying to work out how to get basic auth working, but haven't cracked it yet!
  * By comparison: docker's single go binary

### enter dogestry

Dogestry aims to solve these particular problems by simplifying the image storage story, while maintaining some of the convenience of
`docker run ubuntu`.

Dogestry's design aims to support a wide range of dumb and smart transports.

It centres around a common portable repository format.

### synchronisation

Using the new feature for de/serialising self-consistent image histories (`GET /images/<name>/get` and `POST /images/load`) 

* dogestry push - push images from local docker instance to the remote in the portable repo format
* dogestry pull - pull images from the remote into the local docker instance

### remotes

"Remotes" are the external storage locations for the docker images. Dogestry implements transports for each kind of remote, much
like git.

#### local remote

Dumb transport, synchronises with a directory on the same machine using normal filesystem operations and rsync.

#### s3 remote

Dumb transport, synchronises with an s3 bucket using the s3 api.

#### registry remote (not implemented)

Smart transport, synchronises with an instance of docker-registry.

#### others

Dedicated dogestry server, ssh, other cloud file providers.

### portable repository format

* able to serve as a repository over dumb transports (rsync, s3)

example layout:

images:
```
images/5d4e24b3d968cc6413a81f6f49566a0db80be401d647ade6d977a9dd9864569f/layer.tar
images/5d4e24b3d968cc6413a81f6f49566a0db80be401d647ade6d977a9dd9864569f/VERSION
images/5d4e24b3d968cc6413a81f6f49566a0db80be401d647ade6d977a9dd9864569f/json 
```

To better support eventually-consistent remotes using dumb transports (i.e. s3) The repositories json is unrolled into files (like `.git/refs`)
```
repositories/myapp/20131210     (content: 5d4e24b3d968cc6413a81f6f49566a0db80be401d647ade6d977a9dd9864569f)
repositories/myapp/latest       (content: 5d4e24b3d968cc6413a81f6f49566a0db80be401d647ade6d977a9dd9864569f)
```

#### optional - compression

I've chosen to use lz4 as the compression format as its very fast and for `layer.tar` still seems to provide reasonable compression ratios. 
There's a [go implementation] but there's no streaming version and I wouldn't know where to start in converting it.

Given that remotes are generally, well, remote, I don't think its a stretch to include compression for the portable repository format.

It probably should be optional though.

Currently its part of the Push/Pull command, but I intend to push the implementation down into the s3 remote.

#### optional - checksumming

Some remotes support cheap checksumming by default, others don't.

I've implemented checksumming as part of the s3 remote.

## docker changes
* dogestry can work with docker as-is
* at the very least, I'd like to add a flag to enact the zero external dependency requirement (this could be done with e.g. hacking /etc/hosts, but a docker flag would be neater & more pro).
* nice to have would be a refinement of `GET /images/<name>/get` to exclude images already on the remote.
* nice to have would be in-stream de/compression of layer.tar in `GET /images/<name>/get` and `POST /images/load`.
* best of all would be to integrate some different registry approaches into docker.

## TODO

- more tests.
- move compression into remote.
- more remotes.
- more tag operations


## conclusion

Although I'd like docker's external image storage approach to be more flexible and less complex, I can support what I need reasonably efficiently with current docker features.

I do hope that this code stimulates some discussion on the subject, but its main aim is to support my use-case.


[cap]: https://github.com/capistrano/capistrano
[golz4]: https://github.com/bkaradzic/go-lz4