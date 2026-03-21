1. Entry Source vs Loan Disbursement (End-to-End Attribution) - User Journeys

[
  {
    "$match": {
      "entrySource": { "$exists": true, "$ne": null }
    }
  },
  {
    "$group": {
      "_id": "$entrySource",
      "totalJourneys": { "$sum": 1 },
      "converted":     { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } },
      "avgTimeSpent":  { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } }
    }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$totalJourneys", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      },
      "avgEngagement_min": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$avgTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "totalJourneys": -1 }
  },
  {
    "$group": {
      "_id": null,
      "sources": {
        "$push": {
          "entrySource":      "$_id",
          "totalJourneys":    "$totalJourneys",
          "converted":        "$converted",
          "conversionRate":   "$conversionRate",
          "avgEngagement_min": "$avgEngagement_min"
        }
      }
    }
  },
  {
    "$unwind": {
      "path": "$sources",
      "includeArrayIndex": "stepOrder"
    }
  },
  {
    "$addFields": {
      "stepName": {
        "$concat": [
          { "$toString": { "$add": ["$stepOrder", 1] } },
          ". ",
          "$sources.entrySource"
        ]
      }
    }
  },
  {
    "$project": {
      "stepName":         1,
      "entrySource":      "$sources.entrySource",
      "totalJourneys":    "$sources.totalJourneys",
      "converted":        "$sources.converted",
      "conversionRate":   "$sources.conversionRate",
      "avgEngagement_min": "$sources.avgEngagement_min",
      "_id": 0
    }
  }
]

2. Referrer Domain Analysis - User Journeys

[
  {
    "$match": {
      "referrer": {
        "$exists": true,
        "$nin": [null, ""]
      }
    }
  },
  {
    "$addFields": {
      "referrerDomain": {
        "$arrayElemAt": [
          { "$split": [
            { "$arrayElemAt": [
              { "$split": ["$referrer", "//"] },
              1
            ]},
            "/"
          ]},
          0
        ]
      }
    }
  },
  {
    "$group": {
      "_id": "$referrerDomain",
      "visitors":  { "$sum": 1 },
      "converted": { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } }
    }
  },
  {
    "$match": { "visitors": { "$gte": 3 } }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$visitors", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      }
    }
  },
  {
    "$sort": { "visitors": -1 }
  },
  {
    "$limit": 20
  },
  {
    "$project": {
      "referrerDomain": "$_id",
      "visitors":       1,
      "converted":      1,
      "conversionRate": 1,
      "_id": 0
    }
  }
]

3. UTM Source Performance - Full Funnel - User Journeys

[
  {
    "$match": {
      "utmData.source": { "$exists": true, "$ne": null }
    }
  },
  {
    "$group": {
      "_id": "$utmData.source",
      "visitors":     { "$sum": 1 },
      "started":      { "$sum": { "$cond": [{ "$ne": ["$journeyStatus", "ABANDONED"] }, 1, 0] } },
      "converted":    { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } },
      "avgTimeSpent": { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } },
      "avgSteps":     { "$avg": { "$ifNull": ["$stepsCompleted", 0] } }
    }
  },
  {
    "$match": { "visitors": { "$gte": 5 } }
  },
  {
    "$addFields": {
      "startRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$started",   { "$max": ["$visitors", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$visitors", 1] }] }, 10000] }, 0.5] } },
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
      "source":          "$_id",
      "visitors":        1,
      "started":         1,
      "converted":       1,
      "startRate":       1,
      "conversionRate":  1,
      "avgTimeSpent_min": 1,
      "avgSteps":        1,
      "_id": 0
    }
  }
]

4. UTM Campaign ROI Tracker - User Journeys

[
  {
    "$match": {
      "utmData.campaign": { "$exists": true, "$ne": null }
    }
  },
  {
    "$group": {
      "_id": {
        "source":   { "$ifNull": ["$utmData.source",   "direct"] },
        "medium":   { "$ifNull": ["$utmData.medium",   "none"] },
        "campaign": "$utmData.campaign"
      },
      "visitors":     { "$sum": 1 },
      "converted":    { "$sum": { "$cond": [{ "$eq": ["$journeyStatus", "CONVERTED"] }, 1, 0] } },
      "avgTimeSpent": { "$avg": { "$ifNull": ["$totalTimeSpent", 0] } }
    }
  },
  {
    "$match": { "visitors": { "$gte": 3 } }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$visitors", 1] }] }, 10000] }, 0.5] } },
          100
        ]
      },
      "avgTimeSpent_min": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$avgTimeSpent", 60] }, 10] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "converted": -1 }
  },
  {
    "$project": {
      "source":           "$_id.source",
      "medium":           "$_id.medium",
      "campaign":         "$_id.campaign",
      "visitors":         1,
      "converted":        1,
      "conversionRate":   1,
      "avgTimeSpent_min": 1,
      "_id": 0
    }
  }
]