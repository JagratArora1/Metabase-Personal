1. Blacklist Summary by Type - Blacklists

[
  {
    "$group": {
      "_id": "$type",
      "count": { "$sum": 1 },
      "active": { "$sum": { "$cond": [{ "$eq": ["$status", "ACTIVE"] }, 1, 0] } },
      "expired": {
        "$sum": {
          "$cond": [{ "$eq": ["$status", "EXPIRED"] }, 1, 0]
        }
      }
    }
  },
  {
    "$project": {
      "type": "$_id",
      "total": "$count",
      "active": 1,
      "expired": 1,
      "activeRate": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$active", { "$max": ["$count", 1] }] }, 100] }
          },
          "in": {
            "$divide": [
              { "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] },
              1
            ]
          }
        }
      },
      "_id": 0
    }
  },
  { "$sort": { "total": -1 } }
]

2. Blacklist Reasons Distribution - Blacklists

[
  {"$match": {"reason": {"$exists": true, "$ne": null}}},
  {"$group": {
    "_id": "$reason",
    "count": {"$sum": 1}
  }},
  {"$project": {
    "reason": "$_id",
    "count": 1,
    "_id": 0
  }},
  {"$sort": {"count": -1}},
  {"$limit": 20}
]

3. Blacklist Trend Over Time - Blacklists

[
  {"$match": {"createdAt": {"$exists": true}}},
  {"$group": {
    "_id": {
      "month": {"$dateToString": {"format": "%Y-%m", "date": "$createdAt"}},
      "type": "$type"
    },
    "count": {"$sum": 1}
  }},
  {"$project": {
    "month": "$_id.month",
    "type": "$_id.type",
    "count": 1,
    "_id": 0
  }},
  {"$sort": {"month": 1, "type": 1}}
]

4. Blacklisted Entities Attempting Re-entry - Customers

[
  {
    "$match": {
      "email": { "$ne": null, "$ne": "" }
    }
  },
  {
    "$lookup": {
      "from": "blacklists",
      "localField": "email",
      "foreignField": "value",
      "as": "blacklistHits"
    }
  },
  {
    "$addFields": {
      "blacklistHits": {
        "$filter": {
          "input": "$blacklistHits",
          "as": "hit",
          "cond": { "$eq": ["$$hit.isActive", true] }
        }
      }
    }
  },
  { "$match": { "blacklistHits": { "$ne": [] } } },
  {
    "$lookup": {
      "from": "applications",
      "localField": "_id",
      "foreignField": "customerId",
      "as": "applications"
    }
  },
  { "$match": { "applications": { "$ne": [] } } },
  { "$unwind": "$applications" },
  {
    "$project": {
      "applicationId": "$applications._id",
      "applicationNumber": "$applications.applicationNumber",
      "customerName": {
        "$concat": [
          { "$ifNull": ["$firstName", ""] },
          " ",
          { "$ifNull": ["$lastName", ""] }
        ]
      },
      "email": 1,
      "applicationDate": "$applications.appliedDate",
      "applicationStatus": "$applications.status",
      "requestedAmount": "$applications.requestedLoanAmount",
      "blacklistedOn": { "$arrayElemAt": ["$blacklistHits.createdAt", 0] },
      "blacklistReason": { "$arrayElemAt": ["$blacklistHits.reason", 0] },
      "blacklistType": { "$arrayElemAt": ["$blacklistHits.type", 0] },
      "loanNumber": { "$arrayElemAt": ["$blacklistHits.metadata.loanNumber", 0] },
      "dpd": { "$arrayElemAt": ["$blacklistHits.metadata.dpd", 0] },
      "isReEntry": {
        "$cond": [
          {
            "$gt": [
              "$applications.appliedDate",
              { "$arrayElemAt": ["$blacklistHits.createdAt", 0] }
            ]
          },
          "YES",
          "NO"
        ]
      },
      "_id": 0
    }
  },
  { "$sort": { "applicationDate": -1 } },
  { "$limit": 50 }
]

