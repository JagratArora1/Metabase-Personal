1. Velocity Check - Rapid Applications from Same IP/Device - User Journeys

[
  {
    "$unwind": "$sessions"
  },
  {
    "$match": {
      "sessions.location.ip": { "$exists": true, "$ne": null, "$ne": "" }
    }
  },
  {
    "$group": {
      "_id": "$sessions.location.ip",
      "journeyCount": { "$sum": 1 },
      "uniqueVisitors": { "$addToSet": "$visitorId" },
      "firstSeen": { "$min": "$sessions.startTime" },
      "lastSeen": { "$max": "$sessions.startTime" },
      "landingPages": { "$addToSet": "$landingPage" },
      "suspiciousCount": { "$sum": { "$cond": [{ "$eq": ["$isSuspicious", true] }, 1, 0] } },
      "botCount": { "$sum": { "$cond": [{ "$eq": ["$isBot", true] }, 1, 0] } }
    }
  },
  {
    "$addFields": {
      "uniqueVisitorCount": { "$size": "$uniqueVisitors" },
      "hoursBetween": {
        "$divide": [{ "$subtract": ["$lastSeen", "$firstSeen"] }, 3600000]
      }
    }
  },
  {
    "$match": {
      "$or": [
        { "journeyCount": { "$gt": 10 } },
        {
          "$and": [
            { "uniqueVisitorCount": { "$gt": 3 } },
            { "hoursBetween": { "$lt": 24 } }
          ]
        }
      ]
    }
  },
  {
    "$project": {
      "ipAddress": "$_id",
      "totalJourneys": "$journeyCount",
      "uniqueVisitors": "$uniqueVisitorCount",
      "hourSpan": {
        "$let": {
          "vars": { "raw": { "$multiply": ["$hoursBetween", 10] } },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "firstSeen": 1,
      "lastSeen": 1,
      "landingPages": 1,
      "suspiciousCount": 1,
      "botCount": 1,
      "_id": 0
    }
  },
  { "$sort": { "uniqueVisitors": -1 } },
  { "$limit": 30 }
]

2. Application-to-Loan Fraud Funnel - Applications

[
  {
    "$lookup": {
      "from": "user_journeys",
      "localField": "customerId",
      "foreignField": "customerId",
      "as": "journeys"
    }
  },
  {
    "$lookup": {
      "from": "devicefingerprints",
      "localField": "customerId",
      "foreignField": "customerId",
      "as": "devices"
    }
  },
  {
    "$group": {
      "_id": "$status",
      "count": { "$sum": 1 },
      "withPasteDetected": {
        "$sum": {
          "$cond": [
            {
              "$anyElementTrue": {
                "$map": {
                  "input": "$journeys",
                  "as": "j",
                  "in": {
                    "$cond": {
                      "if": {
                        "$or": [
                          { "$eq": ["$$j.formBehavior.pasteDetected.pan", true] },
                          { "$eq": ["$$j.formBehavior.pasteDetected.aadhaar", true] },
                          { "$eq": ["$$j.formBehavior.pasteDetected.accountNumber", true] },
                          { "$eq": ["$$j.formBehavior.pasteDetected.ifsc", true] }
                        ]
                      },
                      "then": true,
                      "else": false
                    }
                  }
                }
              }
            },
            1, 0
          ]
        }
      },
      "withSuspiciousJourney": {
        "$sum": {
          "$cond": [
            {
              "$anyElementTrue": {
                "$map": {
                  "input": "$journeys",
                  "as": "j",
                  "in": { "$eq": ["$$j.isSuspicious", true] }
                }
              }
            },
            1, 0
          ]
        }
      },
      "withBotJourney": {
        "$sum": {
          "$cond": [
            {
              "$anyElementTrue": {
                "$map": {
                  "input": "$journeys",
                  "as": "j",
                  "in": { "$eq": ["$$j.isBot", true] }
                }
              }
            },
            1, 0
          ]
        }
      },
      "multiDevice": {
        "$sum": { "$cond": [{ "$gt": [{ "$size": "$devices" }, 2] }, 1, 0] }
      }
    }
  },
  {
    "$project": {
      "applicationStatus": "$_id",
      "count": 1,
      "withPasteDetected": 1,
      "pasteRate": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$withPasteDetected", { "$max": ["$count", 1] }] }, 1000] }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "withSuspiciousJourney": 1,
      "suspiciousRate": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$withSuspiciousJourney", { "$max": ["$count", 1] }] }, 1000] }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "withBotJourney": 1,
      "botRate": {
        "$let": {
          "vars": {
            "raw": { "$multiply": [{ "$divide": ["$withBotJourney", { "$max": ["$count", 1] }] }, 1000] }
          },
          "in": { "$divide": [{ "$subtract": ["$$raw", { "$mod": ["$$raw", 1] }] }, 10] }
        }
      },
      "multiDeviceCount": "$multiDevice",
      "_id": 0
    }
  },
  { "$sort": { "count": -1 } }
]

3. Form Paste Detection (Copy-Paste Fraud Signal) - User Journeys

[
  {"$match": {"behavioralSignals.formPasteDetected": true}},
  {"$lookup": {"from": "customers", "localField": "customerId", "foreignField": "_id", "as": "customer"}},
  {"$unwind": {"path": "$customer", "preserveNullAndEmptyArrays": true}},
  {"$project": {
    "customerId": 1,
    "customerName": {"$concat": [{"$ifNull": ["$customer.firstName", ""]}, " ", {"$ifNull": ["$customer.lastName", ""]}]},
    "landingPage": 1,
    "ipAddress": 1,
    "status": 1,
    "deviceInfo": "$deviceInfo.deviceType",
    "browser": "$deviceInfo.browser",
    "rageClicks": "$behavioralSignals.rageClicks",
    "tabSwitches": "$behavioralSignals.tabSwitches",
    "timestamp": "$createdAt",
    "_id": 0
  }},
  {"$sort": {"timestamp": -1}},
  {"$limit": 50}
]