# Build neard

compose-neard.sh

```
#nocache="--no-cache"
nocache=""
if [ "$ver" == "" ]; then
ver=shardnet-neard-g62f66348a
fi

echo "docker build -t \"scholtz2/stakewars:$ver\" -f neard.dockerfile context $nocache"
nice -n 5 docker build -t "scholtz2/stakewars:$ver" -f neard.dockerfile context $nocache || error_code=$?
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

docker tag scholtz2/stakewars:$ver scholtz2/stakewars:shardnet-neard-master
docker push "scholtz2/stakewars:shardnet-neard-master"
```

context/start.sh

```
#!/bin/bash
echo "$@"

if [ ! -f /app/.near/config.json ]
then
    echo "Welcome to near blockchain. Downloading snapshot."
	neard --home /app/.near init --chain-id shardnet --download-genesis
	rm /app/.near/config.json && wget -O /app/.near/config.json https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/shardnet/config.json
else
    echo "Welcome to near blockchain"
fi
neard --home "/app/.near" run
```

neard.dockerfile

```
FROM ubuntu:latest as build
USER root
ENV NEAR_ENV=shardnet
ENV DEBIAN_FRONTEND noninteractive
WORKDIR /tools/rust
RUN apt update && apt dist-upgrade -y && apt install -y mc wget telnet git curl iotop atop vim jq && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
WORKDIR /tools
RUN apt update && apt dist-upgrade -y && apt install -y build-essential libclang-dev && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
RUN echo RUN git checkout 62f66348ae7871e2c5879a60395f4003a9e9bbf1
RUN git clone https://github.com/near/nearcore /tools/nearcore
WORKDIR /tools/nearcore
RUN git checkout 62f66348ae7871e2c5879a60395f4003a9e9bbf1
RUN ls /root/.cargo/bin
ENV PATH="/root/.cargo/bin:$PATH" 
RUN cargo build -p neard --release --features shardnet
ENV PATH="/tools/nearcore/target/release:$PATH" 
#WORKDIR /tools/nearcore/pytest
#RUN apt update && apt dist-upgrade -y && apt install -y python3 pip && apt-get autoremove --yes && rm -rf /var/lib/{apt,dpkg,cache,log}/
#RUN python3 -m pip install -U -r requirements.txt
#RUN python3 tests/sanity/one_val.py


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

Autoupdate script

```
#!/bin/bash

x=`curl https://api.github.com/repos/near/nearcore/git/refs/heads/master`
ver=`echo $x | jq .object.sha | sed s/\"//g`
echo $ver

sed s~ver\=shardnet\-neard\-.*~ver\=shardnet\-neard\-g${ver:0:9}~g -i /home/scholtz/stakewars/compose-neard.sh
cat compose-neard.sh | grep shardnet-neard

echo "updated compose-neard.sh file"

sed s~RUN\ git\ checkout\ .*~RUN\ git\ checkout\ $ver~g -i /home/scholtz/stakewars/neard.dockerfile
cat neard.dockerfile | grep "RUN git checkout"

echo "updated neard.dockerfile file"

./compose-neard.sh || error_code=$?
if [ "$error_code" != "" ]; then
echo "$error_code";
    echo "failed to build";
	exit 1;
fi

sed s~scholtz2\/stakewars:shardnet-neard-g.*~scholtz2\/stakewars:shardnet-neard-g${ver:0:9}~g -i /home/scholtz/stakewars/fi-2-sch-nonarchiva.yaml
cat fi-2-sch-nonarchiva.yaml | grep stakewars:shardnet-neard

echo "updated fi-2-sch-nonarchiva.yaml file"

kubectl apply -f fi-2-sch-nonarchiva.yaml

echo "processed kubectl apply -f fi-2-sch-nonarchiva.yaml"
```

cron record

```
37 */3 * * * cd /home/scholtz/stakewars && ./upgrade-sch.sh 2> upgrade-error.txt 1> upgrade.txt
```
