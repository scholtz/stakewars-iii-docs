# Build neard

compose-neard.sh

```
if [ "$ver" == "" ]; then
ver=shardnet-neard-1.28.0-dev
fi

echo "docker build -t \"scholtz2/stakewars:$ver\" -f Dockerfile context"
docker build -t "scholtz2/stakewars:$ver" -f Dockerfile context || error_code=$?
if [ "$error_code" != "" ]; then
echo "$error_code";
    echo "failed to build";
	exit 1;
fi

docker push "scholtz2/stakewars:$ver" || error_code=$?
if [ "$error_code" != "" ]; then
echo "$error_code";
    echo "failed to push";
	exit 1;
fi

echo "Image: scholtz2/stakewars:$ver"
```

context/start.sh

```
#!/bin/bash
echo "$@"

if [ ! -f /app/.near/config.json ]
then
    echo "Welcome to near blockchain. Downloading snapshot."
	wget -c http://build.openshards.io.s3.amazonaws.com/stakewars/shardnet/data.tar.gz
	tar -xvzf data.tar.gz -C /app/.near/
	neard --home /app/.near init --chain-id shardnet --download-genesis
	rm /app/.near/config.json && wget -O /app/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/shardnet/config.json
else
    echo "Welcome to near blockchain"
fi
neard --home "/app/.near" run
```

Dockerfile

```
FROM ubuntu:latest as build
USER root
ENV DEBIAN_FRONTEND noninteractive
RUN apt update && apt dist-upgrade -y && apt install -y mc wget telnet git curl iotop atop vim && apt-get clean autoclean clang build-essential make && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN curl -ysSf https://sh.rustup.rs | sh
WORKDIR /tools
RUN curl -L https://raw.githubusercontent.com/tj/n/master/bin/n -o n
RUN bash n 18
RUN npm install -g n
RUN node -v
RUN npm install -g npm@8.14.0
RUN npm -v
RUN npm install -g near-cli
ENV NEAR_ENV=shardnet
RUN git clone https://github.com/near/nearcore /tools/nearcore
WORKDIR /tools/rust
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
RUN ls
WORKDIR /tools/nearcore
RUN ls
RUN git checkout 1.28.0
RUN ls /root/.cargo/bin
ENV PATH="/root/.cargo/bin:$PATH" 
RUN apt update && apt dist-upgrade -y && apt install build-essential -y && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN apt update && apt dist-upgrade -y && apt install libclang-dev -y && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN cargo build -p neard --release --features shardnet
ENV PATH="/tools/nearcore/target/release:$PATH" 
WORKDIR /app

FROM ubuntu:latest
USER root
ENV DEBIAN_FRONTEND noninteractive
RUN apt update && apt dist-upgrade -y && apt install -y mc wget telnet git curl iotop atop vim jq && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN useradd -d /app -ms /bin/bash app
COPY start.sh /app/start.sh
RUN chown app:app /app/start.sh && chmod 0700 /app/start.sh
USER app
WORKDIR /app
COPY --from=build /tools/nearcore/target/release/neard /app/near/neard
ENV PATH="/app/near:$PATH" 
RUN echo 'export NEAR_ENV=shardnet' >> /app/.bashrc
ENTRYPOINT [ "/app/start.sh" ]
```
