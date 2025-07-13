# FDA JSON to Relational DB

This project transforms deeply nested, non-relational JSON data into a clean set of relational tables suitable for PostgreSQL.

It is designed to help analysts and engineers handle structured medical records or event reporting data — especially when stored in formats that are difficult to query directly.

> **Note**: This project uses *publicly available data* from the [openFDA API](https://api.fda.gov/). The data is de-identified and intended for public use. This project is for demonstration purposes only and should not be used for clinical or regulatory decision-making.

## Features

- ✅ Flattens complex nested JSON (including arrays and objects)
- 🧱 Creates normalized tables for clean relational storage
- 🔗 Maintains shared keys for joining across tables (e.g., `safetyreportid`)
- 💾 Loads directly into a PostgreSQL database via SQLAlchemy
- 🛠️ Easy to modify or expand for other JSON schemas

## Technologies Used

- Python 3.x
- pandas
- SQLAlchemy
- PostgreSQL

## Example Output

This tool extracts and loads four tables:

- `logistics` — top-level report metadata
- `patient` — patient-level attributes
- `drugs` — medications involved
- `reactions` — recorded adverse reactions

Each table includes a shared `safetyreportid` to allow easy joins.

## Setup

1. Clone the repository  
2. Install dependencies:
   ```bash
   pip install pandas sqlalchemy psycopg2-binary
