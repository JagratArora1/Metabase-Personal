1. Total Revenue - Payments (EMI Income, Foreclosure Income, Late Charges Income, Settlement Income, Bounce Charges Income, Part Payment Income)

[
  {"$match": {"status": "SUCCESS", "paymentType": {"$ne": "DISBURSEMENT"}}},
  {"$group": {
    "_id": null,
    "totalRevenue": {"$sum": "$amount"},
    "EMI Income": {"$sum": {"$cond": [{"$eq": ["$paymentType", "EMI"]}, "$amount", 0]}},
    "foreclosureIncome": {"$sum": {"$cond": [{"$eq": ["$paymentType", "FORECLOSURE"]}, "$amount", 0]}},
    "lateChargesIncome": {"$sum": {"$cond": [{"$eq": ["$paymentType", "LATE_CHARGES"]}, "$amount", 0]}},
    "settlementIncome": {"$sum": {"$cond": [{"$eq": ["$paymentType", "SETTLEMENT"]}, "$amount", 0]}},
    "bounceChargesIncome": {"$sum": {"$cond": [{"$eq": ["$paymentType", "BOUNCE_CHARGES"]}, "$amount", 0]}},
    "partPaymentIncome": {"$sum": {"$cond": [{"$eq": ["$paymentType", "PART_PAYMENT"]}, "$amount", 0]}}
  }},
  {"$project": {"_id": 0}}
]

2. Monthly Revenue Trend - Payments

[
  {
    "$match": {
      "status": "SUCCESS",
      "paymentType": { "$ne": "DISBURSEMENT" }
    }
  },
  {
    "$project": {
      "month": {
        "$dateToString": {
          "format": "%Y-%m",
          "date": "$paymentDate"
        }
      },
      "paymentType": 1,
      "amount": 1
    }
  },
  {
    "$group": {
      "_id": {
        "month": "$month",
        "paymentType": "$paymentType"
      },
      "amount": { "$sum": "$amount" }
    }
  },
  {
    "$project": {
      "_id": 0,
      "month": "$_id.month",
      "paymentType": "$_id.paymentType",
      "amount": 1
    }
  },
  {
    "$sort": {
      "month": 1
    }
  }
]

3. Revenue by Type (Pie) - Payments

[
  {"$match": {"status": "SUCCESS", "paymentType": {"$ne": "DISBURSEMENT"}}},
  {"$group": {"_id": "$paymentType", "totalAmount": {"$sum": "$amount"}, "count": {"$sum": 1}}},
  {"$sort": {"totalAmount": -1}},
  {"$project": {"revenueType": "$_id", "totalAmount": 1, "count": 1, "_id": 0}}
]

4. Processing Fee Revenue - Applications 

[
  {"$match": {"status": "DISBURSED", "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m", "date": "$disbursementDate"}},
    "totalProcessingFee": {"$sum": "$processingFee"},
    "totalGST": {"$sum": "$gstOnProcessingFee"},
    "loanCount": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"month": "$_id", "totalProcessingFee": 1, "totalGST": 1, "loanCount": 1, "_id": 0}}
]

5. Interest Income Analysis - Applications

[
  {"$match": {"status": "DISBURSED", "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m", "date": "$disbursementDate"}},
    "totalInterestExpected": {"$sum": "$totalInterest"},
    "totalRepaymentExpected": {"$sum": "$totalRepayment"},
    "totalDisbursed": {"$sum": "$netDisbursalAmount"}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"month": "$_id", "totalInterestExpected": 1, "totalRepaymentExpected": 1, "totalDisbursed": 1, "_id": 0}}
]

6. Average Revenue Per Loan - Applications

[
  {"$match": {"status": {"$in": ["DISBURSED", "CLOSED"]}, "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": null,
    "avgProcessingFee": {"$avg": "$processingFee"},
    "avgGST": {"$avg": "$gstOnProcessingFee"},
    "avgInterest": {"$avg": "$totalInterest"},
    "avgTotalRevenue": {"$avg": {"$add": [
      {"$ifNull": ["$processingFee", 0]},
      {"$ifNull": ["$gstOnProcessingFee", 0]},
      {"$ifNull": ["$totalInterest", 0]}
    ]}}
  }},
  {"$project": {"_id": 0}}
]

7. Revenue vs Disbursement Ratio - Payments

[
  {"$match": {"status": "SUCCESS"}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m", "date": "$paymentDate"}},
    "disbursements": {"$sum": {"$cond": [{"$eq": ["$paymentType", "DISBURSEMENT"]}, "$amount", 0]}},
    "collections": {"$sum": {"$cond": [{"$ne": ["$paymentType", "DISBURSEMENT"]}, "$amount", 0]}}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {
    "month": "$_id",
    "disbursements": 1,
    "collections": 1,
    "ratio": {"$cond": [{"$gt": ["$disbursements", 0]}, {"$divide": ["$collections", "$disbursements"]}, 0]},
    "_id": 0
  }}
]