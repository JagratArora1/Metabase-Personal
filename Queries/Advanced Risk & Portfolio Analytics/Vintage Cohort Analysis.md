1. Loan Vintage by Disbursement Week (Status Breakdown) - Loans

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": {
        "week": {
          "$dateToString": {
            "format": "%Y-%m-%d",
            "date": {
              "$subtract": [
                "$disbursementDate",
                {
                  "$multiply": [
                    {
                      "$subtract": [
                        { "$dayOfWeek": "$disbursementDate" },
                        2
                      ]
                    },
                    86400000
                  ]
                }
              ]
            }
          }
        },
        "status": "$status"
      },
      "count": { "$sum": 1 },
      "totalDisbursed": { "$sum": "$disbursementAmount" },
      "totalOutstanding": { "$sum": "$totalOutstanding" },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } }
    }
  },
  {
    "$group": {
      "_id": "$_id.week",
      "totalLoans": { "$sum": "$count" },
      "totalDisbursed": { "$sum": "$totalDisbursed" },
      "totalOutstanding": { "$sum": "$totalOutstanding" },
      "activeCount": {
        "$sum": {
          "$cond": [{ "$eq": ["$_id.status", "ACTIVE"] }, "$count", 0]
        }
      },
      "closedCount": {
        "$sum": {
          "$cond": [{ "$eq": ["$_id.status", "CLOSED"] }, "$count", 0]
        }
      },
      "overdueCount": {
        "$sum": {
          "$cond": [{ "$eq": ["$_id.status", "OVERDUE"] }, "$count", 0]
        }
      },
      "avgDPD": { "$avg": "$avgDPD" }
    }
  },
  {
    "$addFields": {
      "avgDPD": {
        "$divide": [
          {
            "$toLong": {
              "$add": [
                { "$multiply": ["$avgDPD", 10] },
                0.5
              ]
            }
          },
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
      "weekStarting": "$_id",
      "totalLoans": 1,
      "totalDisbursed": 1,
      "totalOutstanding": 1,
      "activeCount": 1,
      "closedCount": 1,
      "overdueCount": 1,
      "avgDPD": 1,
      "_id": 0
    }
  }
]

2. Weekly Cohort Overdue Rate Trend - Loans

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "disbursementWeek": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": {
            "$subtract": [
              "$disbursementDate",
              {
                "$multiply": [
                  { "$subtract": [{ "$dayOfWeek": "$disbursementDate" }, 2] },
                  86400000
                ]
              }
            ]
          }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$disbursementWeek",
      "total": { "$sum": 1 },
      "overdue": {
        "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] }
      },
      "closed": {
        "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] }
      },
      "totalDisbursed": { "$sum": "$disbursementAmount" },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } }
    }
  },
  {
    "$addFields": {
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdue", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "closureRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$closed", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "avgDPD": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$avgDPD", 10] }, 0.5] } },
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
      "week": "$_id",
      "total": 1,
      "overdue": 1,
      "closed": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "totalDisbursed": 1,
      "avgDPD": 1,
      "_id": 0
    }
  }
]

3. Vintage Overdue Rate By Count - Loans

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "disbursementWeek": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": {
            "$subtract": [
              "$disbursementDate",
              {
                "$multiply": [
                  { "$subtract": [{ "$dayOfWeek": "$disbursementDate" }, 2] },
                  86400000
                ]
              }
            ]
          }
        }
      },
      "dpdBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lte": [{ "$ifNull": ["$dpd", 0] }, 0] }, "then": "0_Current" },
            { "case": { "$lte": ["$dpd", 7] },                      "then": "1_DPD 1-7" },
            { "case": { "$lte": ["$dpd", 15] },                     "then": "2_DPD 8-15" },
            { "case": { "$lte": ["$dpd", 30] },                     "then": "3_DPD 16-30" },
            { "case": { "$lte": ["$dpd", 60] },                     "then": "4_DPD 31-60" }
          ],
          "default": "5_DPD 60+"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "disbursementWeek": "$disbursementWeek",
        "dpdBucket": "$dpdBucket"
      },
      "count": { "$sum": 1 },
      "totalOutstanding": { "$sum": { "$ifNull": ["$totalOutstanding", 0] } }
    }
  },
  {
    "$sort": { "_id.disbursementWeek": 1, "_id.dpdBucket": 1 }
  },
  {
    "$project": {
      "disbursementWeek": "$_id.disbursementWeek",
      "dpdBucket": "$_id.dpdBucket",
      "count": 1,
      "totalOutstanding": 1,
      "_id": 0
    }
  }
]

