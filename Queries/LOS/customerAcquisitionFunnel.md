1. Customer by Employment Type - Customers

[
  {"$match": {"isDeleted": {"$ne": true}, "employmentType": {"$exists": true, "$ne": null}}},
  {"$group": {"_id": "$employmentType", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"employmentType": "$_id", "count": 1, "_id": 0}}
]

2. Customer by Gender - Customers

[
  {"$match": {"isDeleted": {"$ne": true}, "gender": {"$exists": true, "$ne": null}}},
  {"$group": {"_id": "$gender", "count": {"$sum": 1}}},
  {"$project": {"gender": "$_id", "count": 1, "_id": 0}}
]

3. Customer Risk Category - Customers

[
  {"$match": {"isDeleted": {"$ne": true}, "riskCategory": {"$exists": true, "$ne": null}}},
  {"$group": {"_id": "$riskCategory", "count": {"$sum": 1}}},
  {"$project": {"riskCategory": "$_id", "count": 1, "_id": 0}}
]

4. Customer Tier Distribution - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": {"$ifNull": ["$customerTier", "Not Set"]}, "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"tier": "$_id", "count": 1, "_id": 0}}
]

5. Full Funnel - Registration to Disbursement  - Customers

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,

      "Registered": { "$sum": 1 },

      "Basic Details Filled": {
        "$sum": {
          "$cond": [
            { "$eq": ["$isBasicDetailsFilled", true] },
            1,
            0
          ]
        }
      },

      "KYC Filled": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isBasicDetailsFilled", true] },
                { "$eq": ["$isKycDetailsFilled", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Bank Filled": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isBasicDetailsFilled", true] },
                { "$eq": ["$isKycDetailsFilled", true] },
                { "$eq": ["$isBankDetailsFilled", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Submitted": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isBasicDetailsFilled", true] },
                { "$eq": ["$isKycDetailsFilled", true] },
                { "$eq": ["$isBankDetailsFilled", true] },
                { "$eq": ["$isSubmit", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Has Active Loan": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isBasicDetailsFilled", true] },
                { "$eq": ["$isKycDetailsFilled", true] },
                { "$eq": ["$isBankDetailsFilled", true] },
                { "$eq": ["$isSubmit", true] },
                { "$gt": ["$activeLoanCount", 0] }
              ]
            },
            1,
            0
          ]
        }
      }
    }
  },
  {
    "$project": {
      "stages": [
        { "stage": "Registered", "count": "$Registered" },
        { "stage": "Basic Details Filled", "count": "$Basic Details Filled" },
        { "stage": "KYC Filled", "count": "$KYC Filled" },
        { "stage": "Bank Filled", "count": "$Bank Filled" },
        { "stage": "Submitted", "count": "$Submitted" },
        { "stage": "Has Active Loan", "count": "$Has Active Loan" }
      ]
    }
  },
  { "$unwind": "$stages" },
  {
    "$project": {
      "_id": 0,
      "Stage": "$stages.stage",
      "Count": "$stages.count"
    }
  }
]

6. Monthly Registrations with KYC Completion - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m", "date": "$createdAt"}},
    "registered": {"$sum": 1},
    "kycCompleted": {"$sum": {"$cond": [{"$eq": ["$kycStatus", "SUCCESS"]}, 1, 0]}},
    "submitted": {"$sum": {"$cond": [{"$eq": ["$isSubmit", true]}, 1, 0]}}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"month": "$_id", "registered": 1, "kycCompleted": 1, "submitted": 1, "_id": 0}}
]

