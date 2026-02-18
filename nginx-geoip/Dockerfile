# syntax=docker/dockerfile:1

############################
# 1️⃣  Build stage
############################
FROM debian:bookworm-slim AS builder
ARG NGINX_VERSION=1.29.0

RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc make wget git \
        libpcre3-dev zlib1g-dev libssl-dev \
        libmaxminddb-dev ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src

# Sources
RUN wget "http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" \
    && tar xzf "nginx-${NGINX_VERSION}.tar.gz"
RUN git clone --depth 1 https://github.com/leev/ngx_http_geoip2_module.git

WORKDIR /usr/src/nginx-${NGINX_VERSION}

# Compile nginx with GeoIP2 as a dynamic module
RUN ./configure \
        --prefix=/etc/nginx \
        --sbin-path=/usr/sbin/nginx \
        --modules-path=/usr/lib/nginx/modules \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --pid-path=/var/run/nginx.pid \
        --lock-path=/var/run/nginx.lock \
        --http-client-body-temp-path=/var/cache/nginx/client_temp \
        --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
        --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
        --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
        --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
        --user=nginx --group=nginx \
        --with-compat --with-file-aio --with-threads \
        --with-http_ssl_module --with-http_v2_module \
        --with-http_stub_status_module --with-http_realip_module --with-http_sub_module \
        --with-stream_ssl_module --with-stream_ssl_preread_module \
        --add-dynamic-module=/usr/src/ngx_http_geoip2_module --with-stream \
    && make -j"$(nproc)" && make install


############################
# 2️⃣  Runtime stage
############################
FROM debian:bookworm-slim
ARG NGINX_VERSION=1.29.0
LABEL org.opencontainers.image.title="nginx ${NGINX_VERSION} with GeoIP2"

RUN apt-get update && apt-get install -y --no-install-recommends \
        libpcre3 zlib1g libssl3 libmaxminddb0 ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# unprivileged user
RUN groupadd --system nginx && \
    useradd  --system --gid nginx --no-create-home \
             --home /nonexistent --shell /sbin/nologin nginx

# copy build artefacts
COPY --from=builder /usr/sbin/nginx        /usr/sbin/nginx
COPY --from=builder /etc/nginx             /etc/nginx
COPY --from=builder /usr/lib/nginx/modules /usr/lib/nginx/modules

# runtime dirs
RUN mkdir -p /var/cache/nginx/{client,proxy,fastcgi,uwsgi,scgi}_temp \
             /var/log/nginx /usr/share/nginx/html /usr/share/GeoIP && \
    chown -R nginx:nginx /var/cache/nginx /var/log/nginx /usr/share/GeoIP

##### default nginx.conf #####
RUN cat >/etc/nginx/nginx.conf <<'EOF'
load_module modules/ngx_http_geoip2_module.so;

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log;
pid        /var/run/nginx.pid;

events { worker_connections  1024; }

http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # GeoIP2 config – uncomment when DB mounted
    # geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    #     $geoip2_data_country_code country iso_code;
    # }

    include /etc/nginx/conf.d/*.conf;
}
EOF
#######################

##### default vhost #####
RUN mkdir -p /etc/nginx/conf.d && cat >/etc/nginx/conf.d/default.conf <<'EOF'
server {
    listen 80 default_server;
    server_name _;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html { root /usr/share/nginx/html; }
}
EOF
#########################

RUN echo "<h1>NGINX ${NGINX_VERSION} with GeoIP2</h1>" \
    > /usr/share/nginx/html/index.html

# container logs
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80
STOPSIGNAL SIGQUIT
CMD ["nginx", "-g", "daemon off;"]
