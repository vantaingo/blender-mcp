FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    UV_IPV6=0

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl build-essential libssl-dev && \
    rm -rf /var/lib/apt/lists/*

COPY pyproject.toml uv.lock README.md ./
COPY src ./src
COPY main.py addon.py ./

RUN pip install --upgrade pip setuptools wheel && \
    pip install "mcp[cli]>=1.3.0" && \
    pip install .

EXPOSE 3000

ENTRYPOINT ["blender-mcp"]
