# JSON to Relational Table Pipeline

## Overview

This script demonstrates how to transform nested JSON data from the [OpenFDA API](https://api.fda.gov/drug/event.json) into clean, normalized relational tables and load them into a PostgreSQL database.

The data is separated into logical entities: `logistics`, `patient`, `drugs`, and `reactions`. This makes it easier to query and analyze real-world pharmacovigilance data.

---

## Core Concept

> **"Flatten the JSON into a tabular format. It's just a few lines of code to implement. Assuming the relationships are known, you can extract from the flattened tabular format and insert into the relational tables."**

This pipeline operates on this principle: once JSON is flattened, known relationships between data entities can be used to split the flat structure into meaningful relational tables. This is especially helpful when working with complex API responses in a structured database setting.

---

## Steps

### 1. Fetch Data from OpenFDA

We call the OpenFDA drug event API and isolate one record for inspection:

```python
url = "https://api.fda.gov/drug/event.json?limit=1"
response = requests.get(url)
record = response.json()["results"][0]
data = [record]
```

### 2. Flatten the JSON

We use `pandas.json_normalize()` to flatten nested JSON keys into dot-separated column names for easy parsing:

```python
flat_df = json_normalize(record, sep='.')
```

### 3. Build Relational Tables

We construct separate DataFrames from the flattened data. For example:

```python
logistics_df = flat_df[[
    'safetyreportid', 'transmissiondateformat', 'transmissiondate',
    'serious', 'seriousnessdeath', 'receivedateformat', 'receivedate',
    'receiptdateformat', 'receiptdate', 'fulfillexpeditecriteria',
    'companynumb', 'receiver', 'primarysource.reportercountry',
    'primarysource.qualification', 'sender.senderorganization'
]]
logistics_df.columns = [col.split('.')[-1] for col in logistics_df.columns]

patient_df = flat_df[[
    'safetyreportid',
    'patient.patientsex',
    'patient.patientonsetage',
    'patient.patientonsetageunit',
    'patient.patientdeath.patientdeathdateformat',
    'patient.patientdeath.patientdeathdate'
]]
patient_df.columns = [
    'safetyreportid', 'sex', 'onsetage', 'onsetageunit',
    'death_date_format', 'death_date'
]

drugs_df = json_normalize(
    data,
    record_path=['patient', 'drug'],
    meta=['safetyreportid']
)

reactions_df = json_normalize(
    data,
    record_path=['patient', 'reaction'],
    meta=['safetyreportid']
)
```

### 4. 4. Load into PostgreSQL

Each relational table is written into the PostgreSQL database:

```python
engine = create_engine("postgresql://postgres:password@localhost:5432/fda_relational")

logistics_df.to_sql("logistics", engine, if_exists="append", index=False)
patient_df.to_sql("patient", engine, if_exists="append", index=False)
drugs_df.to_sql("drugs", engine, if_exists="append", index=False)
reactions_df.to_sql("reactions", engine, if_exists="append", index=False)
```

### Result:

This approach results in a scalable, relational schema that reflects the original JSON structure, while making querying more performant and organized. Each record maintains referential integrity via `safetyreportid`.
