1. Session Analytics - Sessions

[
  {
    "$group": {
      "_id": { "$dateToString": { "format": "%Y-%m-%d", "date": "$createdAt" } },
      "totalSessions": { "$sum": 1 },
      "activeSessions": {
        "$sum": { "$cond": [{ "$eq": ["$isActive", true] }, 1, 0] }
      },
      "uniqueUsers": { "$addToSet": "$userId" },
      "sessionLengthSum": {
        "$sum": {
          "$divide": [{ "$subtract": ["$expiresAt", "$createdAt"] }, 3600000]
        }
      }
    }
  },
  {
    "$project": {
      "date": "$_id",
      "totalSessions": 1,
      "activeSessions": 1,
      "uniqueUsers": { "$size": "$uniqueUsers" },
      "avgSessionLengthMin": {
        "$let": {
          "vars": {
            "raw": {
              "$multiply": [
                { "$divide": ["$sessionLengthSum", { "$max": ["$totalSessions", 1] }] },
                10
              ]
            }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "_id": 0
    }
  },
  { "$sort": { "date": -1 } }
]

2. IP Whitelist Status - Ip Whitelists

[
  {
    "$project": {
      "ipAddress": 1,
      "isApproved": 1,
      "action": 1,
      "deviceId": 1,
      "addedAt": 1,
      "userId": 1,
      "_id": 0
    }
  },
  { "$sort": { "addedAt": -1 } }
]