1. KYC Step Completion Funnel (with Timing) - User Journeys

[
  {
    "$match": {
      "journeyStatus": { "$in": ["CONVERTED", "IN_PROGRESS", "ABANDONED"] }
    }
  },
  {
    "$project": {
      "steps": {
        "pan":     { "$ifNull": ["$kycSteps.panVerification.completed",    false] },
        "aadhaar": { "$ifNull": ["$kycSteps.aadhaarVerification.completed", false] },
        "bank":    { "$ifNull": ["$kycSteps.bankVerification.completed",   false] },
        "face":    { "$ifNull": ["$kycSteps.faceVerification.completed",   false] },
        "esign":   { "$ifNull": ["$kycSteps.eSign.completed",              false] }
      }
    }
  },
  {
    "$group": {
      "_id": null,
      "total":            { "$sum": 1 },
      "completedPAN":     { "$sum": { "$cond": ["$steps.pan",     1, 0] } },
      "completedAadhaar": { "$sum": { "$cond": ["$steps.aadhaar", 1, 0] } },
      "completedBank":    { "$sum": { "$cond": ["$steps.bank",    1, 0] } },
      "completedFace":    { "$sum": { "$cond": ["$steps.face",    1, 0] } },
      "completedESign":   { "$sum": { "$cond": ["$steps.esign",   1, 0] } }
    }
  },
  {
    "$project": {
      "_id": 0,
      "steps": [
        { "stepOrder": 1, "stepName": "1. Started",          "count": "$total" },
        { "stepOrder": 2, "stepName": "2. PAN Verified",     "count": "$completedPAN" },
        { "stepOrder": 3, "stepName": "3. Aadhaar Verified", "count": "$completedAadhaar" },
        { "stepOrder": 4, "stepName": "4. Bank Verified",    "count": "$completedBank" },
        { "stepOrder": 5, "stepName": "5. Face Verified",    "count": "$completedFace" },
        { "stepOrder": 6, "stepName": "6. E-Sign Done",      "count": "$completedESign" }
      ]
    }
  },
  {
    "$unwind": "$steps"
  },
  {
    "$replaceRoot": { "newRoot": "$steps" }
  },
  {
    "$sort": { "stepOrder": 1 }
  },
  {
    "$project": {
      "stepName": 1,
      "count": 1,
      "_id": 0
    }
  }
]

2. Converted vs Abandoned - Time Spent Comparison - User Journeys

[
  {
    "$match": {
      "journeyStatus": { "$in": ["CONVERTED", "ABANDONED"] }
    }
  },
  {
    "$group": {
      "_id": "$journeyStatus",
      "count": { "$sum": 1 },
      "avgTimeSpent": { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } },
      "maxTimeSpent": { "$max": { "$ifNull": ["$totalTimeSpent", 0] } },
      "minTimeSpent": { "$min": { "$ifNull": ["$totalTimeSpent", 0] } }
    }
  },
  {
    "$addFields": {
      "avgTimeSpent_seconds": {
        "$toLong": { "$add": ["$avgTimeSpent", 0.5] }
      },
      "avgTimeSpent_minutes": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$avgTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      },
      "maxTimeSpent_minutes": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$maxTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      },
      "minTimeSpent_minutes": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$minTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "_id": 1 }
  },
  {
    "$project": {
      "status": "$_id",
      "count": 1,
      "avgTimeSpent_seconds": 1,
      "avgTimeSpent_minutes": 1,
      "maxTimeSpent_minutes": 1,
      "minTimeSpent_minutes": 1,
      "_id": 0
    }
  }
]

3. Time-Spent Distribution (Histogram) - User Journeys

[
  {"$match": {"journeyStatus": {"$in": ["CONVERTED", "ABANDONED"]}, "totalTimeSpent": {"$gt": 0}}},
  {"$addFields": {
    "timeBucket": {"$switch": {
      "branches": [
        {"case": {"$lt": ["$totalTimeSpent", 30]}, "then": "A: <30s (Bounce)"},
        {"case": {"$lt": ["$totalTimeSpent", 120]}, "then": "B: 30s-2m (Glance)"},
        {"case": {"$lt": ["$totalTimeSpent", 300]}, "then": "C: 2-5m (Browse)"},
        {"case": {"$lt": ["$totalTimeSpent", 600]}, "then": "D: 5-10m (Interested)"},
        {"case": {"$lt": ["$totalTimeSpent", 1200]}, "then": "E: 10-20m (Engaged)"},
        {"case": {"$lt": ["$totalTimeSpent", 1800]}, "then": "F: 20-30m (Deep)"}
      ],
      "default": "G: 30m+ (Very Deep)"
    }}
  }},
  {"$group": {
    "_id": {"bucket": "$timeBucket", "status": "$journeyStatus"},
    "count": {"$sum": 1}
  }},
  {"$project": {
    "timeBucket": "$_id.bucket",
    "status": "$_id.status",
    "count": 1,
    "_id": 0
  }},
  {"$sort": {"timeBucket": 1, "status": 1}}
]

4. Device & Network Impact on Conversion - User Journeys

[
  {
    "$match": {
      "journeyStatus": { "$in": ["CONVERTED", "ABANDONED"] }
    }
  },
  {
    "$group": {
      "_id": { "$ifNull": ["$deviceInfo.deviceType", "unknown"] },
      "total":        { "$sum": 1 },
      "converted":    { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } },
      "avgTimeSpent": { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } },
      "avgSteps":     { "$avg": { "$ifNull": ["$stepsCompleted", 0] } },
      "slowNetwork":  { "$sum": { "$cond": [{ "$eq": ["$deviceInfo.slowNetworkDetected", true] }, 1, 0] } }
    }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$total", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      },
      "avgTimeSpent_min": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$avgTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      },
      "avgSteps": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$avgSteps", 10] }, 0.5] } },
          10
        ]
      },
      "slowNetworkPct": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$slowNetwork", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "total": -1 }
  },
  {
    "$project": {
      "deviceType": "$_id",
      "total": 1,
      "converted": 1,
      "conversionRate": 1,
      "avgTimeSpent_min": 1,
      "avgSteps": 1,
      "slowNetworkPct": 1,
      "_id": 0
    }
  }
]

5. Landing Page Conversion Comparison (Detailed) - User Journeys

[
  {
    "$group": {
      "_id": { "$ifNull": ["$landingPage", "UNKNOWN"] },
      "totalVisitors": { "$sum": 1 },
      "started": { "$sum": { "$cond": [{ "$in": ["$journeyStatus", ["IN_PROGRESS", "CONVERTED"]] }, 1, 0] } },
      "converted": { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } },
      "abandoned": { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "ABANDONED"] }, 1, 0] } },
      "avgTimeSpent": { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } },
      "avgSteps": { "$avg": { "$ifNull": ["$stepsCompleted", 0] } }
    }
  },
  {
    "$match": { "totalVisitors": { "$gte": 10 } }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$totalVisitors", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      },
      "startRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$started", { "$max": ["$totalVisitors", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      },
      "avgTimeSpent_min": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$avgTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      },
      "avgSteps": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$avgSteps", 10] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "conversionRate": -1 }
  },
  {
    "$project": {
      "landingPage": "$_id",
      "totalVisitors": 1,
      "started": 1,
      "converted": 1,
      "abandoned": 1,
      "conversionRate": 1,
      "startRate": 1,
      "avgTimeSpent_min": 1,
      "avgSteps": 1,
      "_id": 0
    }
  }
]