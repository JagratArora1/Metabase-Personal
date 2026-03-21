1. Loan Product Profitability Matrix - Loans

[
  {"$match": {"isDeleted": {"$ne": true}, "tenure": {"$in": [7, 15, 30]}}},
  {"$addFields": {
    "amountGroup": {"$switch": {
      "branches": [
        {"case": {"$lt": ["$principalAmount", 10000]}, "then": "Small (<10K)"},
        {"case": {"$lt": ["$principalAmount", 20000]}, "then": "Medium (10-20K)"}
      ],
      "default": "Large (20K+)"
    }}
  }},
  {"$group": {
    "_id": {"tenure": "$tenure", "amount": "$amountGroup"},
    "count": {"$sum": 1},
    "totalDisbursed": {"$sum": "$disbursementAmount"},
    "totalFees": {"$sum": {"$ifNull": ["$processingFee", 0]}},
    "totalInterest": {"$sum": {"$ifNull": ["$interestAmount", 0]}},
    "totalLateCharges": {"$sum": {"$ifNull": ["$lateCharges", 0]}},
    "totalOutstanding": {"$sum": "$totalOutstanding"},
    "overdueCount": {"$sum": {"$cond": [{"$in": ["$status", ["OVERDUE", "NPA"]]}, 1, 0]}},
    "closedCount": {"$sum": {"$cond": [{"$eq": ["$status", "CLOSED"]}, 1, 0]}}
  }},
  {"$match": {"count": {"$gte": 5}}},
  {"$addFields": {
    "totalRevenue": {"$add": ["$totalFees", "$totalInterest", "$totalLateCharges"]},
    "overdueRate": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$overdueCount", {"$max": ["$count", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$overdueCount", {"$max": ["$count", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    },
    "closureRate": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$closedCount", {"$max": ["$count", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$closedCount", {"$max": ["$count", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    }
  }},
  {"$project": {
    "_id": 0,
    "tenure": {"$concat": [{"$toString": "$_id.tenure"}, " days"]},
    "amountGroup": "$_id.amount",
    "count": 1,
    "totalDisbursed": 1,
    "totalRevenue": 1,
    "revenuePerLoan": {
      "$subtract": [
        {"$divide": ["$totalRevenue", {"$max": ["$count", 1]}]},
        {"$mod": [{"$divide": ["$totalRevenue", {"$max": ["$count", 1]}]}, 1]}
      ]
    },
    "yield": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    },
    "overdueRate": 1,
    "closureRate": 1,
    "riskAdjustedYield": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [
            {"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]},
            {"$subtract": [1, {"$divide": ["$overdueCount", {"$max": ["$count", 1]}]}]},
            100
          ]},
          {"$mod": [
            {"$multiply": [
              {"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]},
              {"$subtract": [1, {"$divide": ["$overdueCount", {"$max": ["$count", 1]}]}]},
              100
            ]},
            1
          ]}
        ]},
        10
      ]
    }
  }},
  {"$sort": {"riskAdjustedYield": -1}}
]

2. Average Revenue Per Loan (Breakdown) - Loans

