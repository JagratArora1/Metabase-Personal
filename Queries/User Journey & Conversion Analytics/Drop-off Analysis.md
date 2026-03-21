1. Drop-off at Each KYC Step - User Journeys

[
  {
    "$group": {
      "_id": null,
      "started": { "$sum": 1 },
      "panAttempted": {
        "$sum": {
          "$cond": [
            { "$gt": [{ "$ifNull": ["$kycProgress.panAttempts", 0] }, 0] },
            1,
            0
          ]
        }
      },
      "panVerified": {
        "$sum": {
          "$cond": [{ "$eq": ["$kycProgress.panVerified", true] }, 1, 0]
        }
      },
      "aadhaarVerified": {
        "$sum": {
          "$cond": [{ "$eq": ["$kycProgress.aadhaarVerified", true] }, 1, 0]
        }
      },
      "selfieVerified": {
        "$sum": {
          "$cond": [{ "$eq": ["$kycProgress.selfieVerified", true] }, 1, 0]
        }
      },
      "bankVerified": {
        "$sum": {
          "$cond": [{ "$eq": ["$kycProgress.bankVerified", true] }, 1, 0]
        }
      },
      "esignCompleted": {
        "$sum": {
          "$cond": [{ "$eq": ["$kycProgress.esignCompleted", true] }, 1, 0]
        }
      }
    }
  },
  {
    "$project": {
      "stages": [
        { "stage": "Started", "count": "$started", "order": 1 },
        { "stage": "PAN Attempted", "count": "$panAttempted", "order": 2 },
        { "stage": "PAN Verified", "count": "$panVerified", "order": 3 },
        { "stage": "Aadhaar Verified", "count": "$aadhaarVerified", "order": 4 },
        { "stage": "Selfie Verified", "count": "$selfieVerified", "order": 5 },
        { "stage": "Bank Verified", "count": "$bankVerified", "order": 6 },
        { "stage": "eSign Completed", "count": "$esignCompleted", "order": 7 }
      ]
    }
  },
  { "$unwind": "$stages" },
  { "$replaceRoot": { "newRoot": "$stages" } },
  { "$sort": { "order": 1 } }
]

2. Verification Time Analysis - User Journeys

[
  {"$match": {"verificationFriction": {"$exists": true}}},
  {"$group": {
    "_id": null,
    "avgPanVerificationTime": {"$avg": {"$ifNull": ["$verificationFriction.panVerificationTime", 0]}},
    "avgAadhaarOtpWaitTime": {"$avg": {"$ifNull": ["$verificationFriction.aadhaarOtpWaitTime", 0]}},
    "avgBankVerificationTime": {"$avg": {"$ifNull": ["$verificationFriction.bankVerificationTime", 0]}},
    "avgEsignTime": {"$avg": {"$ifNull": ["$verificationFriction.esignTime", 0]}}
  }},
  {"$project": {"_id": 0}}
]

3. Average Verification Retries - User Journeys 

[
  {"$match": {"kycProgress": {"$exists": true}}},
  {"$group": {
    "_id": null,
    "avgPanRetries": {"$avg": {"$ifNull": ["$verificationFriction.panRetriesBeforeSuccess", 0]}},
    "avgAadhaarRetries": {"$avg": {"$ifNull": ["$verificationFriction.aadhaarRetriesBeforeSuccess", 0]}},
    "avgBankRetries": {"$avg": {"$ifNull": ["$verificationFriction.bankRetriesBeforeSuccess", 0]}},
    "avgAadhaarOtpResends": {"$avg": {"$ifNull": ["$verificationFriction.aadhaarOtpResendCount", 0]}},
    "avgAadhaarOtpExpired": {"$avg": {"$ifNull": ["$verificationFriction.aadhaarOtpExpiredCount", 0]}}
  }},
  {"$project": {"_id": 0}}
]

4. Technical Error Rate - User Journeys 

[
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "slowNetworkDetected": {"$sum": {"$cond": [{"$eq": ["$technicalIssues.slowNetworkDetected", true]}, 1, 0]}},
    "hasApiTimeouts": {"$sum": {"$cond": [{"$gt": [{"$ifNull": ["$technicalIssues.apiTimeouts", 0]}, 0]}, 1, 0]}},
    "hasJsErrors": {"$sum": {"$cond": [{"$gt": [{"$size": {"$ifNull": ["$technicalIssues.jsErrors", []]}}, 0]}, 1, 0]}}
  }},
  {"$project": {
    "_id": 0,
    "total": 1,
    "slowNetworkRate": {"$multiply": [{"$divide": ["$slowNetworkDetected", {"$max": ["$total", 1]}]}, 100]},
    "apiTimeoutRate": {"$multiply": [{"$divide": ["$hasApiTimeouts", {"$max": ["$total", 1]}]}, 100]},
    "jsErrorRate": {"$multiply": [{"$divide": ["$hasJsErrors", {"$max": ["$total", 1]}]}, 100]}
  }}
]

5. Abandonment Reasons Distribution - User Journeys

[
  {"$match": {"journeyStatus": "ABANDONED", "abandonmentReason": {"$exists": true, "$ne": null}}},
  {"$group": {"_id": "$abandonmentReason", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"reason": "$_id", "count": 1, "_id": 0}}
]

6. Loan Amount Change Behavior - User Journeys

[
  {"$match": {"loanBehavior.initialRequestedAmount": {"$gt": 0}}},
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "changedAmount": {"$sum": {"$cond": [
      {"$ne": ["$loanBehavior.initialRequestedAmount", "$loanBehavior.finalRequestedAmount"]},
      1, 0
    ]}},
    "reducedAfterRejection": {"$sum": {"$cond": [{"$eq": ["$loanBehavior.amountReducedAfterRejection", true]}, 1, 0]}},
    "avgInitialAmount": {"$avg": "$loanBehavior.initialRequestedAmount"},
    "avgFinalAmount": {"$avg": "$loanBehavior.finalRequestedAmount"},
    "avgTenureChanges": {"$avg": {"$ifNull": ["$loanBehavior.tenureChanges", 0]}}
  }},
  {"$project": {"_id": 0}}
]

7. Comeback Pattern Analysis - User Journeys

[
  {"$match": {"comebackPattern.totalVisits": {"$gt": 1}}},
  {"$group": {
    "_id": "$comebackPattern.totalVisits",
    "count": {"$sum": 1},
    "converted": {"$sum": {"$cond": [{"$eq": ["$journeyStatus", "CONVERTED"]}, 1, 0]}}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {
    "totalVisits": "$_id",
    "users": "$count",
    "converted": 1,
    "conversionRate": {"$multiply": [{"$divide": ["$converted", {"$max": ["$count", 1]}]}, 100]},
    "_id": 0
  }}
]

8. Rejection Category Distribution - User Journeys

[
  {"$match": {"finalStatus": "REJECTED", "rejectionCategory": {"$exists": true, "$ne": null}}},
  {"$group": {"_id": "$rejectionCategory", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"category": "$_id", "count": 1, "_id": 0}}
]