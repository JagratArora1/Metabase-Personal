1. CIBIL Score vs Loan Performance - Loans 

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "principalAmount": { "$gt": 0 }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "cust"
    }
  },
  {
    "$unwind": { "path": "$cust", "preserveNullAndEmptyArrays": true }
  },
  {
    "$match": {
      "cust.cibilScore": { "$gt": 0 }
    }
  },
  {
    "$addFields": {
      "cibilBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$cust.cibilScore", 600] }, "then": "A: Below 600" },
            { "case": { "$lt": ["$cust.cibilScore", 650] }, "then": "B: 600-650" },
            { "case": { "$lt": ["$cust.cibilScore", 700] }, "then": "C: 650-700" },
            { "case": { "$lt": ["$cust.cibilScore", 750] }, "then": "D: 700-750" },
            { "case": { "$lt": ["$cust.cibilScore", 800] }, "then": "E: 750-800" }
          ],
          "default": "F: 800+"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$cibilBucket",
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "closed": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } },
      "avgAmount": { "$avg": { "$ifNull": ["$principalAmount", 0] } }
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
      },
      "avgAmount": {
        "$toLong": { "$add": ["$avgAmount", 0.5] }
      }
    }
  },
  {
    "$sort": { "_id": 1 }
  },
  {
    "$project": {
      "cibilRange": "$_id",
      "total": 1,
      "overdue": 1,
      "closed": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "avgDPD": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]

2. Multi-Dimensional Risk Matrix (Amount) - Loans

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "tenure": { "$in": [7, 15, 30] }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "cust"
    }
  },
  {
    "$unwind": { "path": "$cust", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "cibilGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": [{ "$ifNull": ["$cust.cibilScore", 0] }, 650] }, "then": "Low (<650)" },
            { "case": { "$lt": ["$cust.cibilScore", 750] }, "then": "Medium (650-750)" }
          ],
          "default": "High (750+)"
        }
      },
      "amountGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$principalAmount", 10000] }, "then": "A: Small (<10K)" },
            { "case": { "$lt": ["$principalAmount", 20000] }, "then": "B: Medium (10-20K)" }
          ],
          "default": "C: Large (20K+)"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "tenure": "$tenure",
        "cibil": "$cibilGroup",
        "amount": "$amountGroup"
      },
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } }
    }
  },
  {
    "$match": { "total": { "$gte": 3 } }
  },
  {
    "$addFields": {
      "tenure": { "$concat": [{ "$toString": "$_id.tenure" }, "d"] },
      "cibilGroup": "$_id.cibil",
      "amountGroup": "$_id.amount",
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdue", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
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
    "$sort": { "overdueRate": -1 }
  },
  {
    "$project": {
      "tenure": 1,
      "cibilGroup": 1,
      "amountGroup": 1,
      "total": 1,
      "overdue": 1,
      "overdueRate": 1,
      "avgDPD": 1,
      "_id": 0
    }
  }
]

3. Multi-Dimensional Risk Matrix (Tenure) - Loans 

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "tenure": { "$in": [7, 15, 30] }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "cust"
    }
  },
  {
    "$unwind": { "path": "$cust", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "cibilGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": [{ "$ifNull": ["$cust.cibilScore", 0] }, 650] }, "then": "Low (<650)" },
            { "case": { "$lt": ["$cust.cibilScore", 750] }, "then": "Medium (650-750)" }
          ],
          "default": "High (750+)"
        }
      },
      "amountGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$principalAmount", 10000] }, "then": "A: Small (<10K)" },
            { "case": { "$lt": ["$principalAmount", 20000] }, "then": "B: Medium (10-20K)" }
          ],
          "default": "C: Large (20K+)"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "tenure": "$tenure",
        "cibil": "$cibilGroup",
        "amount": "$amountGroup"
      },
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } }
    }
  },
  {
    "$match": { "total": { "$gte": 3 } }
  },
  {
    "$addFields": {
      "tenure": { "$concat": [{ "$toString": "$_id.tenure" }, "d"] },
      "cibilGroup": "$_id.cibil",
      "amountGroup": "$_id.amount",
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdue", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
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
    "$sort": { "overdueRate": -1 }
  },
  {
    "$project": {
      "tenure": 1,
      "cibilGroup": 1,
      "amountGroup": 1,
      "total": 1,
      "overdue": 1,
      "overdueRate": 1,
      "avgDPD": 1,
      "_id": 0
    }
  }
]

