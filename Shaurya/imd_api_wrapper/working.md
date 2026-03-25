# WORKING.md
# IMD API Wrapper — Complete Testing Guide

---

## Project Structure

```
imd_api_wrapper/
│
├── api/
│   ├── __init__.py
│   └── main.py
│
├── wrapper/
│   ├── __init__.py
│   ├── config.py
│   ├── api_mapping.py
│   ├── client.py
│   └── router.py
│
├── app.py
├── run_api.py
├── requirements.txt
├── WORKING.md          ← this file
└── README.md
```

---

## Step 1 — Install Dependencies

Paste in JupyterHub:

```python
import sys
!{sys.executable} -m pip install fastapi uvicorn requests pandas matplotlib
```

Expected output:
```
Successfully installed fastapi uvicorn ...
```

---

## Step 2 — Set Project Path

Run this at the start of every session before anything else:

```python
import sys, os
PROJECT_DIR = os.path.join(os.path.expanduser("~"), "imd_api_wrapper")
sys.path.insert(0, PROJECT_DIR)
print("Path set:", PROJECT_DIR)
```

Expected output:
```
Path set: /home/jupyter-shaurya/imd_api_wrapper
```

---

## Step 3 — Verify All Imports

```python
from wrapper.config      import TOTAL_WEATHER_QUERIES
from wrapper.api_mapping import get_full_mapping_table
from wrapper.client      import IMDClient
from wrapper.router      import route_query, route_batch
from wrapper             import IMDClient, route_query

print("All imports OK")
print("Total KCC weather queries:", f"{TOTAL_WEATHER_QUERIES:,}")
```

Expected output:
```
All imports OK
Total KCC weather queries: 15,549,889
```

---

## Step 4 — Verify Config Numbers

```python
from wrapper.config import (
    TOTAL_WEATHER_QUERIES, TOTAL_QUERIES_PER_NEED,
    CLUSTER_QUERY_COUNTS, CLUSTER_TO_NEED,
)

total = sum(CLUSTER_QUERY_COUNTS.values())
print(f"Cluster total    : {total:,}")
print(f"Expected         : 15,549,889")
print(f"Match            : {'YES' if total == 15_549_889 else 'NO'}")
print(f"Clusters defined : {len(CLUSTER_TO_NEED)} (expected 59)")

print("\nPer-need breakdown:")
for need, count in TOTAL_QUERIES_PER_NEED.items():
    pct = count / TOTAL_WEATHER_QUERIES * 100
    print(f"  {need:<35} {count:>12,}  {pct:.2f}%")
```

Expected output:
```
Cluster total    : 15,549,889
Expected         : 15,549,889
Match            : YES
Clusters defined : 59 (expected 59)

Per-need breakdown:
  General Weather Forecast              13,706,092  88.14%
  District Weather Forecast              1,335,570   8.59%
  Rain Forecast                            248,633   1.60%
  Current Weather Condition                180,855   1.16%
  Short Term Forecast                       57,084   0.37%
  Weather Impact on Crops                   21,655   0.14%
```

---

## Step 5 — Test All 6 Client Methods Directly

```python
from wrapper.client import IMDClient

client = IMDClient()

# CRITICAL — General Weather Forecast (88.14% of KCC queries)
r = client.get_city_forecast(city="Delhi", state="Delhi")
print("city_forecast     :", r["priority"], "|", r["temperature_c"], "C")

# HIGH — District Weather Forecast (8.59%)
r = client.get_district_forecast(district="Aligarh", state="Uttar Pradesh")
print("district_forecast :", r["priority"], "|", r["temperature_c"], "C")

# MEDIUM — Rain Forecast (1.60%)
r = client.get_rainfall_forecast(district="Nashik", state="Maharashtra", days=5)
print("rainfall_forecast :", r["priority"], "| days:", r["raw"]["days"])

# MEDIUM — Current Weather Condition (1.16%)
r = client.get_current_weather(city="Mumbai", state="Maharashtra")
print("current_weather   :", r["priority"], "|", r["temperature_c"], "C")

# LOW — Short Term Forecast (0.37%)
r = client.get_nowcast(district="Nagpur", state="Maharashtra")
print("nowcast           :", r["priority"], "|", r["raw"]["nowcast"])

# LOW — Weather Impact on Crops (0.14%)
r = client.get_agromet_advisory(district="Pune", state="Maharashtra", crop="Paddy")
print("agromet_advisory  :", r["priority"], "|", r["alert_type"])
```

