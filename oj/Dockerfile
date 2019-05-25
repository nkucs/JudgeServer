FROM python:3.7-alpine3.9

ENV OJ_ENV production

ADD . /app
WORKDIR /app

RUN mkdir -p /data/test_case

# HEALTHCHECK --interval=5s --retries=3 CMD python2 /app/deploy/health_check.py
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk add --update build-base nginx openssl curl unzip supervisor jpeg-dev zlib-dev postgresql-dev freetype-dev
RUN pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
RUN pip install --no-cache-dir -r /app/requirements.txt
RUN apk del build-base --purge

EXPOSE 8100

CMD gunicorn -b :8100 oj.wsgi