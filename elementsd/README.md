## Elementsd
`Dockerfile` downloads the compiled binaries from https://github.com/ElementsProject/elements/releases. The file was adapted from https://github.com/jamesob/docker-bitcoind. Whereas `Dockerfile.gitian` uses [gitian](https://github.com/devrandom/gitian-builder) to build from [source](https://github.com/ElementsProject/elements).

Feel free to adapt `build-and-push-to-dockerhub.sh` to push to your own repo/registry.

You can update Elements' version in the `Dockerfile` and/or `run-gitian.sh` based on the kind of image you want to build (e.g. `v0.17.1` or `commit_hash`).

### Building from source
If you're on MAC (or Windows), run inside an Ubuntu container (you need [Docker](https://docs.docker.com/install/#supported-platforms)). Assuming you've `git cloned && cd bitcoin-images`:
```
docker run -itd --name ub -v `pwd`/elementsd:/opt/elementsd -v /var/run/docker.sock:/var/run/docker.sock ubuntu:bionic sleep infinity
```
Open a shell inside the container (`docker exec -it ub bash`) and install the necessary packages:
```
apt-get update && \
apt-get install -y git ruby apt-transport-https ca-certificates curl gnupg-agent software-properties-common && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - && \
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable" && \
apt-get update && apt-get install docker-ce -y
cd /opt/elementsd 
./run_gitian.sh
```
You can watch the install or build logs by running `docker exec ub tail -f /opt/elementsd/gitian/var/install|build.log` in another terminal window.
After gitian has finished successfully, you can `Ctrl+D` out of the container and remove it `docker rm -f ub`.

Lastly, you can build the actual Docker image with the Elements binaries by
```
docker build -t blockstream/elementsd:latest -f Dockerfile.gitian .
``` 
And possibly adapt `build-and-push-to-dockerhub.sh` to build from source as well. 

### How to run
```
/etc/systemd/system/elements.service
[Unit]
Description=elementsd pseudo node
Wants=docker.target
After=docker.service

[Service]
Restart=always
RestartSec=3
ExecStartPre=/usr/bin/docker pull blockstream/elementsd:latest
ExecStart=/usr/bin/docker run \
    --network=host \
    --pid=host \
    --name=elements \
    -v /mnt/data/elements/elements.conf:/root/.elements/elements.conf:ro \
    -v /mnt/data/elements:/root/.elements:rw \
    blockstream/elementsd:latest elementsd -printtoconsole
ExecStop=/usr/bin/docker exec elements elements-cli stop
ExecStopPost=/usr/bin/sleep 3
ExecStopPost=/usr/bin/docker rm -f elements
```