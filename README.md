# Streaming Workshop

This project is a small end-to-end streaming pipeline for learning the basics of event-driven data processing with Python notebooks.

It reads NYC yellow taxi trip data from a public Parquet file, publishes ride events to a Kafka-compatible broker (Redpanda), and consumes those events into PostgreSQL.

## What This Workshop Covers

- Loading batch data with `pandas`
- Converting rows into strongly typed Python objects
- Serializing and publishing events with `kafka-python`
- Consuming events from Kafka/Redpanda
- Writing streamed events into PostgreSQL

## Architecture

```text
NYC taxi parquet file
        |
        v
notebook/producer.ipynb
        |
        v
   Redpanda topic: rides
        |
        v
notebook/consumer.ipynb
        |
        v
 PostgreSQL table: processed_events
```

## Project Structure

```text
.
|-- docker-compose.yml        # Redpanda and PostgreSQL services
|-- pyproject.toml            # Python dependencies
|-- notebook/
|   |-- models.py            # Ride model + serializer/deserializer helpers
|   |-- producer.ipynb       # Produces ride events to topic `rides`
|   `-- consumer.ipynb       # Consumes ride events and inserts into Postgres
|-- producer.ipynb           # Earlier exploratory notebook
`-- main.py                  # Placeholder entry point
```

## Tech Stack

- Python 3.12
- Jupyter Notebook
- Redpanda (Kafka-compatible broker)
- PostgreSQL
- `pandas`
- `kafka-python`
- `psycopg2-binary`
- `pyarrow`

## Data Used

The producer notebook reads this public dataset:

- `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2025-11.parquet`

Only these columns are used:

- `PULocationID`
- `DOLocationID`
- `trip_distance`
- `total_amount`
- `tpep_pickup_datetime`

The notebook loads the first `1000` rows and streams them to Kafka with a small delay between messages.

## Prerequisites

Make sure you have:

- Docker and Docker Compose
- Python 3.12+
- `uv` or another way to install the Python dependencies

## Getting Started

### 1. Start infrastructure

From the project root:

```bash
docker compose up -d
```

This starts:

- Redpanda on `localhost:9092`
- PostgreSQL on `localhost:5432`

### 2. Install Python dependencies

If you use `uv`:

```bash
uv sync
```

If you prefer a virtual environment manually:

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyter kafka-python pandas psycopg2-binary pyarrow
```

### 3. Launch Jupyter

```bash
uv run jupyter notebook
```

If you are not using `uv`:

```bash
jupyter notebook
```

Open the notebooks from the repository root so the local imports resolve cleanly.

## Database Setup

Before running the consumer notebook, create the destination table in PostgreSQL:

```sql
CREATE TABLE IF NOT EXISTS processed_events (
    id SERIAL PRIMARY KEY,
    PULocationID INTEGER NOT NULL,
    DOLocationID INTEGER NOT NULL,
    trip_distance DOUBLE PRECISION NOT NULL,
    total_amount DOUBLE PRECISION NOT NULL,
    pickup_datetime TIMESTAMP NOT NULL
);
```

You can run that SQL with `psql`, a database UI, or from inside the Postgres container. Example:

```bash
docker exec -i $(docker ps -qf "name=postgres") psql -U postgres -d postgres <<'SQL'
CREATE TABLE IF NOT EXISTS processed_events (
    id SERIAL PRIMARY KEY,
    PULocationID INTEGER NOT NULL,
    DOLocationID INTEGER NOT NULL,
    trip_distance DOUBLE PRECISION NOT NULL,
    total_amount DOUBLE PRECISION NOT NULL,
    pickup_datetime TIMESTAMP NOT NULL
);
SQL
```

## Running the Workshop

### Producer notebook

Open [notebook/producer.ipynb](notebook/producer.ipynb) and run the cells in order.

What it does:

- Downloads the taxi dataset
- Converts each row into a `Ride` object
- Serializes each ride as JSON bytes
- Publishes events to the `rides` topic on `localhost:9092`

The serializer logic lives in [notebook/models.py](notebook/models.py).

### Consumer notebook

Open [notebook/consumer.ipynb](notebook/consumer.ipynb) and run the cells in order after the producer has started sending messages.

What it does:

- Connects to the `rides` topic
- Deserializes messages back into `Ride` objects
- Converts pickup time from epoch milliseconds to `datetime`
- Inserts each event into the `processed_events` table

The consumer uses:

- Kafka broker: `localhost:9092`
- Consumer group: `rides-console`
- PostgreSQL database: `postgres`
- PostgreSQL user/password: `postgres` / `postgres`

## Event Schema

Each message represents a ride with this shape:

```json
{
  "PULocationID": 43,
  "DOLocationID": 186,
  "trip_distance": 1.68,
  "total_amount": 22.15,
  "tpep_pickup_datetime": 1761956005000
}
```

The `tpep_pickup_datetime` field is stored as epoch milliseconds in the Kafka message and converted back to a timestamp before inserting into PostgreSQL.

## Verifying It Worked

You can check that rows were written to PostgreSQL:

```bash
docker exec -it $(docker ps -qf "name=postgres") \
  psql -U postgres -d postgres -c "SELECT COUNT(*) FROM processed_events;"
```

You can also inspect a few records:

```bash
docker exec -it $(docker ps -qf "name=postgres") \
  psql -U postgres -d postgres -c "SELECT * FROM processed_events LIMIT 5;"
```

## Notes and Current Gaps

- The current repo is notebook-driven. There is no packaged producer or consumer application yet.
- [main.py](main.py) is only a placeholder.
- The root-level [producer.ipynb](producer.ipynb) appears to be an earlier exploratory version; the notebook workflow in `notebook/` is the more complete path.
- The repository does not currently include automated tests.
- The consumer notebook assumes the `processed_events` table already exists, so that setup must be done manually.

## Suggested Next Improvements

- Add a SQL bootstrap script for the database schema
- Turn the notebooks into reusable Python scripts or services
- Add topic creation and health-check steps
- Add simple tests for serialization/deserialization in [notebook/models.py](notebook/models.py)
- Add a small query notebook or dashboard for inspecting ingested rides
