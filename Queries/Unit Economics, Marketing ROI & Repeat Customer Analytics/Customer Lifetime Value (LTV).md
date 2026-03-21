1. Overview - Loans

[
  {"$match": {"isDeleted": {"$ne": true}, "status": "CLOSED", "closureDate": {"$exists": true}}},
  {"$sort": {"customerId": 1, "closureDate": 1}},
  {"$group": {
    "_id": "$customerId",
    "loanCount": {"$sum": 1},
    "closureDates": {"$push": "$closureDate"},
    "disbursementDates": {"$push": "$disbursementDate"}
  }},
  {"$match": {"loanCount": {"$gte": 2}}},
  {"$addFields": {
    "prevClosures": {"$slice": ["$closureDates", 0, {"$subtract": [{"$size": "$closureDates"}, 1]}]},
    "nextDisbursements": {"$slice": ["$disbursementDates", 1, {"$size": "$disbursementDates"}]}
  }},
  {"$addFields": {
    "gapPairs": {"$zip": {"inputs": ["$prevClosures", "$nextDisbursements"]}}
  }},
  {"$addFields": {
    "avgGapDays": {"$avg": {
      "$map": {
        "input": "$gapPairs",
        "as": "pair",
        "in": {"$divide": [
          {"$subtract": [
            {"$arrayElemAt": ["$$pair", 1]},
            {"$arrayElemAt": ["$$pair", 0]}
          ]},
          86400000
        ]}
      }
    }}
  }},
  {"$group": {
    "_id": null,
    "repeatCustomers": {"$sum": 1},
    "avgGapDays": {"$avg": "$avgGapDays"},
    "minGapDays": {"$min": "$avgGapDays"},
    "maxGapDays": {"$max": "$avgGapDays"}
  }},
  {"$project": {
    "_id": 0,
    "repeatCustomers": 1,
    "avgGapDays": {
      "$divide": [
        {"$subtract": [
          {"$multiply": ["$avgGapDays", 10]},
          {"$mod": [{"$multiply": ["$avgGapDays", 10]}, 1]}
        ]},
        10
      ]
    },
    "minGapDays": {
      "$divide": [
        {"$subtract": [
          {"$multiply": ["$minGapDays", 10]},
          {"$mod": [{"$multiply": ["$minGapDays", 10]}, 1]}
        ]},
        10
      ]
    },
    "maxGapDays": {
      "$divide": [
        {"$subtract": [
          {"$multiply": ["$maxGapDays", 10]},
          {"$mod": [{"$multiply": ["$maxGapDays", 10]}, 1]}
        ]},
        10
      ]
    }
  }}
]

2. Repeat Customer Loan Performance vs First-Time - Loans

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$lookup": {"from": "customers", "localField": "customerId", "foreignField": "_id", "as": "cust"}},
  {"$unwind": {"path": "$cust", "preserveNullAndEmptyArrays": true}},
  {"$addFields": {
    "isRepeat": {"$cond": [{"$gte": [{"$ifNull": ["$cust.totalLoans", 0]}, 2]}, true, false]}
  }},
  {"$group": {
    "_id": "$isRepeat",
    "totalLoans": {"$sum": 1},
    "overdue": {"$sum": {"$cond": [{"$in": ["$status", ["OVERDUE", "NPA"]]}, 1, 0]}},
    "closed": {"$sum": {"$cond": [{"$eq": ["$status", "CLOSED"]}, 1, 0]}},
    "avgDPD": {"$avg": {"$ifNull": ["$dpd", 0]}},
    "avgAmount": {"$avg": "$principalAmount"},
    "totalDisbursed": {"$sum": "$disbursementAmount"}
  }},
  {"$project": {
    "_id": 0,
    "customerType": {"$cond": ["$_id", "Repeat Customer", "First-Time Customer"]},
    "totalLoans": 1,
    "overdue": 1,
    "closed": 1,
    "overdueRate": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$overdue", {"$max": ["$totalLoans", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$overdue", {"$max": ["$totalLoans", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    },
    "closureRate": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$closed", {"$max": ["$totalLoans", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$closed", {"$max": ["$totalLoans", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    },
    "avgDPD": {
      "$divide": [
        {"$subtract": [
          {"$multiply": ["$avgDPD", 10]},
          {"$mod": [{"$multiply": ["$avgDPD", 10]}, 1]}
        ]},
        10
      ]
    },
    "avgAmount": {
      "$subtract": [
        "$avgAmount",
        {"$mod": ["$avgAmount", 1]}
      ]
    }
  }}
]

3. Repeat Customer Analysis - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$addFields": {
    "customerSegment": {"$switch": {
      "branches": [
        {"case": {"$eq": [{"$ifNull": ["$totalLoans", 0]}, 0]}, "then": "No Loan"},
        {"case": {"$eq": ["$totalLoans", 1]}, "then": "First-Time"},
        {"case": {"$eq": ["$totalLoans", 2]}, "then": "Repeat (2x)"},
        {"case": {"$gte": ["$totalLoans", 3]}, "then": "Loyal (3x+)"}
      ],
      "default": "No Loan"
    }},
    "segmentOrder": {"$switch": {
      "branches": [
        {"case": {"$eq": [{"$ifNull": ["$totalLoans", 0]}, 0]}, "then": 0},
        {"case": {"$eq": ["$totalLoans", 1]}, "then": 1},
        {"case": {"$eq": ["$totalLoans", 2]}, "then": 2},
        {"case": {"$gte": ["$totalLoans", 3]}, "then": 3}
      ],
      "default": 0
    }}
  }},
  {"$group": {
    "_id": {"segment": "$customerSegment", "order": "$segmentOrder"},
    "count": {"$sum": 1},
    "avgLoans": {"$avg": {"$ifNull": ["$totalLoans", 0]}},
    "avgCibil": {"$avg": {
      "$cond": [{"$gt": [{"$ifNull": ["$cibilScore", 0]}, 0]}, "$cibilScore", null]
    }},
    "avgLTV": {"$avg": {"$ifNull": ["$lifetimeValue", 0]}}
  }},
  {"$project": {
    "_id": 0,
    "segment": "$_id.segment",
    "sortOrder": "$_id.order",
    "count": 1,
    "avgLoans": {
      "$divide": [
        {"$subtract": [
          {"$multiply": ["$avgLoans", 10]},
          {"$mod": [{"$multiply": ["$avgLoans", 10]}, 1]}
        ]},
        10
      ]
    },
    "avgCibil": {
      "$subtract": [
        {"$ifNull": ["$avgCibil", 0]},
        {"$mod": [{"$ifNull": ["$avgCibil", 0]}, 1]}
      ]
    },
    "avgLTV": {
      "$subtract": [
        "$avgLTV",
        {"$mod": ["$avgLTV", 1]}
      ]
    }
  }},
  {"$sort": {"sortOrder": 1}}
]