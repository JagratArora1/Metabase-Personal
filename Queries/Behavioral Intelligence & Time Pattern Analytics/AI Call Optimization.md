1. AI Call Answer Rate by Hour (IST) - Ai Calls Logs

[
  {
    "$match": { "startedAt": { "$exists": true } }
  },
  {
    "$addFields": {
      "startedAtDate": {
        "$dateFromString": { "dateString": "$startedAt" }
      }
    }
  },
  {
    "$addFields": {
      "istHour": {
        "$hour": { "$add": ["$startedAtDate", 19800000] }
      }
    }
  },
  {
    "$group": {
      "_id": "$istHour",
      "total": { "$sum": 1 },
      "answered": { "$sum": { "$cond": [{ "$eq": ["$status", "ANSWERED"] }, 1, 0] } },
      "noAnswer": { "$sum": { "$cond": [{ "$eq": ["$status", "NO_ANSWER"] }, 1, 0] } },
      "busy": { "$sum": { "$cond": [{ "$eq": ["$status", "BUSY"] }, 1, 0] } }
    }
  },
  {
    "$addFields": {
      "answerRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$answered", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "noAnswerRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$noAnswer", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "busyRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$busy", { "$max": ["$total", 1] }] }, 1000] }, 0.5] } },
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
      "hour_IST": {
        "$concat": [
          { "$cond": [{ "$lt": ["$_id", 10] }, { "$concat": ["0", { "$toString": "$_id" }] }, { "$toString": "$_id" }] },
          ":00"
        ]
      },
      "total": 1,
      "answered": 1,
      "noAnswer": 1,
      "busy": 1,
      "answerRate": 1,
      "noAnswerRate": 1,
      "busyRate": 1,
      "_id": 0
    }
  }
]

2. AI Call Effectiveness by DPD Category - Ai Call Logs

[
  {
    "$match": { "startedAt": { "$exists": true } }
  },
  {
    "$group": {
      "_id": { "$ifNull": ["$dpdCategory", "UNKNOWN"] },
      "totalCalls": { "$sum": 1 },
      "answered": { "$sum": { "$cond": [{ "$eq": ["$status", "ANSWERED"] }, 1, 0] } },
      "notAnswered": { "$sum": { "$cond": [{ "$eq": ["$status", "NO_ANSWER"] }, 1, 0] } },
      "busy": { "$sum": { "$cond": [{ "$eq": ["$status", "BUSY"] }, 1, 0] } },
      "avgDuration": { "$avg": { "$ifNull": ["$duration", 0] } }
    }
  },
  {
    "$addFields": {
      "answerRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$answered", { "$max": ["$totalCalls", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "avgDuration_sec": {
        "$toLong": { "$add": ["$avgDuration", 0.5] }
      },
      "sortOrder": {
        "$switch": {
          "branches": [
            { "case": { "$eq": ["$_id", "CURRENT"] },   "then": 0 },
            { "case": { "$eq": ["$_id", "DPD_1_7"] },   "then": 1 },
            { "case": { "$eq": ["$_id", "DPD_8_15"] },  "then": 2 },
            { "case": { "$eq": ["$_id", "DPD_16_30"] }, "then": 3 },
            { "case": { "$eq": ["$_id", "DPD_31_60"] }, "then": 4 },
            { "case": { "$eq": ["$_id", "DPD_60_PLUS"] }, "then": 5 }
          ],
          "default": 99
        }
      }
    }
  },
  {
    "$sort": { "sortOrder": 1 }
  },
  {
    "$project": {
      "dpdCategory": "$_id",
      "totalCalls": 1,
      "answered": 1,
      "notAnswered": 1,
      "busy": 1,
      "answerRate": 1,
      "avgDuration_sec": 1,
      "_id": 0
    }
  }
]

3. AI Call to Payment Correlation - Ai Call Logs

[
  {
    "$match": {
      "status": "ANSWERED",
      "loanId": { "$exists": true }
    }
  },
  {
    "$addFields": {
      "startedAtDate": {
        "$dateFromString": { "dateString": "$startedAt" }
      }
    }
  },
  {
    "$lookup": {
      "from": "payments",
      "localField": "loanId",
      "foreignField": "loanId",
      "as": "allPayments"
    }
  },
  {
    "$addFields": {
      "followUpPayments": {
        "$filter": {
          "input": { "$ifNull": ["$allPayments", []] },
          "as": "p",
          "cond": {
            "$and": [
              { "$eq": ["$$p.status", "SUCCESS"] },
              { "$gte": ["$$p.paymentDate", "$startedAtDate"] },
              { "$lte": [
                "$$p.paymentDate",
                { "$add": ["$startedAtDate", 172800000] }
              ]}
            ]
          }
        }
      }
    }
  },
  {
    "$group": {
      "_id": null,
      "totalAnsweredCalls": { "$sum": 1 },
      "callsWithPayment48h": {
        "$sum": { "$cond": [{ "$gt": [{ "$size": "$followUpPayments" }, 0] }, 1, 0] }
      },
      "totalPaymentAmount": {
        "$sum": {
          "$reduce": {
            "input": "$followUpPayments",
            "initialValue": 0,
            "in": { "$add": ["$$value", { "$ifNull": ["$$this.amount", 0] }] }
          }
        }
      }
    }
  },
  {
    "$addFields": {
      "conversionRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$callsWithPayment48h", { "$max": ["$totalAnsweredCalls", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "avgPaymentPerCall": {
        "$toLong": {
          "$add": [
            { "$divide": ["$totalPaymentAmount", { "$max": ["$totalAnsweredCalls", 1] }] },
            0.5
          ]
        }
      }
    }
  },
  {
    "$project": {
      "totalAnsweredCalls": 1,
      "callsWithPayment48h": 1,
      "conversionRate": 1,
      "totalPaymentAmount": 1,
      "avgPaymentPerCall": 1,
      "_id": 0
    }
  }
]