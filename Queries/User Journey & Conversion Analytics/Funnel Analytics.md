1. Average/Minimum/Maximum Time - User Journeys

[
  {"$match": {"journeyStatus": "CONVERTED", "totalTimeSpent": {"$gt": 0}}},
  {"$group": {
    "_id": null,
    "avgTimeMinutes": {"$avg": {"$divide": ["$totalTimeSpent", 60]}},
    "minTimeMinutes": {"$min": {"$divide": ["$totalTimeSpent", 60]}},
    "maxTimeMinutes": {"$max": {"$divide": ["$totalTimeSpent", 60]}},
    "count": {"$sum": 1}
  }},
  {"$project": {"_id": 0}}
]

2. Overall Funnel - Journey Status - User Journeys

[
  {"$group": {
    "_id": "$journeyStatus",
    "count": {"$sum": 1}
  }},
  {"$sort": {"count": -1}},
  {"$project": {"status": "$_id", "count": 1, "_id": 0}}
]

3. Conversion Rate Trend (Weekly) - User Journeys 

[
  {
    "$addFields": {
      "weekStart": {
        "$toDate": {
          "$multiply": [
            {
              "$subtract": [
                {
                  "$toLong": "$createdAt"
                },
                {
                  "$mod": [
                    {
                      "$subtract": [
                        { "$toLong": "$createdAt" },
                        259200000
                      ]
                    },
                    604800000
                  ]
                }
              ]
            },
            1
          ]
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$weekStart",
      "total": { "$sum": 1 },
      "converted": {
        "$sum": {
          "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0]
        }
      }
    }
  },
  { "$sort": { "_id": 1 } },
  {
    "$project": {
      "_id": 0,
      "week": "$_id",
      "total": 1,
      "converted": 1,
      "conversionRate": {
        "$multiply": [
          { "$divide": ["$converted", { "$max": ["$total", 1] }] },
          100
        ]
      }
    }
  }
]

4. Conversion Rate by Landing Page - User Journeys 

[
  {"$group": {
    "_id": "$landingPage",
    "total": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}},
    "abandoned": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "ABANDONED"]}, 1, 0]}}
  }},
  {"$project": {
    "landingPage": "$_id",
    "total": 1,
    "converted": 1,
    "abandoned": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$total", 1]}]}, 100]},
    "abandonRate": {"$multiply": [{"$divide": ["$abandoned", {"$max": ["$total", 1]}]}, 100]},
    "_id": 0
  }},
  {"$sort": {"total": -1}}
]

5. Daily Visitor Volume by Landing Page - User Journeys

[
  {"$group": {
    "_id": {
      "date": {"$dateToString": {"format": "%Y-%m-%d", "date": "$createdAt"}},
      "page": "$landingPage"
    },
    "visitors": {"$sum": 1}
  }},
  {"$sort": {"_id.date": 1}},
  {"$project": {"date": "$_id.date", "landingPage": "$_id.page", "visitors": 1, "_id": 0}}
]

6. KYC Progress Funnel - User Journeys

[
  {
    "$group": {
      "_id": null,
      "totalVisitors": { "$sum": 1 },
      "panAttempted": { "$sum": { "$cond": [{ "$gt": [{ "$ifNull": ["$kycProgress.panAttempts", 0] }, 0] }, 1, 0] } },
      "panVerified": { "$sum": { "$cond": [{ "$eq": ["$kycProgress.panVerified", true] }, 1, 0] } },
      "aadhaarAttempted": { "$sum": { "$cond": [{ "$gt": [{ "$ifNull": ["$kycProgress.aadhaarAttempts", 0] }, 0] }, 1, 0] } },
      "aadhaarVerified": { "$sum": { "$cond": [{ "$eq": ["$kycProgress.aadhaarVerified", true] }, 1, 0] } },
      "selfieVerified": { "$sum": { "$cond": [{ "$eq": ["$kycProgress.selfieVerified", true] }, 1, 0] } },
      "bankVerified": { "$sum": { "$cond": [{ "$eq": ["$kycProgress.bankVerified", true] }, 1, 0] } },
      "esignCompleted": { "$sum": { "$cond": [{ "$eq": ["$kycProgress.esignCompleted", true] }, 1, 0] } }
    }
  },
  {
    "$project": {
      "_id": 0,
      "stages": [
        { "stage": "1. Total Visitors",     "count": "$totalVisitors" },
        { "stage": "2. PAN Attempted",      "count": "$panAttempted" },
        { "stage": "3. PAN Verified",       "count": "$panVerified" },
        { "stage": "4. Aadhaar Attempted",  "count": "$aadhaarAttempted" },
        { "stage": "5. Aadhaar Verified",   "count": "$aadhaarVerified" },
        { "stage": "6. Selfie Verified",    "count": "$selfieVerified" },
        { "stage": "7. Bank Verified",      "count": "$bankVerified" },
        { "stage": "8. eSign Completed",    "count": "$esignCompleted" }
      ]
    }
  },
  { "$unwind": "$stages" },
  {
    "$project": {
      "stage": "$stages.stage",
      "count": "$stages.count"
    }
  }
]