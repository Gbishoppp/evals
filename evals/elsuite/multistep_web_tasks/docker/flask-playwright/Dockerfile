# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/playwright/python:v1.32.1-jammy

# Install Flask with pip cache
RUN --mount=type=cache,target=/root/.cache/pip pip3 install flask

# install wget for container ready check
RUN apt-get update && apt-get install -y wget

# Copy your Flask app code into the container
COPY . /app

# Set the working directory
WORKDIR /app

# Run the Flask app
CMD ["python3", "app.py"]