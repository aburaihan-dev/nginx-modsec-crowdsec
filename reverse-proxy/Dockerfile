FROM debian:12.1-slim

# Install dependencies
RUN apt-get update \
    && apt-get install -y \
    build-essential \
    libpcre2-dev libpcre3 libpcre3-dev \
    libssl-dev \
    git \
    zlib1g-dev \
    wget \
    automake \
    autoconf \
    libtool

# Fetch and compile ModSecurity
RUN git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity
WORKDIR /ModSecurity
RUN git submodule init && git submodule update
RUN ./build.sh && ./configure && make && make install

# Fetch ModSecurity-nginx connector
WORKDIR /
RUN git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git

# Fetch and compile Nginx
RUN wget https://nginx.org/download/nginx-1.24.0.tar.gz
RUN tar zxvf nginx-1.24.0.tar.gz
WORKDIR /nginx-1.24.0
RUN ./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --http-log-path=/var/log/nginx/access.log \
    --error-log-path=/var/log/nginx/error.log \
    --add-dynamic-module=/ModSecurity-nginx \
    --with-http_stub_status_module \
    --with-http_ssl_module \
    --with-select_module \
    --with-poll_module \
    --with-threads \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-stream_realip_module \
    --with-http_slice_module \
    --with-pcre-jit \
    --with-zlib-asm=CPU \
    --with-stream=dynamic \
    --with-pcre \
    && make \
    && make install 

RUN make modules

# Copy the ModSecurity configuration file
RUN cp /ModSecurity/modsecurity.conf-recommended /etc/nginx/modsecurity.conf
RUN cp /ModSecurity/unicode.mapping /etc/nginx/

# Enable SecRuleEngine
RUN sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsecurity.conf

RUN echo "# Include custom OWASP rules \tInclude /etc/nginx/crs/owasp-crs.conf" >> /etc/nginx/modsecurity.conf

# Copy the OWASP ModSecurity Core Rule Set (CRS)
RUN git clone --depth 1 -b v3.3/master --single-branch https://github.com/coreruleset/coreruleset /etc/nginx/crs

RUN cp /etc/nginx/crs/crs-setup.conf.example /etc/nginx/crs/crs-setup.conf


# Add the line to load ngx_http_modsecurity_module at the top of nginx.conf
RUN sed -i '1i\load_module modules/ngx_http_modsecurity_module.so;' /etc/nginx/nginx.conf

# Configure Nginx to use ModSecurity
RUN sed -i '/http {/a \\tmodsecurity on;\n\tmodsecurity_rules_file /etc/nginx/modsecurity.conf;' /etc/nginx/nginx.conf

RUN touch /var/log/nginx/error.log

VOLUME [ "/var/log/nginx" ]

# Expose port 80
EXPOSE 80
EXPOSE 443

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]