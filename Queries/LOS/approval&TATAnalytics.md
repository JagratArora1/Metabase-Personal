1. Applications Breaching SLA - Applications

[
  {
    "$match": {
      "status": { "$in": ["PENDING", "PROCESSING", "APPROVED", "PENDING_DISBURSED"] },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "applicationNumber": 1,
      "status": 1,
      "requestedLoanAmount": 1,
      "hoursAgo": {
        "$divide": [
          { "$subtract": [ new Date(), "$createdAt" ] },
          3600000
        ]
      }
    }
  },
  {
    "$match": {
      "hoursAgo": { "$gt": 24 }
    }
  },
  {
    "$sort": { "hoursAgo": -1 }
  },
  {
    "$limit": 50
  }
]

2. Average TAT - Application to Approval - Applications

[
 {
  $match: {
   isDeleted: { $ne: true },
   approvedDate: { $ne: null },
   createdAt: { $ne: null }
  }
 },
 {
  $match: {
   $expr: { $gte: ["$approvedDate", "$createdAt"] }
  }
 },
 {
  $project: {
   tatHours: {
    $divide: [
     { $subtract: ["$approvedDate", "$createdAt"] },
     3600000
    ]
   }
  }
 },
 {
  $match: {
   tatHours: { $ne: null }
  }
 },
 {
  $group: {
   _id: null,
   avgTAT_hours: { $avg: "$tatHours" },
   minTAT_hours: { $min: "$tatHours" },
   maxTAT_hours: { $max: "$tatHours" }
  }
 }
]

3. Average TAT - Approval to Disbursement - Applications

[
  {
    "$match": {
      "approvedDate": { "$exists": true },
      "disbursementDate": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "tatHours": {
         $divide: [
          {
           $abs: {
            $subtract: ["$disbursementDate", "$approvedDate"]
           }
          },
          3600000
        ]
      }
    }
  },
  {
    "$group": {
      "_id": null,
      "avgTAT_hours": { "$avg": "$tatHours" }
    }
  }
]

4. End-to-End TAT (Apply to Disburse) - Applications

[
  {"$match": {
    "disbursementDate": {"$exists": true},
    "createdAt": {"$exists": true},
    "isDeleted": {"$ne": true}
  }},
  {"$project": {
    "tatHours": {"$divide": [{"$subtract": ["$disbursementDate", "$createdAt"]}, 3600000]}
  }},
  {"$group": {
    "_id": null,
    "avgTAT_hours": {"$avg": "$tatHours"}
  }}
]

5. TAT Distribution Histogram - Applications 

[
  {
    "$match": {
      "disbursementDate": { "$exists": true, "$ne": null },
      "createdAt": { "$exists": true, "$ne": null },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "tatHours": {
        "$divide": [
          { "$subtract": ["$disbursementDate", "$createdAt"] },
          3600000
        ]
      }
    }
  },
  {
    "$project": {
      "bucket": {
        "$switch": {
          "branches": [
            { "case": { "$lte": ["$tatHours", 1] }, "then": { "label": "0-1 hr", "order": 1 } },
            { "case": { "$lte": ["$tatHours", 2] }, "then": { "label": "1-2 hr", "order": 2 } },
            { "case": { "$lte": ["$tatHours", 4] }, "then": { "label": "2-4 hr", "order": 3 } },
            { "case": { "$lte": ["$tatHours", 8] }, "then": { "label": "4-8 hr", "order": 4 } },
            { "case": { "$lte": ["$tatHours", 12] }, "then": { "label": "8-12 hr", "order": 5 } },
            { "case": { "$lte": ["$tatHours", 24] }, "then": { "label": "12-24 hr", "order": 6 } },
            { "case": { "$lte": ["$tatHours", 48] }, "then": { "label": "24-48 hr", "order": 7 } },
            { "case": { "$lte": ["$tatHours", 72] }, "then": { "label": "48-72 hr", "order": 8 } },
            { "case": { "$lte": ["$tatHours", 168] }, "then": { "label": "72-168 hr", "order": 9 } }
          ],
          "default": { "label": "168+ hr", "order": 10 }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$bucket",
      "count": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "bucket": "$_id.label",
      "count": 1
    }
  },
  {
    "$sort": { "order": 1 }
  }
]

6. TAT Trend (Weekly Average) - Applications

[
  {
    "$match": {
      "disbursementDate": { "$exists": true },
      "createdAt": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "year": { "$isoWeekYear": "$createdAt" },
      "week": { "$isoWeek": "$createdAt" },
      "tatHours": {
        "$divide": [
          { "$subtract": ["$disbursementDate", "$createdAt"] },
          3600000
        ]
      }
    }
  },
  {
    "$group": {
      "_id": {
        "year": "$year",
        "week": "$week"
      },
      "avgTATHours": { "$avg": "$tatHours" },
      "disbursements": { "$sum": 1 }
    }
  },
  {
    "$sort": {
      "_id.year": 1,
      "_id.week": 1
    }
  },
  {
    "$project": {
      "_id": 0,
      "year": "$_id.year",
      "week": "$_id.week",
      "avgTATHours": 1,
      "disbursements": 1
    }
  }
]