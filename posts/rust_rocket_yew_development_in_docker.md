# Docker Development for Rust Rocket Yew

Created: 01/31/2022

Developing a Rocket Yew App in Docker

## Setting up the development container

Create a directory for your code:

```bash
mkdir rust_website_tutorial
```

Now make a devcontainer directory in the rust_website_tutorial directory

```bash
cd rust_website_tutorial
mkdir devcontainer
```

In the devcontainer directory create a file called Dockerfile

./rust_website_tutorial/devcontainer/Dockerfile

```
#Base Container 
FROM rust:latest
#Add the cargo to the PATH
RUN echo "export PATH=$PATH:/usr/local/cargo/bin" >> /root/.bashrc
#Install the rust tools we want
RUN rustup target add wasm32-unknown-unknown
RUN cargo install trunk cargo-watch
```

Create a docker-compose.yml file as well

./rust_website_tutorial/devcontainer/docker-compose.yml

```
version: '3'

services:
  rust_website:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/workspace:cached
    command: tail -f /dev/null
    network_mode: "host"
    working_dir: /workspace/
```

To start the docker container run (from the devcontainer directory):

```bash
docker-compose up
```

I prefer to attach vscode to the running docker container. Which allows you develope inside the container seamlessly. This can be done using the Remote Development extension ms-vscode-remote.vscode-remote-extensionpack.

For additional help check out the documentation [Attach to a running container](https://code.visualstudio.com/docs/remote/attach-container).
