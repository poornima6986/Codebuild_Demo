FROM python:3.6

ENV FLASK_RUN_HOST=0.0.0.0

WORKDIR /src

COPY ./requirements.txt requirements.txt

# Install the dependencies required for the Application during build inthe container itself
RUN pip install -r requirements.txt

COPY . .

CMD python app.py
