Detailed Explanation of the Code
1. Imports
pythonimport os
import json
import pandas as pd
import pymysql
from sqlalchemy import create_engine
from urllib.parse import quote_plus

os: Used to interact with the file system (e.g., listing files in a directory).
json: Used to read and parse JSON files.
pandas: Provides DataFrames for efficient data manipulation and storage.
pymysql: A MySQL client library to establish a direct connection to the database.
sqlalchemy: A SQL toolkit that simplifies database interactions, used here to write DataFrames to MySQL tables.
urllib.parse.quote_plus: Encodes special characters in the database password (e.g., @) to ensure compatibility with SQLAlchemy’s connection string.

Key Concept: Libraries like pandas and sqlalchemy are powerful for data processing and database interactions. quote_plus is essential when passwords contain special characters that might break the connection string.

2. MySQL Connection Parameters
pythonhost = "localhost"
port = 3306
user = "root"
password = "Anton@19801024"
database = "Cricket_analytics"

These variables define the connection details for the MySQL database:

host: The server address (localhost means the database is on the same machine).
port: The MySQL port (3306 is the default).
user: The database username (root is the default admin user in MySQL).
password: The password for the root user. Note the @ character, which needs encoding.
database: The target database name (Cricket_analytics).



Key Concept: Hardcoding credentials like this is convenient for development but insecure for production. In a real application, you’d store credentials in environment variables or a configuration file.

3. Creating the Database
pythonconn = pymysql.connect(host=host, user=user, password=password, port=port)
cursor = conn.cursor()
cursor.execute(f"CREATE DATABASE IF NOT EXISTS {database}")
cursor.close()
conn.close()

Purpose: Ensures the Cricket_analytics database exists before attempting to use it.
pymysql.connect: Establishes a connection to the MySQL server without specifying a database (since it may not exist yet).
cursor.execute: Runs the SQL command CREATE DATABASE IF NOT EXISTS Cricket_analytics, which creates the database if it doesn’t already exist.
Closing connections: The cursor and connection are closed to free up resources.

Key Concept: The IF NOT EXISTS clause prevents errors if the database already exists. Properly closing database connections is a best practice to avoid resource leaks.
Potential Issue: If the root user lacks permissions to create databases, this will fail. Always ensure the user has appropriate privileges.

4. SQLAlchemy Engine
pythonpassword_enc = quote_plus(password)
engine = create_engine(f"mysql+pymysql://{user}:{password_enc}@{host}:{port}/{database}")

quote_plus(password): Encodes the password to handle special characters (e.g., @ becomes %40). This ensures the connection string is valid.
create_engine: Creates a SQLAlchemy engine, which is a high-level interface for interacting with the database. The connection string format is:
textmysql+pymysql://<user>:<password>@<host>:<port>/<database>

The engine will be used later to write pandas DataFrames to MySQL tables.

Key Concept: SQLAlchemy abstracts database operations, making it easier to switch databases (e.g., from MySQL to PostgreSQL) with minimal code changes. Encoding the password is critical for special characters.
Potential Issue: If the database server is not running or the credentials are incorrect, this will raise an error. Always test connectivity before running the script.

5. The process_json_folder Function
This is the core function that processes JSON files and stores data in MySQL. Let’s break it down:
pythondef process_json_folder(json_folder, match_table, delivery_table):
    all_match_info = []
    all_deliveries = []

Parameters:

json_folder: Path to the folder containing JSON files.
match_table: Name of the MySQL table for match information (e.g., ipl_match_info).
delivery_table: Name of the MySQL table for delivery data (e.g., ipl_deliveries).


Lists:

all_match_info: Stores match metadata (e.g., match ID, teams, winner).
all_deliveries: Stores ball-by-ball delivery data (e.g., batter, bowler, runs).



Key Concept: Using lists to collect data before converting to DataFrames is memory-efficient for small to medium datasets.

5.1 Reading JSON Files
pythonfor file_name in os.listdir(json_folder):
    if file_name.endswith(".json"):
        match_id = int(file_name.replace(".json", ""))
        with open(os.path.join(json_folder, file_name), "r") as f:
            data = json.load(f)

os.listdir(json_folder): Lists all files in the specified folder.
Filter: Only processes files ending with .json.
match_id: Extracts the match ID from the filename (e.g., 12345.json → 12345).
json.load: Reads and parses the JSON file into a Python dictionary.

Key Concept: The filename is used as the match_id, assuming it’s a unique identifier. This is a common pattern when filenames encode meaningful data.
Potential Issue: If the filename isn’t numeric or contains unexpected characters, int(file_name.replace(".json", "")) could raise an error. Add error handling (e.g., try-except) for robustness.

5.2 Extracting Match Information
pythonif isinstance(data["meta"], list):
    meta = data["meta"][0]
else:
    meta = data["meta"]
info = data["info"]

