1. DPD Bucket Distribution (Active Portfolio) - Loans

[
  {
    "$match": {
      "status": { "$in": ["ACTIVE", "OVERDUE", "NPA"] },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "dpdBucketOrder": {
        "$switch": {
          "branches": [
            { "case": { "$lte": [{ "$ifNull": ["$dpd", 0] }, 0] }, "then": 0 },
            { "case": { "$lte": ["$dpd", 7] },                      "then": 1 },
            { "case": { "$lte": ["$dpd", 15] },                     "then": 2 },
            { "case": { "$lte": ["$dpd", 30] },                     "then": 3 },
            { "case": { "$lte": ["$dpd", 60] },                     "then": 4 }
          ],
          "default": 5
        }
      },
      "dpdGroup": {
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
        "dpdGroup": "$dpdGroup",
        "dpdBucketOrder": "$dpdBucketOrder"
      },
      "loanCount": { "$sum": 1 },
      "totalOutstanding": { "$sum": { "$ifNull": ["$totalOutstanding", 0] } },
      "avgLoanAmount": { "$avg": { "$ifNull": ["$principalAmount", 0] } },
      "avgBounceCount": { "$avg": { "$ifNull": ["$paymentBehavior.bounceCount", 0] } }
    }
  },
  {
    "$group": {
      "_id": null,
      "buckets": {
        "$push": {
          "dpdGroup": "$_id.dpdGroup",
          "dpdBucketOrder": "$_id.dpdBucketOrder",
          "loanCount": "$loanCount",
          "totalOutstanding": "$totalOutstanding",
          "avgLoanAmount": "$avgLoanAmount",
          "avgBounceCount": "$avgBounceCount"
        }
      },
      "grandTotalLoans": { "$sum": "$loanCount" },
      "grandTotalOutstanding": { "$sum": "$totalOutstanding" }
    }
  },
  { "$unwind": "$buckets" },
  {
    "$addFields": {
      "portfolioSharePct": {
        "$divide": [
          {
            "$toLong": {
              "$add": [
                {
                  "$multiply": [
                    { "$divide": ["$buckets.totalOutstanding", { "$max": ["$grandTotalOutstanding", 1] }] },
                    1000
                  ]
                },
                0.5
              ]
            }
          },
          10
        ]
      },
      "avgLoanAmount": {
        "$toLong": { "$add": ["$buckets.avgLoanAmount", 0.5] }
      },
      "avgBounceCount": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$buckets.avgBounceCount", 10] }, 0.5] } },
          10
        ]
      }
    }
  },
  {
    "$sort": { "buckets.dpdBucketOrder": 1 }
  },
  {
    "$project": {
      "dpdBucket": "$buckets.dpdGroup",
      "loanCount": "$buckets.loanCount",
      "totalOutstanding": "$buckets.totalOutstanding",
      "avgLoanAmount": 1,
      "avgBounceCount": 1,
      "portfolioSharePct": 1,
      "_id": 0
    }
  }
]

2. Portfolio at Risk (PAR) Trend - Daily - Loans 

[
  {
    "$match": {
      "status": { "$in": ["ACTIVE", "OVERDUE", "NPA"] },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,
      "totalOutstanding": { "$sum": "$totalOutstanding" },
      "par1":  { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 1]  }, "$totalOutstanding", 0] } },
      "par7":  { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 7]  }, "$totalOutstanding", 0] } },
      "par15": { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 15] }, "$totalOutstanding", 0] } },
      "par30": { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 30] }, "$totalOutstanding", 0] } },
      "par60": { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 60] }, "$totalOutstanding", 0] } },
      "par90": { "$sum": { "$cond": [{ "$gte": [{ "$ifNull": ["$dpd", 0] }, 90] }, "$totalOutstanding", 0] } },
      "totalLoans": { "$sum": 1 }
    }
  },
  {
    "$addFields": {
      "par1_pct":  { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par1",  { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] },
      "par7_pct":  { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par7",  { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] },
      "par15_pct": { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par15", { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] },
      "par30_pct": { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par30", { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] },
      "par60_pct": { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par60", { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] },
      "par90_pct": { "$divide": [{ "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$par90", { "$max": ["$totalOutstanding", 1] }] }, 10000] }, 0.5] } }, 100] }
    }
  },
  {
    "$project": {
      "totalOutstanding": 1,
      "totalLoans": 1,
      "par1_pct": 1,
      "par7_pct": 1,
      "par15_pct": 1,
      "par30_pct": 1,
      "par60_pct": 1,
      "par90_pct": 1,
      "_id": 0
    }
  }
]

3. Loans Entering Overdue This Week (Early Warning) - Loans

[
  {
    "$match": {
      "status": "OVERDUE",
      "dpd": { "$gte": 1, "$lte": 7 },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "customer"
    }
  },
  {
    "$unwind": { "path": "$customer", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "startDate": {
        "$dateFromString": {
          "dateString": { "$concat": [ {{day_start}}, "T18:30:00.000+00:00" ] }
        }
      },
      "endDate": {
        "$dateFromString": {
          "dateString": { "$concat": [ {{day_end}}, "T18:29:59.000+00:00" ] }
        }
      },
      "dueDateIST": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": { "$add": ["$dueDate", 19800000] }
        }
      },
      "disbursementDateIST": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": { "$add": ["$disbursementDate", 19800000] }
        }
      }
    }
  },
  {
    "$match": {
      "$expr": {
        "$and": [
          { "$gte": ["$dueDate", "$startDate"] },
          { "$lte": ["$dueDate", "$endDate"] }
        ]
      }
    }
  },
  {
    "$project": {
      "loanNumber": 1,
      "customerName": { "$ifNull": ["$customer.fullName", "Unknown"] },
      "mobile": { "$ifNull": ["$customer.mobile", "N/A"] },
      "principalAmount": 1,
      "totalOutstanding": 1,
      "dpd": 1,
      "dueDateIST": 1,
      "disbursementDateIST": 1,
      "tenure": 1,
      "_id": 0
    }
  },
  {
    "$sort": { "dpd": -1 }
  },
  {
    "$limit": 50
  }
]