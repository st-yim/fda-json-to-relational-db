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
```

This preserved the structure but made querying difficult. For example, accessing patient.sex would require complex JSON path expressions inside SQL.


## Stage 2: Walking the JSON Tree

To understand the nested structure, I wrote a recursive parser that prints each key path and its value. This provided a conceptual map of the JSON hierarchy.

```python
import requests
import json

url = "https://api.fda.gov/drug/event.json?limit=1"
response = requests.get(url)
data = response.json()
record = data["results"][0]

def walk_json(node, path=""):
    if isinstance(node, dict):
        for key, value in node.items():
            new_path = f"{path}.{key}" if path else key
            walk_json(value, new_path)
    elif isinstance(node, list):
        for index, item in enumerate(node):
            new_path = f"{path}[{index}]"
            walk_json(item, new_path)
    else:
        print(f"{path}: {node}")

walk_json(record)
```
This helped expose every field and nested object. But while useful for inspection, it didn’t give a usable data model yet.

## Stage 3: Flattening into Relational Tables

Using `pandas` and `defaultdict`, I created a script that recursively extracts values into field paths and maps them into row-aligned tables.

```python
from collections import defaultdict
import pandas as pd

# Example list of records
records = [
    {
        "safetyreportid": "5801206-7",
        "patient": {
            "sex": "M",
            "age": 45,
            "risk_factors": {
                "genetic": True,
                "family_history": {
                    "mother": True,
                    "sister": False
                }
            }
        }
    },
    {
        "safetyreportid": "5801206-8",
        "patient": {
            "sex": "F",
            "age": 38,
            "risk_factors": {
                "genetic": False,
                "family_history": {
                    "mother": False,
                    "sister": True
                }
            }
        }
    }
]

tables = defaultdict(list)

def walk_json(node, path="", row_index=None):
    if isinstance(node, dict):
        for key, value in node.items():
            new_path = f"{path}.{key}" if path else key
            walk_json(value, new_path, row_index)
    elif isinstance(node, list):
        for idx, item in enumerate(node):
            new_path = f"{path}[{idx}]"
            walk_json(item, new_path, row_index)
    else:
        table = tables[path]
        while len(table) <= row_index:
            table.append(None)
        table[row_index] = node

for i, record in enumerate(records):
    walk_json(record, row_index=i)

for path, values in tables.items():
    df = pd.DataFrame({path: values})
    print(f"\n=== Table: {path} ===")
    print(df)

=== Table: safetyreportid ===
  safetyreportid
0      5801206-7
1      5801206-8

=== Table: patient.sex ===
  patient.sex
0           M
1           F

=== Table: patient.age ===
   patient.age
0           45
1           38

=== Table: patient.risk_factors.genetic ===
   patient.risk_factors.genetic
0                          True
1                         False

=== Table: patient.risk_factors.family_history.mother ===
   patient.risk_factors.family_history.mother
0                                        True
1                                       False

=== Table: patient.risk_factors.family_history.sister ===
   patient.risk_factors.family_history.sister
0                                       False
1                                        True
```

This approach creates clean, normalized field structures from each nested record.



