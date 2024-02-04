FROM python:3.10-slim-buster

USER root

WORKDIR /src

COPY ./requirements.txt requirements.txt

# Install the dependencies required for the Application during build inthe container itself
RUN pip install -r requirements.txt

COPY . .

CMD python app.py