Purpose: Extracts metadata and match information from the JSON.
Metadata Handling: Checks if data["meta"] is a list (to handle inconsistent JSON structures) and extracts the first element if it is, otherwise uses data["meta"] directly.
info: Contains match details like city, teams, toss, etc.

Key Concept: Handling inconsistent JSON structures (e.g., meta as a list or dict) shows defensive programming to accommodate variations in data.
Potential Issue: If data["meta"] is missing or empty, this will raise a KeyError. Add checks to handle missing keys.

5.3 Building Match Info Dictionary
pythonmatch_info = {
    "match_id": match_id,
    "data_version": meta.get("data_version"),
    "created": meta.get("created"),
    "revision": meta.get("revision"),
    "city": info.get("city"),
    "event_name": info["event"]["name"] if "event" in info else None,
    "match_number": info["event"].get("match_number") if "event" in info else None,
    "gender": info.get("gender"),
    "match_type": info.get("match_type"),
    "season": info.get("season"),
    "venue": info.get("venue"),
    "toss_winner": info["toss"]["winner"] if "toss" in info else None,
    "toss_decision": info["toss"]["decision"] if "toss" in info else None,
    "winner": info["outcome"].get("winner") if "outcome" in info else None,
    "win_margin_type": list(info["outcome"].get("by", {}).keys())[0]
                       if "outcome" in info and "by" in info["outcome"] else None,
    "win_margin": list(info["outcome"].get("by", {}).values())[0]
                  if "outcome" in info and "by" in info["outcome"] else None,
    "player_of_match": info["player_of_match"][0]
                       if "player_of_match" in info and len(info["player_of_match"]) > 0 else None
}
all_match_info.append(match_info)

Purpose: Creates a dictionary for each match, extracting fields like match ID, city, toss details, winner, etc.
Safe Access:

Uses .get() to safely access dictionary keys, returning None if the key is missing.
Conditional checks (e.g., if "event" in info) prevent errors for missing nested keys.
For win_margin_type and win_margin, it extracts the first key/value from the by dictionary (e.g., {"runs": 10} → runs and 10).


Appends: Adds the match_info dictionary to the all_match_info list.

Key Concept: Using .get() and conditional checks is a best practice for handling potentially missing data in JSON files.
Potential Issue: Assumes info["player_of_match"] is a list with at least one element. If it’s empty or missing, this could fail. The existing check mitigates this, but further validation could ensure robustness.

5.4 Processing Deliveries
pythonfor inning in data["innings"]:
    inning_team = inning["team"]
    for over in inning["overs"]:
        over_no = over["over"]
        for ball_no, delivery in enumerate(over["deliveries"], start=1):
            row = {
                "match_id": match_id,
                "inning_team": inning_team,
                "over_number": over_no,
                "ball_number": ball_no,
                "batter": delivery["batter"],
                "bowler": delivery["bowler"],
                "non_striker": delivery["non_striker"],
                "runs_batter": delivery["runs"]["batter"],
                "runs_extras": delivery["runs"]["extras"],
                "runs_total": delivery["runs"]["total"],
                "extras_type": None,
                "wicket_player_out": None,
                "wicket_kind": None,
                "umpire_review": None
            }
            if "extras" in delivery:
                row["extras_type"] = list(delivery["extras"].keys())[0]
            if "wickets" in delivery:
                row["wicket_player_out"] = delivery["wickets"][0]["player_out"]
                row["wicket_kind"] = delivery["wickets"][0]["kind"]
            if "review" in delivery:
                row["umpire_review"] = delivery["review"]["decision"]
            all_deliveries.append(row)

Purpose: Extracts ball-by-ball data for each delivery in the match.
Nested Loops:

Iterates over innings (e.g., first and second innings).
Iterates over overs within each inning.
Iterates over deliveries within each over, using enumerate to generate ball numbers (starting from 1).


Data Extracted:

Core fields: match_id, inning_team, over_number, ball_number, batter, bowler, non_striker, runs_batter, runs_extras, runs_total.
Optional fields: extras_type (e.g., wide, no-ball), wicket_player_out, wicket_kind (e.g., bowled, caught), umpire_review (e.g., umpire review decision).


Safe Access: Checks for the presence of extras, wickets, and review before accessing nested keys.

Key Concept: The nested loop structure mirrors the hierarchical JSON data (innings → overs → deliveries). Using enumerate for ball numbers is a clean way to generate sequential indices.
Potential Issue: Assumes the first element of extras or wickets is sufficient. If multiple extras or wickets occur in a single delivery (rare in cricket), this would miss data. Validate the JSON structure to confirm assumptions.

5.5 Creating DataFrames and Writing to MySQL
pythondf_match_info = pd.DataFrame(all_match_info)
df_deliveries = pd.DataFrame(all_deliveries)
df_match_info.to_sql(match_table, engine, if_exists="replace", index=False)
df_deliveries.to_sql(delivery_table, engine, if_exists="replace", index=False)
print(f"✅ Inserted into MySQL tables: {match_table} & {delivery_table} (DB: {database})")

