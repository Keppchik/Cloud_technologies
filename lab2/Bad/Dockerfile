FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip

COPY . /app

WORKDIR /app

RUN pip3 install --break-system-packages -r requirements.txt

CMD ["python3", "app.py"]