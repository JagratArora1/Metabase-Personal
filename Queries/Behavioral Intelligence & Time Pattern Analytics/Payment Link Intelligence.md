1. Payment Link Conversion by Source - Reminder Logs

[
  {
    "$group": {
      "_id": {
        "channel": "$channel",
        "reminderType": "$reminderType"
      },
      "totalSent": { "$sum": 1 },
      "delivered": {
        "$sum": { "$cond": [{ "$eq": ["$status", "sent"] }, 1, 0] }
      },
      "failed": {
        "$sum": { "$cond": [{ "$eq": ["$status", "failed"] }, 1, 0] }
      },
      "errors": {
        "$sum": {
          "$cond": [
            { "$and": [
              { "$ne": ["$error", ""] },
              { "$ne": ["$error", null] }
            ]},
            1, 0
          ]
        }
      }
    }
  },
  {
    "$addFields": {
      "deliveryRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$delivered", { "$max": ["$totalSent", 1] }] }] }, 0.5] } },
          10
        ]
      },
      "failureRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$failed", { "$max": ["$totalSent", 1] }] }] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "_id.channel": 1, "_id.reminderType": 1 }
  },
  {
    "$project": {
      "channel": "$_id.channel",
      "reminderType": "$_id.reminderType",
      "totalSent": 1,
      "delivered": 1,
      "failed": 1,
      "errors": 1,
      "deliveryRate": 1,
      "failureRate": 1,
      "_id": 0
    }
  }
]

2. Payment Link Time-to-Payment Distribution - Payment Links

[
  {
    "$match": { "status": "PAID" }
  },
  {
    "$addFields": {
      "hoursToPay": {
        "$divide": [
          { "$subtract": ["$updatedAt", "$createdAt"] },
          3600000
        ]
      }
    }
  },
  {
    "$addFields": {
      "timeBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$hoursToPay", 1] },   "then": "A: <1 hour" },
            { "case": { "$lt": ["$hoursToPay", 6] },   "then": "B: 1-6 hours" },
            { "case": { "$lt": ["$hoursToPay", 24] },  "then": "C: 6-24 hours" },
            { "case": { "$lt": ["$hoursToPay", 72] },  "then": "D: 1-3 days" },
            { "case": { "$lt": ["$hoursToPay", 168] }, "then": "E: 3-7 days" }
          ],
          "default": "F: 7+ days"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$timeBucket",
      "count": { "$sum": 1 },
      "totalAmount": { "$sum": "$amountPaid" },
      "avgAmount": { "$avg": "$amountPaid" }
    }
  },
  {
    "$addFields": {
      "avgAmount": {
        "$toLong": { "$add": ["$avgAmount", 0.5] }
      }
    }
  },
  {
    "$sort": { "_id": 1 }
  },
  {
    "$project": {
      "timeBucket": "$_id",
      "count": 1,
      "totalAmount": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]