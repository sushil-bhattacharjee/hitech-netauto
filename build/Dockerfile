FROM python:3.12-slim

WORKDIR /app

# System deps for paramiko/cryptography (ncclient transport) and lxml
RUN apt-get update && apt-get install -y --no-install-recommends \
      libssl-dev \
      libffi-dev \
      ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps first (better layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app
COPY main.py .
COPY templates/ templates/

EXPOSE 7071

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "7071"]
