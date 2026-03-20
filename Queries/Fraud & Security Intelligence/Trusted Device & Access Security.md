1. Device Trust Score Distribution - Trusted Devices

[
  {
    "$addFields": {
      "trustBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lte": [{ "$ifNull": ["$trustScore", 0] }, 25] }, "then": "Low Trust (0-25)" },
            { "case": { "$lte": [{ "$ifNull": ["$trustScore", 0] }, 50] }, "then": "Medium Trust (26-50)" },
            { "case": { "$lte": [{ "$ifNull": ["$trustScore", 0] }, 75] }, "then": "High Trust (51-75)" },
            { "case": { "$lte": [{ "$ifNull": ["$trustScore", 0] }, 100] }, "then": "Very High Trust (76-100)" }
          ],
          "default": "Unscored"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$trustBucket",
      "devices": { "$sum": 1 },
      "approved": { "$sum": { "$cond": [{ "$eq": ["$status", "approved"] }, 1, 0] } },
      "rooted": { "$sum": { "$cond": [{ "$eq": ["$isRooted", true] }, 1, 0] } },
      "emulator": { "$sum": { "$cond": [{ "$eq": ["$isEmulator", true] }, 1, 0] } }
    }
  },
  {
    "$project": {
      "trustLevel": "$_id",
      "devices": 1,
      "approved": 1,
      "rooted": 1,
      "emulator": 1,
      "approvedRate": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$approved", { "$max": ["$devices", 1] }] }, 1000] }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "_id": 0
    }
  },
  { "$sort": { "trustLevel": 1 } }
]

2. Trust Score by Device Type - Trusted Devices

[
  {
    "$group": {
      "_id": { "$ifNull": ["$deviceType", "Unknown"] },
      "count": { "$sum": 1 },
      "avgTrustSum": { "$sum": { "$ifNull": ["$trustScore", 0] } },
      "maxTrust": { "$max": "$trustScore" },
      "minTrust": { "$min": "$trustScore" }
    }
  },
  {
    "$project": {
      "deviceType": "$_id",
      "count": 1,
      "maxTrust": 1,
      "minTrust": 1,
      "avgTrustScore": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$avgTrustSum", { "$max": ["$count", 1] }] }, 10] }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "_id": 0
    }
  },
  { "$sort": { "count": -1 } }
]

3. Device Type Distribution - Trusted Devices

[
  {
    "$group": {
      "_id": { "$ifNull": ["$deviceType", "Unknown"] },
      "count": { "$sum": 1 },
      "approved": { "$sum": { "$cond": [{ "$eq": ["$status", "approved"] }, 1, 0] } },
      "rooted": { "$sum": { "$cond": [{ "$eq": ["$isRooted", true] }, 1, 0] } },
      "emulator": { "$sum": { "$cond": [{ "$eq": ["$isEmulator", true] }, 1, 0] } }
    }
  },
  {
    "$project": {
      "deviceType": "$_id",
      "count": 1,
      "approved": 1,
      "rooted": 1,
      "emulator": 1,
      "_id": 0
    }
  },
  { "$sort": { "count": -1 } }
]