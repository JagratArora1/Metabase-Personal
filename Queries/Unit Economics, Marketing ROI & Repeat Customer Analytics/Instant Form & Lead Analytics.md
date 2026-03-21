1. Lead Funnel from Instant Forms - Instantforms

[
  {
    "$group": {
      "_id": { "$ifNull": ["$status", "UNKNOWN"] },
      "count": { "$sum": 1 },
      "avgLoanAmount": { "$avg": { "$ifNull": ["$loanAmountRequested", 0] } }
    }
  },
  {
    "$addFields": {
      "avgLoanAmount": {
        "$toLong": { "$add": ["$avgLoanAmount", 0.5] }
      },
      "stepOrder": {
        "$switch": {
          "branches": [
            { "case": { "$eq": ["$_id", "APPLIED"] },     "then": 1 },
            { "case": { "$eq": ["$_id", "IN_REVIEW"] },   "then": 2 },
            { "case": { "$eq": ["$_id", "APPROVED"] },    "then": 3 },
            { "case": { "$eq": ["$_id", "SANCTIONED"] },  "then": 4 },
            { "case": { "$eq": ["$_id", "DISBURSED"] },   "then": 5 },
            { "case": { "$eq": ["$_id", "REJECTED"] },    "then": 6 },
            { "case": { "$eq": ["$_id", "ABANDONED"] },   "then": 7 }
          ],
          "default": 99
        }
      },
      "stepName": {
        "$switch": {
          "branches": [
            { "case": { "$eq": ["$_id", "APPLIED"] },     "then": "1. Applied" },
            { "case": { "$eq": ["$_id", "IN_REVIEW"] },   "then": "2. In Review" },
            { "case": { "$eq": ["$_id", "APPROVED"] },    "then": "3. Approved" },
            { "case": { "$eq": ["$_id", "SANCTIONED"] },  "then": "4. Sanctioned" },
            { "case": { "$eq": ["$_id", "DISBURSED"] },   "then": "5. Disbursed" },
            { "case": { "$eq": ["$_id", "REJECTED"] },    "then": "6. Rejected" },
            { "case": { "$eq": ["$_id", "ABANDONED"] },   "then": "7. Abandoned" }
          ],
          "default": "$_id"
        }
      }
    }
  },
  {
    "$sort": { "stepOrder": 1 }
  },
  {
    "$project": {
      "stepName": 1,
      "count": 1,
      "avgLoanAmount": 1,
      "_id": 0
    }
  }
]

2. Lead Source Distribution - Instantforms

[
  {
    "$group": {
      "_id": { "$ifNull": ["$source", "DIRECT"] },
      "count": { "$sum": 1 },
      "avgAmount": { "$avg": { "$ifNull": ["$loanAmountRequested", 0] } }
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
    "$sort": { "count": -1 }
  },
  {
    "$project": {
      "source": "$_id",
      "count": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]

3. Lead-to-Customer Conversion Timeline - Instantforms

[
  {
    "$addFields": {
      "date": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": { "$add": ["$createdAt", 19800000] }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$date",
      "totalLeads": { "$sum": 1 },
      "converted":  { "$sum": { "$cond": [{ "$eq": ["$status", "CONVERTED"] }, 1, 0] } },
      "contacted":  { "$sum": { "$cond": [{ "$eq": ["$status", "CONTACTED"] }, 1, 0] } },
      "rejected":   { "$sum": { "$cond": [{ "$eq": ["$status", "REJECTED"] }, 1, 0] } }
    }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$converted", { "$max": ["$totalLeads", 1] }] }, 1000] }, 0.5] } },
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
      "date": "$_id",
      "totalLeads": 1,
      "converted": 1,
      "contacted": 1,
      "rejected": 1,
      "conversionRate": 1,
      "_id": 0
    }
  }
]