Expected output (values vary — mock is random):
```
city_forecast     : CRITICAL | 35.9 C
district_forecast : HIGH | 33.2 C
rainfall_forecast : MEDIUM | days: 5
current_weather   : MEDIUM | 37.7 C
nowcast           : LOW | Light rain likely in next 3 hours
agromet_advisory  : LOW | No Active Warning
```

---

## Step 6 — Test Full Profile (All 6 at Once)

```python
profile = client.get_full_profile(
    city     = "Delhi",
    district = "Aligarh",
    state    = "Uttar Pradesh",
    crop     = "Wheat",
)

for ep, data in profile.items():
    status = "ERROR" if "error" in data else "OK"
    print(f"  {status}  {ep:<28} {data.get('priority','')}")

print(f"\nTotal: {len(profile)}/6 endpoints called")
```

Expected output:
```
  OK  city_forecast            CRITICAL
  OK  district_forecast        HIGH
  OK  rainfall_forecast        MEDIUM
  OK  current_weather          MEDIUM
  OK  nowcast                  LOW
  OK  agromet_advisory         LOW

Total: 6/6 endpoints called
```

---

## Step 7 — Test Query Router

```python
from wrapper.router import route_query

queries = [
    "tell me weather information",
    "weather in aligarh district",
    "will it rain tomorrow",
    "weather forecast for next 5 days",
    "current weather condition today",
    "crop damage due to heavy rainfall",
    "barish kab hogi",
    "frost protection for mustard crop",
    "weather forecast of block tappal in district aligarh",
    "weather report in west midnapur district",
]

for q in queries:
    r = route_query(q)
    print(f"  {r['farmer_need']:<35} ← {q}")
```

Expected output:
```
  General Weather Forecast            ← tell me weather information
  District Weather Forecast           ← weather in aligarh district
  Rain Forecast                       ← will it rain tomorrow
  Short Term Forecast                 ← weather forecast for next 5 days
  Current Weather Condition           ← current weather condition today
  Weather Impact on Crops             ← crop damage due to heavy rainfall
  Rain Forecast                       ← barish kab hogi
  Weather Impact on Crops             ← frost protection for mustard crop
  District Weather Forecast           ← weather forecast of block tappal in district aligarh
  District Weather Forecast           ← weather report in west midnapur district
```

---

## Step 8 — Test API Mapping Functions

```python
from wrapper.api_mapping import (
    get_endpoint_for_cluster,
    get_endpoint_for_need,
    get_full_mapping_table,
    get_need_summary,
)
import pandas as pd

# Single cluster
info = get_endpoint_for_cluster(3)
print("Cluster 3:")
print(f"  Farmer need    : {info['farmer_need']}")
print(f"  Endpoint       : {info['endpoint_key']}")
print(f"  Priority       : {info['priority']}")
print(f"  Cluster queries: {info['cluster_queries']:,}")
print(f"  Coverage       : {info['cluster_coverage_pct']}%")

# Full table
rows = get_full_mapping_table()
df   = pd.DataFrame(rows)
print(f"\nMapping table: {len(df)} rows (expected 59)")
print(f"Total queries: {df['cluster_queries'].sum():,}")
```

Expected output:
```
Cluster 3:
  Farmer need    : General Weather Forecast
  Endpoint       : city_forecast
  Priority       : CRITICAL
  Cluster queries: 6,256,573
  Coverage       : 40.2296%

Mapping table: 59 rows (expected 59)
Total queries: 15,549,889
```

---

## Step 9 — Start the FastAPI Server

```python
import socket, subprocess, sys, time, os

def find_free_port():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(("", 0))
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        return s.getsockname()[1]

PORT        = find_free_port()
PROJECT_DIR = os.path.join(os.path.expanduser("~"), "imd_api_wrapper")

log    = open(os.path.join(PROJECT_DIR, "api_server.log"), "w")
server = subprocess.Popen(
    [sys.executable, "-m", "uvicorn", "api.main:app",
     "--host", "127.0.0.1", "--port", str(PORT)],
    stdout=log, stderr=log, cwd=PROJECT_DIR,
)
time.sleep(4)

if server.poll() is None:
    print(f"Server running  PORT: {PORT}")
    print(f"Swagger UI    : http://127.0.0.1:{PORT}/docs")
else:
    with open(os.path.join(PROJECT_DIR, "api_server.log")) as f:
        print("FAILED:\n", f.read())
```

Expected output:
```
Server running  PORT: 34687
Swagger UI    : http://127.0.0.1:34687/docs
```

> Note: port number will be different every session — always use the printed value.

---

## Step 10 — Verify All API Routes

