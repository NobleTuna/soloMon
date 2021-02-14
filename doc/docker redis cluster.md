### docker-compose.yml
```
version: '3'
services:
  redis_01:
    image: redis:5.0-alpine
    build:
      context: .
      dockerfile: Dockerfile
    network_mode: "host"
    environment:
      - REQUIREPASS=<passwd>
      - CLIENTHOST=192.168.0.81
      - CLIENTPORT=6381
    volumes:
      - "/home/nobletuna/docker/data/redis_01:/data"
    restart: always

```

>redis docker 기본 이미지에는 redis.conf가 포함되어 있지 않음

### Dockerfile
```
FROM redis:alpine

MAINTAINER Carrey (jaehun2841@gmail.com ) / nobletuna

# Copy Redis File
# 복사/추가 하는파일의 Container내 경로는 항상 절대경로로 작성하여야 합니다.
# 기존의 docker-entryporint.sh 파일을 삭제합니다.
RUN rm -rf /usr/local/bin/docker-entrypoint.sh
# 공통적으로 적용할 redis.conf 파일을 복사합니다.
ADD redis.conf /usr/local/bin/redis.conf

# Container가 생성, 시작하는 시점에 실행 시킬 docker-entrypoint.sh 파일을 복사합니다.
ADD docker-entrypoint.sh /usr/local/bin

## change access authority
RUN chmod 755 /usr/local/bin/redis.conf
RUN chmod 755 /usr/local/bin/docker-entrypoint.sh

RUN chown redis:redis /usr/local/bin/redis.conf
RUN chown redis:redis /usr/local/bin/docker-entrypoint.sh

#Redis Container에 대한 Port를 지정합니다. (내부포트이며 외부노출은 안됨)
EXPOSE $CLIENTPORT
#CLIENTPORT에 대한 값은 docker-compose.yml에 정의되어 있습니다.
# Container가 생성, 시작하는 시점에 실행됩니다.
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
# Container 빌드가 완료되고 Redis Server를 실행시킵니다.
CMD [ "redis-server","/usr/local/bin/redis.conf" ]

```

### docker-entrypoint.sh
```
#!/bin/sh
set -e

## from redis-5
sed -i "s/bind 127.0.0.1/bind $CLIENTHOST 127.0.0.1/g" /usr/local/bin/redis.conf
sed -i "s/port 6379/port $CLIENTPORT/g" /usr/local/bin/redis.conf
sed -i "s/# requirepass foobared/requirepass $REQUIREPASS/g" /usr/local/bin/redis.conf
sed -i "s/# masterauth <master-password>/masterauth $REQUIREPASS/g" /usr/local/bin/redis.conf
sed -i "s/# cluster-enabled yes/cluster-enabled yes/g" /usr/local/bin/redis.conf
sed -i "s/# cluster-config-file nodes-6379.conf/cluster-config-file nodes.conf/g" /usr/local/bin/redis.conf
sed -i "s/# cluster-node-timeout 15000/cluster-node-timeout 5000/g" /usr/local/bin/redis.conf

# first arg is `-f` or `--some-option`
# or first arg is `something.conf`
if [ "${1#-}" != "$1" ] || [ "${1%.conf}" != "$1" ]; then
        set -- redis-server "$@"
fi

# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
        chown -R redis .
    exec su-exec redis "$@"
fi

exec "$@"

```

### docker-compose 실행
`docker-compose -f docker-compose.yml up --build -d`

-f : docker-compose.yml 파일을 설정  
up : docker container 실행  
--build : build후 container를 실행  
-d : background로 실행

<br>

### redis-clustering.sh
```
redis-cli -p 6381 -a "passwd" --cluster create 192.168.0.81:6381 192.168.0.81:6383 192.168.0.81:6385 \
192.168.0.81:6382 192.168.0.81:6384 192.168.0.81:6386 \
--cluster-replicas 1

```


### redis-clustering.sh 실행
```
nobletuna@ubuntu01:~$ ./redis-clustering.sh
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.0.81:6384 to 192.168.0.81:6381
Adding replica 192.168.0.81:6386 to 192.168.0.81:6383
Adding replica 192.168.0.81:6382 to 192.168.0.81:6385
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 1423116612ac8cc5b43e8ce87eafb2bd73e9ed35 192.168.0.81:6381
   slots:[0-5460] (5461 slots) master
M: 29264ac55f24641056062affa7de67a8f52f445f 192.168.0.81:6383
   slots:[5461-10922] (5462 slots) master
M: 6a0ae6606da5cda32a831eeeed51675212a915ba 192.168.0.81:6385
   slots:[10923-16383] (5461 slots) master
S: fb33663a9d9decfac3bede3bff7ac79eb65aa7e1 192.168.0.81:6382
   replicates 1423116612ac8cc5b43e8ce87eafb2bd73e9ed35
S: 4ff14273e82af36d0b22ee141ee4d1e557cf8282 192.168.0.81:6384
   replicates 29264ac55f24641056062affa7de67a8f52f445f
S: 3423d787bbdffb4286d37467b812d513854b608c 192.168.0.81:6386
   replicates 6a0ae6606da5cda32a831eeeed51675212a915ba
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node 192.168.0.81:6381)
M: 1423116612ac8cc5b43e8ce87eafb2bd73e9ed35 192.168.0.81:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: fb33663a9d9decfac3bede3bff7ac79eb65aa7e1 192.168.0.81:6382
   slots: (0 slots) slave
   replicates 1423116612ac8cc5b43e8ce87eafb2bd73e9ed35
S: 4ff14273e82af36d0b22ee141ee4d1e557cf8282 192.168.0.81:6384
   slots: (0 slots) slave
   replicates 29264ac55f24641056062affa7de67a8f52f445f
M: 29264ac55f24641056062affa7de67a8f52f445f 192.168.0.81:6383
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 6a0ae6606da5cda32a831eeeed51675212a915ba 192.168.0.81:6385
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 3423d787bbdffb4286d37467b812d513854b608c 192.168.0.81:6386
   slots: (0 slots) slave
   replicates 6a0ae6606da5cda32a831eeeed51675212a915ba
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```




## 참고

https://jaehun2841.github.io/2018/12/03/2018-12-03-docker-10/#fail-over

