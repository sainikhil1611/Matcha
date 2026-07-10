FROM python:3.12-slim-bookworm
WORKDIR /app

# The Devpost scraper (scraper_selenium.py / DevpostScraperService in
# app.py) drives headless Chrome via Selenium -- install a matching
# Chromium + chromedriver so /api/scrape has a browser available.
RUN apt-get update \
    && apt-get install -y --no-install-recommends chromium chromium-driver \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