[
  {"$match": {"isDeleted": {"$ne": true}, "status": {"$in": ["ACTIVE", "OVERDUE", "CLOSED"]}}},
  {"$group": {
    "_id": "$status",
    "count": {"$sum": 1},
    "avgPrincipal": {"$avg": "$principalAmount"},
    "avgProcessingFee": {"$avg": {"$ifNull": ["$processingFee", 0]}},
    "avgGST": {"$avg": {"$ifNull": ["$gstAmount", 0]}},
    "avgInterest": {"$avg": {"$ifNull": ["$interestAmount", 0]}},
    "avgLateCharges": {"$avg": {"$ifNull": ["$lateCharges", 0]}},
    "avgTotalRepayment": {"$avg": {"$ifNull": ["$totalRepayment", 0]}},
    "avgDisbursement": {"$avg": {"$ifNull": ["$disbursementAmount", 0]}}
  }},
  {"$addFields": {
    "avgTotalRevenue": {"$add": [
      "$avgProcessingFee",
      "$avgGST",
      "$avgInterest",
      "$avgLateCharges"
    ]},
    "avgRevenuePerRupeeLent": {"$cond": {
      "if": {"$gt": ["$avgDisbursement", 0]},
      "then": {
        "$divide": [
          {"$subtract": [
            {"$multiply": [{"$divide": [{"$add": ["$avgProcessingFee", "$avgInterest", "$avgLateCharges"]}, "$avgDisbursement"]}, 1000]},
            {"$mod": [{"$multiply": [{"$divide": [{"$add": ["$avgProcessingFee", "$avgInterest", "$avgLateCharges"]}, "$avgDisbursement"]}, 1000]}, 1]}
          ]},
          1000
        ]
      },
      "else": 0
    }}
  }},
  {"$project": {
    "_id": 0,
    "status": "$_id",
    "loanCount": "$count",
    "avgPrincipal": {"$subtract": ["$avgPrincipal", {"$mod": ["$avgPrincipal", 1]}]},
    "avgProcessingFee": {"$subtract": ["$avgProcessingFee", {"$mod": ["$avgProcessingFee", 1]}]},
    "avgGST": {"$subtract": ["$avgGST", {"$mod": ["$avgGST", 1]}]},
    "avgInterest": {"$subtract": ["$avgInterest", {"$mod": ["$avgInterest", 1]}]},
    "avgLateCharges": {"$subtract": ["$avgLateCharges", {"$mod": ["$avgLateCharges", 1]}]},
    "avgTotalRevenue": {"$subtract": ["$avgTotalRevenue", {"$mod": ["$avgTotalRevenue", 1]}]},
    "avgDisbursement": {"$subtract": ["$avgDisbursement", {"$mod": ["$avgDisbursement", 1]}]},
    "revenuePerRupeeLent": "$avgRevenuePerRupeeLent"
  }},
  {"$sort": {"status": 1}}
]

3. Monthly Profit/Loss Proxy - Loans

[
  {"$match": {"disbursementDate": {"$exists": true}, "isDeleted": {"$ne": true}}},
  {"$addFields": {
    "disbMonth": {"$dateToString": {"format": "%Y-%m", "date": "$disbursementDate"}}
  }},
  {"$group": {
    "_id": "$disbMonth",
    "loanCount": {"$sum": 1},
    "totalDisbursed": {"$sum": "$disbursementAmount"},
    "totalProcessingFee": {"$sum": {"$ifNull": ["$processingFee", 0]}},
    "totalGST": {"$sum": {"$ifNull": ["$gstAmount", 0]}},
    "totalInterest": {"$sum": {"$ifNull": ["$interestAmount", 0]}},
    "totalLateCharges": {"$sum": {"$ifNull": ["$lateCharges", 0]}},
    "totalCollected": {"$sum": {"$ifNull": ["$totalRepayment", 0]}},
    "totalOutstanding": {"$sum": "$totalOutstanding"},
    "overdueLoans": {"$sum": {"$cond": [{"$in": ["$status", ["OVERDUE", "NPA"]]}, 1, 0]}}
  }},
  {"$addFields": {
    "totalRevenue": {"$add": ["$totalProcessingFee", "$totalInterest", "$totalLateCharges"]},
    "overdueRate": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$overdueLoans", {"$max": ["$loanCount", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$overdueLoans", {"$max": ["$loanCount", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    }
  }},
  {"$project": {
    "_id": 0,
    "month": "$_id",
    "loanCount": 1,
    "totalDisbursed": 1,
    "totalRevenue": 1,
    "totalProcessingFee": 1,
    "totalInterest": 1,
    "totalLateCharges": 1,
    "totalOutstanding": 1,
    "overdueRate": 1,
    "revenueToDisburseRatio": {
      "$divide": [
        {"$subtract": [
          {"$multiply": [{"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]}, 1000]},
          {"$mod": [{"$multiply": [{"$divide": ["$totalRevenue", {"$max": ["$totalDisbursed", 1]}]}, 1000]}, 1]}
        ]},
        10
      ]
    }
  }},
  {"$sort": {"month": 1}}
]