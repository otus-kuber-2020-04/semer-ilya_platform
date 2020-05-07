FROM nginx

RUN useradd -u 1001 web-user
RUN mkdir /app && chown web-user:web-user /app

COPY ./app/homework.html /app/homework.html
COPY ./conf/test.conf /etc/nginx/conf.d/
COPY ./conf/nginx.conf /etc/nginx/nginx.conf

RUN apt-get update && apt-get install -y procps

EXPOSE 8000
