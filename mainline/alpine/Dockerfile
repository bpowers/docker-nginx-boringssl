FROM alpine:3.8 as builder

MAINTAINER Bobby Powers "bobbypowers@gmail.com"

ENV NGINX_VERSION 1.15.3

RUN GPG_KEYS=B0F4253373F8F6F510D42178520A9993A1C052F8 \
	&& addgroup -S nginx \
	&& adduser -D -S -h /var/cache/nginx -s /sbin/nologin -G nginx nginx \
	&& apk add --no-cache --virtual .build-deps \
		autoconf \
		automake \
		bind-tools \
		binutils \
		build-base \
		ca-certificates \
		cmake \
		curl \
		gcc \
		gd-dev \
		geoip-dev \
		git \
		gnupg \
		gnupg \
		go \
		libc-dev \
		libgcc \
		libstdc++ \
		libtool \
		linux-headers \
		make \
		pcre \
		pcre-dev \
		su-exec \
		tar \
		tzdata \
		zlib \
		zlib-dev \
	&& curl -fSL https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz -o nginx.tar.gz \
	&& curl -fSL https://nginx.org/download/nginx-$NGINX_VERSION.tar.gz.asc  -o nginx.tar.gz.asc \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $GPG_KEYS from $server"; \
		gpg --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$GPG_KEYS" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $GPG_KEYS" && exit 1; \
	gpg --batch --verify nginx.tar.gz.asc nginx.tar.gz \
	&& rm -rf "$GNUPGHOME" nginx.tar.gz.asc \
	&& mkdir -p /usr/src \
	\
	&& (git clone --depth=1 https://github.com/bagder/libbrotli /usr/src/libbrotli \
		&& cd /usr/src/libbrotli \
		&& ./autogen.sh && ./configure CPPFLAGS='-D_FORTIFY_SOURCE=2' CFLAGS='-fstack-protector-strong -O2 -g' && make V=1 -j$(getconf _NPROCESSORS_ONLN) && make install) \
	&& git clone --depth=1 --recurse-submodules https://github.com/google/ngx_brotli /usr/src/ngx_brotli \
	&& (git clone --depth=1 https://boringssl.googlesource.com/boringssl /usr/src/boringssl \
		&& sed -i 's@out \([>=]\) TLS1_2_VERSION@out \1 TLS1_3_VERSION@' /usr/src/boringssl/ssl/ssl_lib.cc \
		&& sed -i 's@ssl->version[ ]*=[ ]*TLS1_2_VERSION@ssl->version = TLS1_3_VERSION@' /usr/src/boringssl/ssl/s3_lib.cc \
		&& sed -i 's@(SSL3_VERSION, TLS1_2_VERSION@(SSL3_VERSION, TLS1_3_VERSION@' /usr/src/boringssl/ssl/ssl_test.cc \
		&& sed -i 's@\$shaext[ ]*=[ ]*0;@\$shaext = 1;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& sed -i 's@\$avx[ ]*=[ ]*[0|1];@\$avx = 2;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& sed -i 's@\$addx[ ]*=[ ]*0;@\$addx = 1;@' /usr/src/boringssl/crypto/*/asm/*.pl \
		&& mkdir -p /usr/src/boringssl/build /usr/src/boringssl/.openssl/lib /usr/src/boringssl/.openssl/include \
		&& ln -sf /usr/src/boringssl/include/openssl /usr/src/boringssl/.openssl/include/openssl \
		&& touch /usr/src/boringssl/.openssl/include/openssl/ssl.h \
		&& CPPFLAGS='-D_FORTIFY_SOURCE=2' CFLAGS='-O2 -D_FORTIFY_SOURCE=2 -fstack-protector-strong' CXXFLAGS='-O2 -D_FORTIFY_SOURCE=2 -fstack-protector-strong' cmake -B/usr/src/boringssl/build -H/usr/src/boringssl \
		&& make VERBOSE=1 -C/usr/src/boringssl/build -j$(getconf _NPROCESSORS_ONLN) \
		&& cp /usr/src/boringssl/build/crypto/libcrypto.a /usr/src/boringssl/build/ssl/libssl.a /usr/src/boringssl/.openssl/lib) \
	\
	&& tar -zxC /usr/src -f nginx.tar.gz \
	&& rm nginx.tar.gz \
	&& cd /usr/src/nginx-$NGINX_VERSION \
	&& curl -fSL https://gist.githubusercontent.com/bpowers/5b7002caac11b56dc9845ccd66b48052/raw/6cc894c432fa8c6267607c99309124413e58fbfc/tls-dynamic-record-resizing.patch -o dynamic_tls_records.patch \
	&& patch -p1 <dynamic_tls_records.patch

# use hand-built, hardened geoip library
RUN cd /usr/src \
  && rm -f /usr/lib/libGeoIP* \
  && curl -fSL -o geoip.tar.gz https://github.com/maxmind/geoip-api-c/releases/download/v1.6.12/GeoIP-1.6.12.tar.gz \
  && tar -xzf geoip.tar.gz \
  && cd GeoIP-1.6.12 \
  && ./configure CPPFLAGS='-D_FORTIFY_SOURCE=2' CFLAGS='-fPIC -fstack-protector-strong -O2 -g' --enable-static=yes --enable-shared=no --prefix=/usr --sysconfdir=/etc/geoip --mandir=/usr/share/man --infodir=/usr/share/info \
  && make -j$(getconf _NPROCESSORS_ONLN) \
  && make install

RUN CONFIG="\
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
		--user=nginx \
		--group=nginx \
		--with-http_ssl_module \
		--with-http_realip_module \
		--with-http_addition_module \
		--with-http_sub_module \
		--with-http_mp4_module \
		--with-http_gunzip_module \
		--with-http_gzip_static_module \
		--with-http_secure_link_module \
		--with-http_auth_request_module \
		--with-http_geoip_module=dynamic \
		--with-threads \
		--with-stream \
		--with-stream_ssl_module \
		--with-stream_ssl_preread_module \
		--with-stream_realip_module \
		--with-stream_geoip_module=dynamic \
		--with-http_slice_module \
		--with-compat \
		--with-file-aio \
		--with-http_v2_module \
		--with-ipv6 \
		--add-dynamic-module=/usr/src/ngx_brotli \
	" \
	&& cd /usr/src/nginx-$NGINX_VERSION \
	&& ./configure $CONFIG --with-debug --with-cc-opt='-O2 -D_FORTIFY_SOURCE=2 -fstack-protector-strong -fPIC -I/usr/src/boringssl/.openssl/include' --with-ld-opt='-fPIE -pie -Wl,-z,relro -Wl,-z,now -L/usr/src/boringssl/.openssl/lib' \
        && make -j$(getconf _NPROCESSORS_ONLN) \
	&& mv objs/nginx objs/nginx-debug \
	&& mv objs/ngx_http_geoip_module.so objs/ngx_http_geoip_module-debug.so \
	&& mv objs/ngx_stream_geoip_module.so objs/ngx_stream_geoip_module-debug.so \
	&& ./configure $CONFIG  --with-cc-opt='-O2 -D_FORTIFY_SOURCE=2 -fstack-protector-strong -fPIC -I/usr/src/boringssl/.openssl/include' --with-ld-opt='-fPIE -pie -Wl,-z,relro -Wl,-z,now -L/usr/src/boringssl/.openssl/lib' \
	&& make -j$(getconf _NPROCESSORS_ONLN) \
	&& make install \
	&& rm -rf /etc/nginx/html/ \
	&& mkdir /etc/nginx/conf.d/ \
	&& mkdir -p /usr/share/nginx/html/ \
	&& install -m644 html/index.html /usr/share/nginx/html/ \
	&& install -m644 html/50x.html /usr/share/nginx/html/ \
	&& install -m755 objs/nginx-debug /usr/sbin/nginx-debug \
	&& install -m755 objs/ngx_http_geoip_module-debug.so /usr/lib/nginx/modules/ngx_http_geoip_module-debug.so \
	&& install -m755 objs/ngx_stream_geoip_module-debug.so /usr/lib/nginx/modules/ngx_stream_geoip_module-debug.so \
	&& ln -s ../../usr/lib/nginx/modules /etc/nginx/modules \
	&& strip /usr/sbin/nginx* \
	&& strip /usr/lib/nginx/modules/*.so \
	&& rm -rf /usr/src/nginx-$NGINX_VERSION \
	&& rm -rf /usr/src/boringssl /usr/src/libbrotli /usr/src/ngx_* \
	\
	# Bring in gettext so we can get `envsubst`, then throw
	# the rest away. To do this, we need to install `gettext`
	# then move `envsubst` out of the way so `gettext` can
	# be deleted completely, then move `envsubst` back.
	&& apk add --no-cache --virtual .gettext gettext \
	&& mv /usr/bin/envsubst /tmp/ \
	\
	&& runDeps="$( \
		scanelf --needed --nobanner /sbin/tini /usr/sbin/nginx /usr/lib/nginx/modules/*.so /tmp/envsubst \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	) tini tzdata ca-certificates" \
	&& apk add --no-cache --virtual .nginx-rundeps $runDeps \
	&& mv /tmp/envsubst /usr/local/bin/ \
	\
	# forward request and error logs to docker log collector
	&& ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log \
	&& mkdir -p /etc/nginx/geoip \
	&& curl -fSL https://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz | gunzip >/etc/nginx/geoip/GeoIP.dat \
	&& curl -fSL https://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz | gunzip >/etc/nginx/geoip/GeoLiteCity.dat \
	&& curl -fSL https://download.maxmind.com/download/geoip/database/asnum/GeoIPASNum.dat.gz | gunzip >/etc/nginx/geoip/GeoIPASNum.dat

FROM alpine:3.7
RUN apk --no-cache add \
    bash \
    ca-certificates \
    diffutils \
    geoip \
    pcre \
    tini \
    tzdata \
  && addgroup -S nginx \
  && adduser -D -S -h /var/cache/nginx -s /sbin/nologin -G nginx nginx

COPY --from=builder \
  /etc/nginx \
  /etc/nginx/
COPY --from=builder \
  /usr/lib/nginx \
  /usr/lib/nginx/
COPY --from=builder \
  /usr/share/nginx \
  /usr/share/nginx/
COPY --from=builder \
  /var/cache/nginx \
  /var/cache/nginx/
COPY --from=builder \
  /var/log/nginx \
  /var/log/nginx/
COPY --from=builder \
  /usr/sbin/nginx \
  /usr/sbin/
COPY --from=builder \
  /usr/sbin/nginx-debug \
  /usr/sbin/

COPY nginx.conf /etc/nginx/nginx.conf
COPY nginx.vh.no-default.conf /etc/nginx/conf.d/default.conf

LABEL description="NGINX Docker built top of rolling release BoringSSL" \
      maintainer="Denis Denisov <denji0k@gmail.com>" \
      openssl="BoringSSL" \
      nginx="nginx $NGINX_VERSION"

EXPOSE 80 443

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["nginx", "-g", "daemon off;"]
