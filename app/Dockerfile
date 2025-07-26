#FROM python:3.11-slim
#WORKDIR /app
#COPY requirements.txt .
#RUN pip install -r requirements.txt
#COPY . .
#CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
# First stage: build dependencies
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# Second stage: slim runtime
FROM python:3.11-slim

WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
