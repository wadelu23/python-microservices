FROM python:3.9-slim
ENV PYTHONUNBUFFERED 1
WORKDIR /app
COPY requirements.txt /app/requirements.txt

RUN apt-get update && \
    apt-get install -y default-libmysqlclient-dev gcc

RUN pip install -r requirements.txt
COPY . /app