```python
import requests

BASE   = f"http://127.0.0.1:{PORT}"
routes = [
    ("Health",            "/",                              {}),
    ("City forecast",     "/weather/city-forecast",         {"city":"Delhi","state":"Delhi"}),
    ("District forecast", "/weather/district-forecast",     {"district":"Aligarh","state":"Uttar Pradesh"}),
    ("Rainfall forecast", "/weather/rainfall-forecast",     {"district":"Nashik","state":"Maharashtra","days":5}),
    ("Current weather",   "/weather/current",               {"city":"Mumbai","state":"Maharashtra"}),
    ("Nowcast",           "/weather/nowcast",               {"district":"Nagpur","state":"Maharashtra"}),
    ("Agromet advisory",  "/weather/agromet-advisory",      {"district":"Pune","state":"Maharashtra","crop":"Paddy"}),
    ("Full profile",      "/weather/full-profile",          {"city":"Delhi","district":"Aligarh","state":"Uttar Pradesh"}),
    ("Router single",     "/router/query",                  {"q":"will it rain tomorrow"}),
    ("Cluster 3",         "/analysis/cluster/3",            {}),
    ("All needs",         "/analysis/needs",                {}),
    ("Full mapping",      "/analysis/mapping",              {}),
]

passed = 0
for desc, path, params in routes:
    resp   = requests.get(BASE + path, params=params, timeout=10)
    status = resp.status_code
    ok     = status == 200
    if ok: passed += 1
    print(f"  {'✓' if ok else '✗'}  {desc:<28} {status}")

resp = requests.post(f"{BASE}/router/batch",
                     json=["will it rain","weather today"], timeout=10)
ok   = resp.status_code == 200
if ok: passed += 1
print(f"  {'✓' if ok else '✗'}  {'Batch router POST':<28} {resp.status_code}")

print(f"\n  Result: {passed}/13")

## Step 11 — Sample API Responses

```python
import json

# City forecast full response
r = requests.get(f"{BASE}/weather/city-forecast",
                 params={"city":"Delhi","state":"Delhi"})
print(json.dumps(r.json(), indent=2))

# Router response
r = requests.get(f"{BASE}/router/query",
                 params={"q":"will it rain tomorrow in my district"})
print(json.dumps(r.json(), indent=2))

# Error handling — invalid cluster
r = requests.get(f"{BASE}/analysis/cluster/999")
print(f"Invalid cluster: HTTP {r.status_code}  {r.json()['detail']}")

# Error handling — days out of range
r = requests.get(f"{BASE}/weather/rainfall-forecast",
                 params={"district":"Nashik","days":10})
print(f"days=10 out of range: HTTP {r.status_code}")
```

---

## Step 12 — Stop the Server

```python
server.terminate()
log.close()
print("Server stopped.")
```

---

## Quick Reference — All API Routes

| Method | Route | Parameters | Priority |
|--------|-------|------------|----------|
| GET | `/` | none | — |
| GET | `/health` | none | — |
| GET | `/weather/city-forecast` | `city`, `state` | CRITICAL |
| GET | `/weather/district-forecast` | `district`, `state` | HIGH |
| GET | `/weather/rainfall-forecast` | `district`, `state`, `days` (1-5) | MEDIUM |
| GET | `/weather/current` | `city`, `state` | MEDIUM |
| GET | `/weather/nowcast` | `district`, `state` | LOW |
| GET | `/weather/agromet-advisory` | `district`, `state`, `crop` | LOW |
| GET | `/weather/full-profile` | `city`, `district`, `state`, `crop` | ALL |
| GET | `/router/query` | `q` (query text) | — |
| POST | `/router/batch` | JSON list of strings | — |
| GET | `/analysis/cluster/{id}` | cluster id 0-58 | — |
| GET | `/analysis/need/{need}` | farmer need name | — |
| GET | `/analysis/mapping` | none | — |
| GET | `/analysis/needs` | none | — |

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ModuleNotFoundError: wrapper` | sys.path not set | Re-run Step 2 |
| `Address already in use` | port taken by JupyterHub | Use `find_free_port()` always |
| `404 on all routes` | JupyterHub occupying port 8000 | Never use port 8000 |
| `name 'null' is not defined` | `.py` file saved as notebook JSON | Rewrite file using `open()` from kernel |
| `cannot import name 'IMDClient'` | stale cached module | Clear cache then reimport |
| `FileNotFoundError: uvicorn` | wrong Python environment | Use `sys.executable -m uvicorn` |
| `days validation error` | days not between 1-5 | Pass `days=5` not `days=10` |