1. Total Disbursed/Transactions/Avg Disbursement - Disbursement Transactions

[
  {"$match": {"status": "success"}},
  {"$group": {
    "_id": null,
    "totalDisbursed": {"$sum": "$amount"},
    "totalTransactions": {"$sum": 1},
    "avgDisbursement": {"$avg": "$amount"}
  }},
  {"$project": {"_id": 0}}
]

2. Daily Disbursement Trend - Disbursement Transactions

[
  {"$match": {"status": "success"}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m-%d", "date": "$disbursementDate"}},
    "totalDisbursed": {"$sum": "$amount"},
    "count": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"date": "$_id", "totalDisbursed": 1, "count": 1, "_id": 0}}
]

3. Disbursement by Mode - Disbursement Transactions

[
  {"$match": {"status": "success"}},
  {"$group": {"_id": "$disbursementMode", "count": {"$sum": 1}, "totalAmount": {"$sum": "$amount"}}},
  {"$sort": {"totalAmount": -1}},
  {"$project": {"mode": "$_id", "count": 1, "totalAmount": 1, "_id": 0}}
]

4. Disbursement Success/Failure Rate - disbursement transactions

[
  {"$group": {"_id": "$status", "count": {"$sum": 1}, "totalAmount": {"$sum": "$amount"}}},
  {"$project": {"status": "$_id", "count": 1, "totalAmount": 1, "_id": 0}}
]

5. Failed Disbursements Detail - disbursement transactions

[
  {"$match": {"status": {"$in": ["FAILED", "REVERSED"]}}},
  {"$lookup": {"from": "customers", "localField": "customerId", "foreignField": "_id", "as": "customer"}},
  {"$unwind": {"path": "$customer", "preserveNullAndEmptyArrays": true}},
  {"$project": {
    "transactionId": 1,
    "customerName": "$customer.fullName",
    "amount": 1,
    "status": 1,
    "disbursementDate": 1,
    "disbursementMode": 1,
    "bankName": "$bankAccountDetails.bankName"
  }},
  {"$sort": {"disbursementDate": -1}},
  {"$limit": 100}
]

6. Disbursement by Bank - disbursement transactions

[
  {"$match": {"status": "success"}},
  {"$group": {
    "_id": {"$ifNull": ["$bankAccountDetails.bankName", "Unknown"]},
    "count": {"$sum": 1},
    "totalAmount": {"$sum": "$amount"}
  }},
  {"$sort": {"count": -1}},
  {"$limit": 15},
  {"$project": {"bankName": "$_id", "count": 1, "totalAmount": 1, "_id": 0}}
]