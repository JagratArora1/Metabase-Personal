1. Day-of-Week Payment Pattern - Payments

[
  {
    "$match": {
      "status": "SUCCESS",
      "paymentType": { "$ne": "DISBURSEMENT" }
    }
  },
  {
    "$group": {
      "_id": { "$dayOfWeek": "$paymentDate" },
      "count": { "$sum": 1 },
      "totalAmount": { "$sum": "$amount" },
      "avgAmount": { "$avg": "$amount" }
    }
  },
  {
    "$addFields": {
      "avgAmount": {
        "$toLong": { "$add": ["$avgAmount", 0.5] }
      },
      "dayOfWeek": {
        "$switch": {
          "branches": [
            { "case": { "$eq": ["$_id", 1] }, "then": "1 Sunday" },
            { "case": { "$eq": ["$_id", 2] }, "then": "2 Monday" },
            { "case": { "$eq": ["$_id", 3] }, "then": "3 Tuesday" },
            { "case": { "$eq": ["$_id", 4] }, "then": "4 Wednesday" },
            { "case": { "$eq": ["$_id", 5] }, "then": "5 Thursday" },
            { "case": { "$eq": ["$_id", 6] }, "then": "6 Friday" },
            { "case": { "$eq": ["$_id", 7] }, "then": "7 Saturday" }
          ],
          "default": "0_Unknown"
        }
      }
    }
  },
  {
    "$sort": { "_id": 1 }
  },
  {
    "$project": {
      "dayOfWeek": 1,
      "count": 1,
      "totalAmount": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]

2. Payment Mode vs Time-of-Day Pattern - Payments

[
  {
    "$match": {
      "status": "SUCCESS",
      "paymentType": { "$ne": "DISBURSEMENT" }
    }
  },
  {
    "$addFields": {
      "istHour": {
        "$hour": { "$add": ["$paymentDate", 19800000] }
      }
    }
  },
  {
    "$addFields": {
      "hourBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$istHour", 6] },  "then": "00-06 Night" },
            { "case": { "$lt": ["$istHour", 10] }, "then": "06-10 Morning" },
            { "case": { "$lt": ["$istHour", 14] }, "then": "10-14 Midday" },
            { "case": { "$lt": ["$istHour", 18] }, "then": "14-18 Afternoon" },
            { "case": { "$lt": ["$istHour", 22] }, "then": "18-22 Evening" }
          ],
          "default": "F: 22-00 Late Night"
        }
      }
    }
  },
  {
    "$group": {
      "_id": {
        "timeBucket": "$hourBucket",
        "mode": { "$ifNull": ["$paymentMode", "UNKNOWN"] }
      },
      "count": { "$sum": 1 },
      "totalAmount": { "$sum": "$amount" }
    }
  },
  {
    "$sort": { "_id.timeBucket": 1, "_id.mode": 1 }
  },
  {
    "$project": {
      "timeBucket": "$_id.timeBucket",
      "paymentMode": "$_id.mode",
      "count": 1,
      "totalAmount": 1,
      "_id": 0
    }
  }
]

3. Hour-of-Day Payment Pattern (IST) - Payments

[
  {
    "$match": {
      "status": "SUCCESS",
      "paymentType": { "$ne": "DISBURSEMENT" }
    }
  },
  {
    "$addFields": {
      "istHour": {
        "$hour": { "$add": ["$paymentDate", 19800000] }
      }
    }
  },
  {
    "$group": {
      "_id": "$istHour",
      "count": { "$sum": 1 },
      "totalAmount": { "$sum": "$amount" },
      "avgAmount": { "$avg": "$amount" }
    }
  },
  {
    "$addFields": {
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
      "hour_IST": {
        "$concat": [
          { "$cond": [{ "$lt": ["$_id", 10] }, { "$concat": ["0", { "$toString": "$_id" }] }, { "$toString": "$_id" }] },
          ":00"
        ]
      },
      "count": 1,
      "totalAmount": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]

4. Time from Reminder to Payment (Reminder Effectiveness) - Payments 

[
  {
    "$match": {
      "status": "SUCCESS",
      "paymentType": { "$ne": "DISBURSEMENT" },
      "paymentDate": { "$exists": true }
    }
  },
  {
    "$lookup": {
      "from": "loans",
      "localField": "loanId",
      "foreignField": "_id",
      "as": "loan"
    }
  },
  {
    "$unwind": { "path": "$loan", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "daysToDue": {
        "$divide": [
          { "$subtract": ["$paymentDate", { "$ifNull": ["$loan.dueDate", "$paymentDate"] }] },
          86400000
        ]
      }
    }
  },
  {
    "$addFields": {
      "paymentTiming": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$daysToDue", -7] },  "then": "A: 7+ days early" },
            { "case": { "$lt": ["$daysToDue", -3] },  "then": "B: 3-7 days early" },
            { "case": { "$lt": ["$daysToDue", -1] },  "then": "C: 1-3 days early" },
            { "case": { "$lte": ["$daysToDue", 0] },  "then": "D: On due date" },
            { "case": { "$lte": ["$daysToDue", 3] },  "then": "E: 1-3 days late" },
            { "case": { "$lte": ["$daysToDue", 7] },  "then": "F: 3-7 days late" },
            { "case": { "$lte": ["$daysToDue", 15] }, "then": "G: 7-15 days late" }
          ],
          "default": "H: 15+ days late"
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$paymentTiming",
      "count": { "$sum": 1 },
      "totalAmount": { "$sum": "$amount" },
      "avgAmount": { "$avg": "$amount" }
    }
  },
  {
    "$addFields": {
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
      "paymentTiming": "$_id",
      "count": 1,
      "totalAmount": 1,
      "avgAmount": 1,
      "_id": 0
    }
  }
]