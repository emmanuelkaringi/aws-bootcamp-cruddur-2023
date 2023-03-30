FROM 257344229169.dkr.ecr.us-east-1.amazonaws.com/cruddur-python:3.10-slim-buster

WORKDIR /backend-flask

COPY requirements.txt requirements.txt

RUN pip3 install -r requirements.txt

COPY . .

ENV FLASK_DEBUG=1

EXPOSE ${PORT}

# RUN cd backend-flask
# export FRONTEND_URL="*"
# export BACKEND_URL="*"
# python3 -m flask run --host=0.0.0.0 --port=4567
# Append /api/activities/home to the URL
# cd ..

CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0", "--port=4567"]

