FROM nginx:1.18-alpine

ENV TZ=UTC

# nginx config file
COPY ./infra/docker/nginx/*.conf /etc/nginx/conf.d/

# Nginxコンテナに入った時の作業ディレクトリを指定
WORKDIR /var/www/html