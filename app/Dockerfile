FROM python:3.6-slim

# Create non-root user and group
RUN addgroup --gid 10001 appgroup && \
    adduser --uid 10001 --gid 10001 --disabled-password appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

# Give access of /app directory to our user
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

CMD ["python", "app.py"]
