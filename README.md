# 🗓️ Conflict-Free Exam Timetable Generator

An applied graph theory project that generates a conflict-free university exam schedule from real student enrollment data, then persists the resulting course graph to a **Neo4j** graph database for querying and exploration.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)
![NetworkX](https://img.shields.io/badge/NetworkX-Graph%20Theory-orange)
![Neo4j](https://img.shields.io/badge/Neo4j-Graph%20Database-green?logo=neo4j)

---

## Overview

Scheduling exams without conflicts (i.e., no student sitting two exams at the same time) is a classic **graph coloring** problem. This notebook models it as follows:

- Each **course** becomes a **node** in a graph.
- Two courses are connected by an **edge** if at least one student is enrolled in both. The edge **weight** is the number of shared students.
- A **graph coloring algorithm** then assigns a "color" (exam slot) to each node, guaranteeing that no two connected nodes share the same color — meaning no student ever has two exams in the same period.

The final schedule and the course-conflict graph are both exported: the schedule to Excel, the graph to Neo4j.

---

## How It Works

### 1. Data Loading & Cleaning

Enrollment data for five faculties is loaded from separate sheets of an Excel file and merged into a single DataFrame after handling nulls, duplicates, and type inconsistencies:

| Faculty                 | Sheet           |
| ----------------------- | --------------- |
| Medical Sciences        | علوم طبية       |
| Pharmacy & Dentistry    | صيدلة وطب اسنان |
| Business Administration | العلوم الادارية |
| Engineering             | الهندسة         |
| Computer Science        | حاسب الي        |

### 2. Graph Construction

```
For every student:
    For every pair of courses they are enrolled in:
        Add an edge between those two course nodes
        (or increment its weight if it already exists)
```

Node size in the visualisation = number of enrolled students.  
Node colour = degree (number of courses it conflicts with).

### 3. Graph Coloring → Exam Slots

Two greedy coloring strategies from NetworkX are compared:

| Strategy        | Speed          | Colors Used | Notes             |
| --------------- | -------------- | ----------- | ----------------- |
| `largest_first` | Fast, one pass | More        | Deterministic     |
| `DSATUR`        | Slower         | Fewer       | Non-deterministic |

Each color maps to a distinct exam period. Periods are then packed into days respecting two constraints:

- max **5 periods per day**
- max **2 000 student-sittings per day**

### 4. Neo4j Export

The full course-conflict graph (nodes + weighted edges) is written to a local Neo4j instance so it can be explored with Cypher queries or visualised in Neo4j Bloom.

### 5. Schedule Export

Both schedules (one per coloring strategy) are saved as `Exam_Schedule.xlsx` and `Exam_Schedule2.xlsx`.

---

## Project Structure

```
.
├── timetabledesign.ipynb   # Main notebook
├── requirements.txt
├── .env                    # Neo4j credentials (git-ignored)
├── .gitignore
├── data/
│   └── studentcourses.xlsx # Enrollment data (one sheet per faculty)
├── Exam_Schedule.xlsx      # Output: largest_first schedule
└── Exam_Schedule2.xlsx     # Output: DSATUR schedule
```

---

## Getting Started

### 1. Prerequisites

- Python 3.10+
- [Neo4j Desktop](https://neo4j.com/download/) installed and a local database running on the default port (`7687`)

### 2. Clone & install dependencies

```bash
git clone https://github.com/<your-username>/timetable-graph.git
cd timetable-graph

python -m venv venv
source venv/bin/activate       # macOS / Linux
venv\Scripts\activate          # Windows

pip install -r requirements.txt
```

### 3. Configure Neo4j credentials

Create a `.env` file in the project root:

```env
URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password_here
```

> ⚠️ Make sure your Neo4j database is **running** before executing the connection cell.

### 4. Add the enrollment data

Place `studentcourses.xlsx` inside a `data/` folder in the project root. The notebook expects five sheets named as listed in the table above.

### 5. Run the notebook

```bash
jupyter notebook timetabledesign.ipynb
```

Run all cells top to bottom. The Neo4j cells will confirm connectivity before writing data, and the two schedule files will be generated in the project root.

---

## Tech Stack

| Tool                    | Purpose                                  |
| ----------------------- | ---------------------------------------- |
| `pandas`                | Data loading, cleaning, merging          |
| `networkx`              | Graph construction & coloring algorithms |
| `matplotlib`            | Graph visualisation                      |
| `neo4j` (Python driver) | Writing the graph to Neo4j               |
| `openpyxl`              | Exporting schedules to Excel             |
| `python-dotenv`         | Loading Neo4j credentials from `.env`    |

---

## Sample Cypher Queries (Neo4j)

Once the graph is loaded, try these in Neo4j Browser:

```cypher
-- Find all courses that conflict with a specific course
MATCH (a:course {id: 110102})-[r:RELATED_TO]-(b:course)
RETURN b.name, r.weight AS shared_students
ORDER BY shared_students DESC;

-- Find the most conflicted course (highest degree)
MATCH (c:course)-[:RELATED_TO]-()
RETURN c.id, c.name, count(*) AS conflicts
ORDER BY conflicts DESC
LIMIT 10;

-- Find courses with the most shared students on a single edge
MATCH (a:course)-[r:RELATED_TO]-(b:course)
RETURN a.name, b.name, r.weight
ORDER BY r.weight DESC
LIMIT 10;
```
