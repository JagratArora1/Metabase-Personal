1. Audit Log Volume Summary - Audit Logs

[
  {"$match": {"timestamp": {"$gte": {"$date": "2026-02-01T00:00:00Z"}}}},
  {"$group": {
    "_id": null,
    "totalLogs": {"$sum": 1}
  }},
  {"$project": {"_id": 0}}
]

2. Action Type Distribution - Audit Logs

[
  {"$match": {"timestamp": {"$gte": {"$date": "2026-02-01T00:00:00Z"}}}},
  {"$group": {"_id": "$action", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"action": "$_id", "count": 1, "_id": 0}}
]

3. Daily API Request Volume - Audit Logs

[
  {"$match": {"timestamp": {"$gte": {"$date": "2026-02-01T00:00:00Z"}}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m-%d", "date": "$timestamp"}},
    "totalRequests": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"date": "$_id", "totalRequests": 1, "_id": 0}}
]

4. Top Endpoints by Volume - Audit Logs

[
  {"$match": {"timestamp": {"$gte": {"$date": "2026-02-01T00:00:00Z"}}}},
  {"$group": {"_id": "$url", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$limit": 20},
  {"$project": {"endpoint": "$_id", "requests": "$count", "_id": 0}}
]

5. Error Rate by Endpoint - Audit Logs 

[
  {"$match": {"timestamp": {"$gte": {"$date": "2026-02-01T00:00:00Z"}}}},
  {"$group": {
    "_id": "$url",
    "total": {"$sum": 1},
    "errors": {"$sum": {"$cond": [{"$in": ["$status", ["FAILED", "ERROR"]]}, 1, 0]}},
    "successes": {"$sum": {"$cond": [{"$eq": ["$status", "SUCCESS"]}, 1, 0]}}
  }},
  {"$match": {"total": {"$gte": 10}}},
  {"$project": {
    "endpoint": "$_id",
    "total": 1,
    "errors": 1,
    "errorRate": {"$multiply": [{"$divide": ["$errors", "$total"]}, 100]},
    "_id": 0
  }},
  {"$sort": {"errorRate": -1}},
  {"$limit": 30}
]