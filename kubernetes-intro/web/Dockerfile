FROM nginx
COPY ./app /usr/share/nginx/html
COPY ./conf/test.conf /etc/nginx/conf.d/
COPY ./conf/nginx.conf /etc/nginx/nginx.conf

RUN useradd -u 1001 web-user

RUN apt-get update && apt-get install -y procps

EXPOSE 8000
