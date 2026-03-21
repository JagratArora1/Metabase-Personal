1. Browser Distribution - User Journeys

[
  {"$match": {"sessions": {"$exists": true, "$ne": []}}},
  {"$project": {"browser": {"$arrayElemAt": ["$sessions.deviceInfo.browser", 0]}}},
  {"$match": {"browser": {"$ne": null}}},
  {"$group": {"_id": "$browser", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$limit": 10},
  {"$project": {"browser": "$_id", "count": 1, "_id": 0}}
]

2. Referrer Analysis - User Journeys

[
  {"$match": {"referrer": {"$exists": true, "$ne": null, "$ne": ""}}},
  {"$group": {"_id": "$referrer", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$limit": 15},
  {"$project": {"referrer": "$_id", "count": 1, "_id": 0}}
]

3. Top Performing Campaigns - User Journeys

[
  {"$match": {"utm.campaign": {"$exists": true, "$ne": null}}},
  {"$group": {
    "_id": {"source": "$utm.source", "medium": "$utm.medium", "campaign": "$utm.campaign"},
    "visitors": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}},
    "abandoned": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "ABANDONED"]}, 1, 0]}}
  }},
  {"$sort": {"visitors": -1}},
  {"$limit": 20},
  {"$project": {
    "source": "$_id.source",
    "medium": "$_id.medium",
    "campaign": "$_id.campaign",
    "visitors": 1,
    "converted": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$visitors", 1]}]}, 100]},
    "_id": 0
  }}
]

4. Traffic by UTM Medium - User Journeys

[
  {"$match": {"utm.medium": {"$exists": true, "$ne": null}}},
  {"$group": {
    "_id": "$utm.medium",
    "visitors": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}}
  }},
  {"$sort": {"visitors": -1}},
  {"$project": {
    "medium": "$_id",
    "visitors": 1,
    "converted": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$visitors", 1]}]}, 100]},
    "_id": 0
  }}
]

5. Traffic by Entry Source - User Journeys

[
  {"$group": {
    "_id": {"$ifNull": ["$entrySource", "UNKNOWN"]},
    "count": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}}
  }},
  {"$sort": {"count": -1}},
  {"$project": {
    "entrySource": "$_id",
    "count": 1,
    "converted": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$count", 1]}]}, 100]},
    "_id": 0
  }}
]

6. Traffic by UTM Source - User Journeys

[
  {"$match": {"utm.source": {"$exists": true, "$ne": null}}},
  {"$group": {
    "_id": "$utm.source",
    "visitors": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}}
  }},
  {"$sort": {"visitors": -1}},
  {"$limit": 15},
  {"$project": {
    "source": "$_id",
    "visitors": 1,
    "converted": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$visitors", 1]}]}, 100]},
    "_id": 0
  }}
]