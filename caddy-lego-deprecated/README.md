# Caddy Lego-Deprecated

It is [deprecated](https://github.com/caddy-dns/lego-deprecated), but for some users there are no other options than to use this, as [libdns](https://github.com/libdns/libdns/) is still very limited. 

And in April 2024 [Caddy removed support](https://github.com/caddyserver/caddy/issues/6228) for lego ... go figure.

For DNS providers like Cloudflare this is reasnoable easy to get to work - just follow the documentation. For providers like Simply.com it is a bit more work, as there are no examples available. I will try to rectify this.

I use docker for everything, it makes my life simple (mostly), and it is easy to replicate a setup across systems - k8s (kubes) is another option, but it woudld be too much work for what I need.



## Caddy with lego-deprecated
(stolen from [williamjacksn /
docker-caddy-lego](https://github.com/williamjacksn/docker-caddy-lego/blob/master/Dockerfile))

Create a directory, in my example I call it 'caddy'. 

1. create a Dockerfile with the following content:
```
FROM caddy:2.8.4-builder AS builder

RUN xcaddy build v2.8.4 \
    --with github.com/caddy-dns/lego-deprecated

FROM caddy:2.8.4

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```
I normally create a directory for these things, so in ./build/caddy create a Dockerfile with the above content.

2. create a docker-compose.yml (extra formatting might be required)
```yaml
services:
  caddy:
    container_name: caddy
    hostname: caddy
    image: acme/caddy-lego:latest
    build:
      context: ./build/caddy
      dockerfile: Dockerfile
      tags:
        - acme/caddy-lego:latest
    environment:
      - TZ=Europe/Copenhagen
      - SIMPLY_ACCOUNT_NAME=<your account key>
      - SIMPLY_API_KEY=<your api key>
    networks:
      public:
        ipv4_address: 10.20.0.10
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./data/caddy/data:/data
      - ./data/caddy/config:/config
    healthcheck:
      test: [ "CMD-SHELL", "nc -z localhost 443 || exit -1" ]
      interval: 30s
      timeout: 5s
      retries: 5
    restart: unless-stopped
		
networks:
  public:
    name: public
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: "10.20.0.0/24"
          ip_range: "10.20.0.64/26"
          gateway: "10.20.0.1"
```


3. Create a Caddyfile in ./data/caddy
```
{
    email me@acme.dk
}

(simply) {
  tls {
    dns lego_deprecated simply
  }
}

*.acme.com {
  import simply

  @home host home.acme.com
  handle @home {
    reverse_proxy http://homey
  }
}
```

4. Create the image:
```
docker compose build --pull
```

5. Create and run the container
```
docker compose up -d && docker compose logs -f
```


This is bascially it, it might not work ... 