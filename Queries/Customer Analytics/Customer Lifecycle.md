1. Monthly New Customers - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m", "date": "$createdAt"}},
    "newCustomers": {"$sum": 1},
    "kycCompleted": {"$sum": {"$cond": [{"$eq": ["$kycStatus", "SUCCESS"]}, 1, 0]}}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"month": "$_id", "newCustomers": 1, "kycCompleted": 1, "_id": 0}}
]

2. Customer Loans Frequency Distribution - Customers

[
  {"$match": {"totalLoans": {"$gt": 0}, "isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$totalLoans", "count": {"$sum": 1}}},
  {"$sort": {"_id": 1}},
  {"$project": {"loanCount": "$_id", "customers": "$count", "_id": 0}}
]

3. Repeat Customers - Customers

[
  {"$match": {"totalLoans": {"$gte": 2}, "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": null,
    "repeatCustomers": {"$sum": 1},
    "avgLoansPerRepeat": {"$avg": "$totalLoans"},
    "avgLifetimeValue": {"$avg": "$lifetimeValue"}
  }},
  {"$project": {"_id": 0}}
]

4. Lifetime Value Distribution - Customers

[
  {
    "$match": {
      "lifetimeValue": { "$gt": 0 },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "range": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$lifetimeValue", 10000] }, "then": "0-10K" },
            { "case": { "$lt": ["$lifetimeValue", 25000] }, "then": "10K-25K" },
            { "case": { "$lt": ["$lifetimeValue", 50000] }, "then": "25K-50K" },
            { "case": { "$lt": ["$lifetimeValue", 100000] }, "then": "50K-1L" },
            { "case": { "$lt": ["$lifetimeValue", 200000] }, "then": "1L-2L" },
            { "case": { "$lt": ["$lifetimeValue", 500000] }, "then": "2L-5L" }
          ],
          "default": "5L+"
        }
      },
      "totalLoans": 1
    }
  },
  {
    "$group": {
      "_id": "$range",
      "count": { "$sum": 1 },
      "avgLoans": { "$avg": "$totalLoans" }
    }
  },
  {
    "$sort": { "_id": 1 }
  }
]

5. Reapply Eligibility - Customers 

[
  {"$match": {"reapply.isEligible": true, "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": "$reapply.approvalType",
    "count": {"$sum": 1}
  }},
  {"$project": {"approvalType": "$_id", "count": 1, "_id": 0}}
]

6. Customer Churn (Registered but No Loan) - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "noApplication": {"$sum": {"$cond": [{"$lte": [{"$ifNull": ["$totalApplications", 0]}, 0]}, 1, 0]}},
    "hasApplication": {"$sum": {"$cond": [{"$gt": [{"$ifNull": ["$totalApplications", 0]}, 0]}, 1, 0]}},
    "hasLoan": {"$sum": {"$cond": [{"$gt": [{"$ifNull": ["$totalLoans", 0]}, 0]}, 1, 0]}}
  }},
  {"$project": {
    "_id": 0,
    "total": 1,
    "noApplication": 1,
    "churnRate": {"$multiply": [{"$divide": ["$noApplication", {"$max": ["$total", 1]}]}, 100]},
    "hasApplication": 1,
    "hasLoan": 1
  }}
]

7. Lifetime Value Distribution - Customers 

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "noApplication": {"$sum": {"$cond": [{"$lte": [{"$ifNull": ["$totalApplications", 0]}, 0]}, 1, 0]}},
    "hasApplication": {"$sum": {"$cond": [{"$gt": [{"$ifNull": ["$totalApplications", 0]}, 0]}, 1, 0]}},
    "hasLoan": {"$sum": {"$cond": [{"$gt": [{"$ifNull": ["$totalLoans", 0]}, 0]}, 1, 0]}}
  }},
  {"$project": {
    "_id": 0,
    "total": 1,
    "noApplication": 1,
    "churnRate": {"$multiply": [{"$divide": ["$noApplication", {"$max": ["$total", 1]}]}, 100]},
    "hasApplication": 1,
    "hasLoan": 1
  }}
]