pd.DataFrame: Converts the lists of dictionaries (all_match_info, all_deliveries) into pandas DataFrames.
to_sql:

Writes the DataFrames to the specified MySQL tables (match_table, delivery_table).
if_exists="replace": Drops and recreates the table if it already exists.
index=False: Excludes the DataFrame index from being written to the database.


Print Statement: Confirms successful insertion with table and database names.

Key Concept: Pandas’ to_sql with SQLAlchemy simplifies database writes. The if_exists="replace" option is useful for development but risky in production, as it deletes existing data.
Potential Issue: Replacing tables can lead to data loss. For production, consider if_exists="append" or a more sophisticated upsert mechanism. Also, large datasets may require batch processing to avoid memory issues.

6. Running the Function for Different Formats
pythonipl_folder = r"C:\Users\ANTON\Documents\VS Code\Project-02\Data\ipljson"
process_json_folder(ipl_folder, "ipl_match_info", "ipl_deliveries")

odi_folder = r"C:\Users\ANTON\Documents\VS Code\Project-02\Data\odis_json\01"
process_json_folder(odi_folder, "odi_match_info", "odi_deliveries")

t20_folder = r"C:\Users\ANTON\Documents\VS Code\Project-02\Data\t20s_json\01"
process_json_folder(t20_folder, "t20_match_info", "t20_deliveries")

test_folder = r"C:\Users\ANTON\Documents\VS Code\Project-02\Data\tests_json"
process_json_folder(test_folder, "test_match_info", "test_deliveries")

Purpose: Calls process_json_folder for four different cricket formats (IPL, ODI, T20, Test), each with its own JSON folder and table names.
Folder Paths: Hardcoded paths to JSON files (e.g., C:\Users\ANTON\...).
Table Names: Separate tables for each format (e.g., ipl_match_info, odi_deliveries) to keep data organized.

Key Concept: Separating data by format allows for format-specific analysis while maintaining a consistent processing logic.
Potential Issue: Hardcoded file paths are brittle and won’t work on other machines. Use relative paths or configuration files for portability. Also, ensure the folders exist and contain valid JSON files to avoid errors.

Key Programming Concepts

File Handling: Using os.listdir and json.load to process files.
Data Transformation: Converting JSON to DataFrames for structured storage.
Database Interaction: Using pymysql for raw SQL and sqlalchemy for high-level operations.
Error Handling: Using .get() and conditional checks to handle missing JSON keys.
Modularity: The process_json_folder function is reusable across different data formats.


Potential Improvements

Error Handling:

Add try-except blocks for file reading, JSON parsing, and database operations.
Validate JSON structure before processing to handle malformed data.


Logging: Replace print with a logging framework to track progress and errors.
Configuration: Store database credentials and file paths in a configuration file or environment variables.
Schema Definition: Define table schemas explicitly in MySQL (e.g., with appropriate data types and constraints) instead of relying on to_sql’s automatic schema generation.
Batch Processing: For large datasets, process JSON files in batches to reduce memory usage.
Data Validation: Check for duplicate match_ids or missing critical fields before insertion.
Performance: Use SQLAlchemy’s fast_executemany for faster inserts with large datasets.


Example JSON Structure (Assumed)
Based on the code, the JSON files likely have a structure like:
json{
  "meta": {
    "data_version": "1.0",
    "created": "2023-01-01",
    "revision": 1
  },
  "info": {
    "city": "Mumbai",
    "event": {"name": "IPL 2023", "match_number": 1},
    "gender": "male",
    "match_type": "T20",
    "season": "2023",
    "venue": "Wankhede Stadium",
    "toss": {"winner": "Team A", "decision": "bat"},
    "outcome": {"winner": "Team A", "by": {"runs": 10}},
    "player_of_match": ["Player X"]
  },
  "innings": [
    {
      "team": "Team A",
      "overs": [
        {
          "over": 0,
          "deliveries": [
            {
              "batter": "Player X",
              "bowler": "Player Y",
              "non_striker": "Player Z",
              "runs": {"batter": 4, "extras": 0, "total": 4}
            },
            {
              "batter": "Player X",
              "bowler": "Player Y",
              "non_striker": "Player Z",
              "runs": {"batter": 0, "extras": 1, "total": 1},
              "extras": {"wide": 1}
            }
          ]
        }
      ]
    }
  ]
}

Why This Matters
This script is a practical example of ETL (Extract, Transform, Load):

Extract: Reading JSON files.
Transform: Converting JSON data into structured DataFrames.
Load: Writing DataFrames to a MySQL database.

It’s useful for cricket analytics, enabling queries like:

Who scored the most runs in IPL 2023?
What’s the average run rate in T20 matches?
How often do teams batting first win in Test matches?
