1. Overview - Balance Monitors

[
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "monitoringEnabled": {"$sum": {"$cond": [{"$eq": ["$monitoring.enabled", true]}, 1, 0]}},
    "activeConsent": {"$sum": {"$cond": [{"$eq": ["$consent.consentStatus", "ACTIVE"]}, 1, 0]}},
    "revokedConsent": {"$sum": {"$cond": [{"$eq": ["$consent.consentStatus", "REVOKED"]}, 1, 0]}}
  }},
  {"$project": {"_id": 0}}
]

2. Alert Distribution - Balance Monitors

[
  {"$unwind": "$alerts"},
  {"$group": {
    "_id": {"type": "$alerts.type", "severity": "$alerts.severity"},
    "count": {"$sum": 1},
    "acknowledged": {"$sum": {"$cond": [{"$eq": ["$alerts.acknowledged", true]}, 1, 0]}}
  }},
  {"$sort": {"count": -1}},
  {"$project": {
    "alertType": "$_id.type",
    "severity": "$_id.severity",
    "count": 1,
    "acknowledged": 1,
    "_id": 0
  }}
]

3. Total Monitors / Total Fetches / Average Consecutive Failures / Successful Last Fetch  - Balance Monitors

[
  {"$match": {"monitoring.totalFetches": {"$gt": 0}}},
  {"$group": {
    "_id": null,
    "totalMonitors": {"$sum": 1},
    "totalFetches": {"$sum": "$monitoring.totalFetches"},
    "avgConsecutiveFailures": {"$avg": "$monitoring.consecutiveFailures"},
    "successfulLastFetch": {"$sum": {"$cond": [{"$eq": ["$monitoring.lastFetchStatus", "SUCCESS"]}, 1, 0]}}
  }},
  {"$project": {"_id": 0}}
]

4. Consent Status Distribution - Balance Monitors

[
  {"$group": {"_id": "$consent.consentStatus", "count": {"$sum": 1}}},
  {"$project": {"consentStatus": "$_id", "count": 1, "_id": 0}}
]