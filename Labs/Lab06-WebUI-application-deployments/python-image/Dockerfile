FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages
RUN pip install Flask
RUN pip install Redis

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME 'Bootcamp Lab06'

# Run app.py when the container launches
CMD ["python", "app.py"]