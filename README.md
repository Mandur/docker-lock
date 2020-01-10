# About
[![Go Report Card](https://goreportcard.com/badge/github.com/michaelperel/docker-lock)](https://goreportcard.com/report/github.com/michaelperel/docker-lock)
[![Build Status](https://dev.azure.com/michaelsethperel/docker-lock/_apis/build/status/michaelperel.docker-lock?branchName=master)](https://dev.azure.com/michaelsethperel/docker-lock/_build/latest?definitionId=4&branchName=master)

`docker-lock` is a [cli-plugin](https://github.com/docker/cli/issues/1534) that uses Lockfiles (think `package-lock.json` or `Pipfile.lock`) to manage image digests. It allows developers to refer to images by their tags, yet receive the same immutability guarantees as if they were referred to by their digests.

If you are unsure about the differences between tags and digests, refer to this [quick summary](#tags-vs-digests).

# Demo
Consider a project with a multi-stage build Dockerfile at its root:
```
FROM ubuntu AS base
# ...
FROM mperel/log:v1
# ...
FROM python:3.6
# ...
```
Running `docker lock generate` from the root queries each images' registry to produce a Lockfile, `docker-lock.json`.

![Generate GIF](gifs/generate.gif)

Note that the Lockfile records image digests. Running `docker lock verify` ensures that the image digests are the same as those on the registry for the same tags.

Now, assume that a change to `mperel/log:v1` has been pushed to the registry. Running `docker lock verify` shows that the image digest in the Lockfile is out-of-date because it differs from the newer image's digest on the registry.

![Verify GIF](gifs/verify.gif)

While developing, it can be useful to generate a Lockfile, commit it to source control, and verify it periodically (for instance on PR merges). In this way, developers can be notified when images change, and if a bug related to a change in an image crops up, it will be easy to identify.

Finally, lets assume the Dockerfile is ready to be built and shared. Running `docker lock rewrite` will add digests from the Lockfile to all of the images.

![Rewrite GIF](gifs/rewrite.gif)

At this point, the Dockerfile will contain all of the digest information from the Lockfile, so it will always maintain the same, known behavior in the future.

# Features
* Supports Dockerfiles and docker-compose v3 files.
* Integrates with docker as the top level command, `docker lock`.
* Out of the box support for public and private repos on Dockerhub, Azure Container Registry, and others via the standard `docker login` command or environment variables.
* Easily extensible to any registry compliant with  [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/).
* Rich CLI flags with smart defaults that make selecting Dockerfiles and docker-compose files easy.
* Installable by placing a single executable binary into docker's cli-plugins folder.
* Written in Go.

# Install
## Linux / Mac
* Docker version >= 19.03
* `mkdir -p ~/.docker/cli-plugins`
* `curl -fsSL https://github.com/michaelperel/docker-lock/releases/download/{VERSION}/docker-lock-{OS} -o ~/.docker/cli-plugins/docker-lock`
* `chmod +x ~/.docker/cli-plugins/docker-lock`
## Windows
* Docker version >= 19.03
* Create the folder `%USERPROFILE%\.docker\cli-plugins`
* Download `docker-lock-windows.exe` from the releases page.
* Rename the file `docker-lock.exe`
* Move `docker-lock.exe` into `%USERPROFILE%\.docker\cli-plugins`

# Tags Vs. Digests
Images can be referenced by tag or digest. For instance, at the time of writing this README, the official most recent version of the python 3.6 image on Dockerhub could be specified by tag, as in `python:3.6`, or by digest of the image's contents, as in `python@sha256:25a189a536ae4d7c77dd5d0929da73057b85555d6b6f8a66bfbcc1a7a7de094b`.

Images referenced by tag are mutable. The python maintainers could push a new image to Dockerhub with the same 3.6 tag. Downstream applications that required the previous `python:3.6` image could break.

Images referenced by digest are immutable. Despite having the same tag, a newly pushed image will have a new digest. The previous image can still be referenced by the previous digest.

When deploying to Kubernetes, digests make it easy to rollback broken deployments. If your previous, working deployment relied on `myimage@sha256:2273f9a536ae4d7c77d6h49k29da73057b85555d6b6f8a66bfbcc1a7a7de094b` and the broken, updated deployment relies on `myimage@sha256:92038492583f9a3a4d7c77d6h49k29057b85555d6b6f8a66bfbcc1a7a7d1947f`, rolling back to the working deployment would be as easy as changing the digest back to the previous digest. Alternatively, if the previous, working deployment relied on `myimage:v1` and the broken, updated image relies on  `myimage:v1`, it would be more challenging to rollback by distinguishing between the images.

Although digests solve mutability problems, manually specifying them comes with a host of problems. Namely:
* Applications will no longer benefit from updates (security updates, performance updates, etc.).
* Dockerfiles and docker-compose files will become stale.
* Digests are considerably less readable than tags.
* Keeping digests up-to-date can become unwieldly in projects with many services.
* Specifying the correct digest is complicated. Local digests may differ from remote digests, and there are many different types of digests (manifest digests, layer digests, etc.)

`docker-lock` solves all of these problems by storing digests in a Lockfile, allowing developers to simply use tags since digests are recorded in the background.

# Contributing
## Development
* A development container based on `ubuntu:bionic` has been provided, so ensure docker is installed and the docker daemon is running

If using VSCode's [Remote Development Extension - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack):
* Open the project in VSCode
* In the command palette (ctrl+shift+p on Windows/Linux, command+shift+p on Mac), type "Reopen in Container"
* In the command palette type: "Go: Install/Update Tools" and select all
* When all tools are finished installing, in the command palette type: "Developer: Reload Window"
* SSH credentials are automatically mapped into the container
* The docker daemon is mapped from the host into the dev container, so you can use docker and docker-compose commands from within the container as if they were run on the host

Without VSCode:
* Build the development container: `docker build -f .devcontainer/Dockerfile -t dev .`
* Mount the root directory into the container, and drop into a bash shell: `docker run -it -v ${PWD}:/workspaces/docker-lock dev`

## CI
* Unit tests, integration tests, and linting run in the [CI pipeline](https://dev.azure.com/michaelsethperel/docker-lock/_build?definitionId=4) on pull requests.
* To format your code: `./tools/format.sh`
* To lint your code: `./tools/lint.sh`
* To run unit tests: `./tools/unittest.sh`
* To generate a coverage report: `./tools/coverage.sh`
* To view the coverage report on your browser, open a console, but not in docker, run `go tool cover -html=coverage.out`
