1. Applications by Priority - Applications

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": {"$ifNull": ["$priority", "NOT_SET"]}, "count": {"$sum": 1}}},
  {"$project": {"priority": "$_id", "count": 1, "_id": 0}}
]

2. Applications Pending > 24 Hours - Applications

[
  {
    "$match": {
      "status": { "$in": ["PENDING", "PROCESSING", "PENDING_APPLICATION"] },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$addFields": {
      "cutoffTime": {
        "$dateFromString": {
          "dateString": { "$concat": [{{start_date}}, "T18:30:00.000+00:00"] }
        }
      }
    }
  },
  {
    "$match": {
      "$expr": { "$lt": ["$createdAt", "$cutoffTime"] }
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
    "$project": {
      "_id": 0,
      "applicationNumber": 1,
      "customerName": "$customer.fullName",
      "mobile": "$customer.mobile",
      "status": 1,
      "requestedLoanAmount": 1,
      "createdAt": 1
    }
  },
  { "$sort": { "createdAt": -1 } },
  { "$limit": 100 }
]

3. Application Status Over Time (Weekly) - Applications

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": {
        "week": {
          "$dateToString": {
            "format": "%Y-W%V",
            "date": "$createdAt"
          }
        },
        "status": "$status"
      },
      "count": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "week": "$_id.week",
      "status": "$_id.status",
      "count": 1
    }
  },
  {
    "$sort": {
      "week": 1
    }
  }
]

4. Approval Rate Trend (Weekly) - Applications

[
  { "$match": { "isDeleted": { "$ne": true } } },

  {
    "$group": {
      "_id": {
        "$dateToString": {
          "format": "%Y-%V",
          "date": "$createdAt",
          "timezone": "Asia/Kolkata"
        }
      },
      "total": { "$sum": 1 },
      "approved": {
        "$sum": {
          "$cond": [
            {
              "$in": [
                "$status",
                ["APPROVED", "PENDING_DISBURSED", "DISBURSED", "CLOSED"]
              ]
            },
            1,
            0
          ]
        }
      },
      "rejected": {
        "$sum": {
          "$cond": [
            { "$eq": ["$status", "REJECTED"] },
            1,
            0
          ]
        }
      }
    }
  },

  { "$sort": { "_id": 1 } },

  {
    "$project": {
      "_id": 0,
      "week": "$_id",
      "approvalRate": {
        "$multiply": [
          { "$divide": ["$approved", { "$max": ["$total", 1] }] },
          100
        ]
      },
      "rejectionRate": {
        "$multiply": [
          { "$divide": ["$rejected", { "$max": ["$total", 1] }] },
          100
        ]
      }
    }
  }
]

5. Average Loan Amount - Requested vs Approved - Applications

[
  {"$match": {"isDeleted": {"$ne": true}, "status": {"$in": ["APPROVED", "PENDING_DISBURSED", "DISBURSED", "CLOSED"]}}},
  {"$group": {
    "_id": null,
    "avgRequested": {"$avg": "$requestedLoanAmount"},
    "avgApproved": {"$avg": "$approvedLoanAmount"},
    "avgNetDisbursal": {"$avg": "$netDisbursalAmount"}
  }}
]

6. Daily Application Volume - Applications

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m-%d", "date": "$createdAt"}},
    "count": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"date": "$_id", "count": 1, "_id": 0}}
]

7. Loan Amount Distribution - Applications

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "requestedLoanAmount": { "$gt": 0 }
    }
  },
  {
    "$project": {
      "range": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$requestedLoanAmount", 10000] }, "then": "0-10K" },
            { "case": { "$lt": ["$requestedLoanAmount", 20000] }, "then": "10K-20K" },
            { "case": { "$lt": ["$requestedLoanAmount", 30000] }, "then": "20K-30K" },
            { "case": { "$lt": ["$requestedLoanAmount", 50000] }, "then": "30K-50K" },
            { "case": { "$lt": ["$requestedLoanAmount", 75000] }, "then": "50K-75K" },
            { "case": { "$lt": ["$requestedLoanAmount", 100000] }, "then": "75K-100K" },
            { "case": { "$lt": ["$requestedLoanAmount", 200000] }, "then": "100K-200K" },
            { "case": { "$lt": ["$requestedLoanAmount", 500000] }, "then": "200K-500K" }
          ],
          "default": "500K+"
        }
      },
      "sortOrder": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$requestedLoanAmount", 10000] }, "then": 1 },
            { "case": { "$lt": ["$requestedLoanAmount", 20000] }, "then": 2 },
            { "case": { "$lt": ["$requestedLoanAmount", 30000] }, "then": 3 },
            { "case": { "$lt": ["$requestedLoanAmount", 50000] }, "then": 4 },
            { "case": { "$lt": ["$requestedLoanAmount", 75000] }, "then": 5 },
            { "case": { "$lt": ["$requestedLoanAmount", 100000] }, "then": 6 },
            { "case": { "$lt": ["$requestedLoanAmount", 200000] }, "then": 7 },
            { "case": { "$lt": ["$requestedLoanAmount", 500000] }, "then": 8 }
          ],
          "default": 9
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "range": "$range",
        "sortOrder": "$sortOrder"
      },
      "count": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "range": "$_id.range",
      "sortOrder": "$_id.sortOrder",
      "count": 1
    }
  },
  {
    "$sort": {
      "sortOrder": 1
    }
  }
]

8. Total Applications by Status - Applications

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$status", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"status": "$_id", "count": 1, "_id": 0}}
]