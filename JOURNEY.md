# Development Journey

This project began with the goal of exploring and analyzing structured medical data from the [OpenFDA API](https://open.fda.gov/apis/drug/event/). The journey involved moving from raw JSON storage to a fully relational pipeline for analysis.

---

## Stage 1: Raw JSON to PostgreSQL

The initial approach was to fetch JSON data from OpenFDA and store each full record as a `jsonb` object in PostgreSQL.

```python
import psycopg2
import requests
import json

# Config
DB_NAME = "fda_json"
DB_USER = "postgres"
DB_PASSWORD = "@password"
DB_HOST = "localhost"
DB_PORT = "5432"

OPENFDA_URL = "https://api.fda.gov/drug/event.json?limit=10"

response = requests.get(OPENFDA_URL)
if response.status_code != 200:
    raise Exception(f"Failed to fetch data: {response.status_code}")

data = response.json()
records = data.get("results", [])

conn = psycopg2.connect(
    dbname=DB_NAME,
    user=DB_USER,
    password=DB_PASSWORD,
    host=DB_HOST,
    port=DB_PORT
)
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS fda_events (
    id SERIAL PRIMARY KEY,
    record JSONB
);
""")
conn.commit()

for record in records:
    cur.execute("INSERT INTO fda_events (record) VALUES (%s)", [json.dumps(record)])

conn.commit()
cur.close()
conn.close()