4. Vintage Overdue Rate By Outstanding Amount - Loans

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "disbursementWeek": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": {
            "$subtract": [
              "$disbursementDate",
              {
                "$multiply": [
                  { "$subtract": [{ "$dayOfWeek": "$disbursementDate" }, 2] },
                  86400000
                ]
              }
            ]
          }
        }
      },
      "dpdBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lte": [{ "$ifNull": ["$dpd", 0] }, 0] }, "then": "0_Current" },
            { "case": { "$lte": ["$dpd", 7] },                      "then": "1_DPD 1-7" },
            { "case": { "$lte": ["$dpd", 15] },                     "then": "2_DPD 8-15" },
            { "case": { "$lte": ["$dpd", 30] },                     "then": "3_DPD 16-30" },
            { "case": { "$lte": ["$dpd", 60] },                     "then": "4_DPD 31-60" }
          ],
          "default": "5_DPD 60+"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "disbursementWeek": "$disbursementWeek",
        "dpdBucket": "$dpdBucket"
      },
      "count": { "$sum": 1 },
      "totalOutstanding": { "$sum": { "$ifNull": ["$totalOutstanding", 0] } }
    }
  },
  {
    "$sort": { "_id.disbursementWeek": 1, "_id.dpdBucket": 1 }
  },
  {
    "$project": {
      "disbursementWeek": "$_id.disbursementWeek",
      "dpdBucket": "$_id.dpdBucket",
      "count": 1,
      "totalOutstanding": 1,
      "_id": 0
    }
  }
]

5. Monthly Vintage Comparison (Outstanding Decay Curve) - Loans

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "disbursementMonth": {
        "$dateToString": { "format": "%Y-%m", "date": "$disbursementDate" }
      },
      "daysSinceDisbursement": {
        "$divide": [
          { "$subtract": [{ "$date": "2026-03-18T00:00:00Z" }, "$disbursementDate"] },
          86400000
        ]
      }
    }
  },
  {
    "$group": {
      "_id": "$disbursementMonth",
      "totalLoans": { "$sum": 1 },
      "totalDisbursed": { "$sum": "$disbursementAmount" },
      "currentOutstanding": { "$sum": "$totalOutstanding" },
      "closedCount": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "avgDaysSince": { "$avg": "$daysSinceDisbursement" }
    }
  },
  {
    "$addFields": {
      "outstandingPct": {
        "$divide": [
          {
            "$toLong": {
              "$add": [
                { "$multiply": [{ "$divide": ["$currentOutstanding", { "$max": ["$totalDisbursed", 1] }] }, 1000] },
                0.5
              ]
            }
          },
          10
        ]
      },
      "closureRate": {
        "$divide": [
          {
            "$toLong": {
              "$add": [
                { "$multiply": [{ "$divide": ["$closedCount", { "$max": ["$totalLoans", 1] }] }, 1000] },
                0.5
              ]
            }
          },
          10
        ]
      },
      "avgDaysSince": {
        "$toLong": { "$add": ["$avgDaysSince", 0.5] }
      }
    }
  },
  {
    "$sort": { "_id": 1 }
  },
  {
    "$project": {
      "month": "$_id",
      "totalLoans": 1,
      "totalDisbursed": 1,
      "currentOutstanding": 1,
      "outstandingPct": 1,
      "closureRate": 1,
      "avgDaysSince": 1,
      "_id": 0
    }
  }
]