4. Multi-Dimensional Risk Matrix (Cibil Group) - Loans

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "tenure": { "$in": [7, 15, 30] }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "cust"
    }
  },
  {
    "$unwind": { "path": "$cust", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "cibilGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": [{ "$ifNull": ["$cust.cibilScore", 0] }, 650] }, "then": "Low (<650)" },
            { "case": { "$lt": ["$cust.cibilScore", 750] }, "then": "Medium (650-750)" }
          ],
          "default": "High (750+)"
        }
      },
      "amountGroup": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$principalAmount", 10000] }, "then": "A: Small (<10K)" },
            { "case": { "$lt": ["$principalAmount", 20000] }, "then": "B: Medium (10-20K)" }
          ],
          "default": "C: Large (20K+)"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "tenure": "$tenure",
        "cibil": "$cibilGroup",
        "amount": "$amountGroup"
      },
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } }
    }
  },
  {
    "$match": { "total": { "$gte": 3 } }
  },
  {
    "$addFields": {
      "tenure": { "$concat": [{ "$toString": "$_id.tenure" }, "d"] },
      "cibilGroup": "$_id.cibil",
      "amountGroup": "$_id.amount",
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdue", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
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
    "$sort": { "overdueRate": -1 }
  },
  {
    "$project": {
      "tenure": 1,
      "cibilGroup": 1,
      "amountGroup": 1,
      "total": 1,
      "overdue": 1,
      "overdueRate": 1,
      "avgDPD": 1,
      "_id": 0
    }
  }
]

5. Loan Amount vs Default Rate - Loans

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "principalAmount": { "$gt": 0 }
    }
  },
  {
    "$addFields": {
      "amountBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$principalAmount", 5000] },  "then": "A: 0-5K" },
            { "case": { "$lt": ["$principalAmount", 10000] }, "then": "B: 5K-10K" },
            { "case": { "$lt": ["$principalAmount", 15000] }, "then": "C: 10K-15K" },
            { "case": { "$lt": ["$principalAmount", 20000] }, "then": "D: 15K-20K" },
            { "case": { "$lt": ["$principalAmount", 25000] }, "then": "E: 20K-25K" },
            { "case": { "$lt": ["$principalAmount", 30000] }, "then": "F: 25K-30K" }
          ],
          "default": "G: 30K+"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$amountBucket",
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "closed": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } },
      "totalDisbursed": { "$sum": "$disbursementAmount" }
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
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$closed", { "$max": ["$total", 1] }] }, 1000] } , 0.5] } },
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
      "amountRange": "$_id",
      "total": 1,
      "overdue": 1,
      "closed": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "avgDPD": 1,
      "totalDisbursed": 1,
      "_id": 0
    }
  }
]

6. Tenure vs Default Rate (Risk Matrix) - Loans 

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "tenure": { "$in": [7, 15, 30, 60, 90] }
    }
  },
  {
    "$group": {
      "_id": "$tenure",
      "totalLoans": { "$sum": 1 },
      "overdueLoans": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "closedLoans": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } },
      "totalDisbursed": { "$sum": "$disbursementAmount" },
      "totalOutstanding": { "$sum": "$totalOutstanding" }
    }
  },
  {
    "$addFields": {
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdueLoans", { "$max": ["$totalLoans", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "closureRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$closedLoans", { "$max": ["$totalLoans", 1] }] }, 1000] }, 0.5] } },
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
      "tenure": { "$concat": [{ "$toString": "$_id" }, " days"] },
      "totalLoans": 1,
      "overdueLoans": 1,
      "closedLoans": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "avgDPD": 1,
      "totalDisbursed": 1,
      "totalOutstanding": 1,
      "_id": 0
    }
  }
]

7. Employment Type vs Loan Performance - Loans 

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "cust"
    }
  },
  {
    "$unwind": { "path": "$cust", "preserveNullAndEmptyArrays": true }
  },
  {
    "$group": {
      "_id": { "$ifNull": ["$cust.employmentType", "UNKNOWN"] },
      "total": { "$sum": 1 },
      "overdue": { "$sum": { "$cond": [{ "$in": ["$status", ["OVERDUE", "NPA"]] }, 1, 0] } },
      "closed": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } },
      "avgAmount": { "$avg": { "$ifNull": ["$principalAmount", 0] } },
      "totalDisbursed": { "$sum": "$disbursementAmount" }
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
      },
      "avgAmount": {
        "$toLong": { "$add": ["$avgAmount", 0.5] }
      }
    }
  },
  {
    "$sort": { "total": -1 }
  },
  {
    "$project": {
      "employmentType": "$_id",
      "total": 1,
      "overdue": 1,
      "closed": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "avgDPD": 1,
      "avgAmount": 1,
      "totalDisbursed": 1,
      "_id": 0
    }
  }
]