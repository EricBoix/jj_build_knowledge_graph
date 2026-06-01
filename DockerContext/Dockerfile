FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY *.py .
COPY env-reference .

CMD ["python", "extracting_graph.py", "--help"]
