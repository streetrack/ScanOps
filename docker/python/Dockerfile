FROM python:3.11.10
WORKDIR /app
COPY requirements.txt app/ /app/
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "app.py"]
