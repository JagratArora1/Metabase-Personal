1. Auto-Decision vs Manual Review - Applications

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$cond": [{"$eq": ["$manualReviewRequired", true]}, "Manual Review", "Auto Decision"]},
    "count": {"$sum": 1}
  }},
  {"$project": {"type": "$_id", "count": 1, "_id": 0}}
]

2. BRE Approval Rate by CIBIL Range - Applications

[
 {
  $match: {
   isDeleted: { $ne: true },
   "breHistory.brePulled": true,
   cibilScore: { $gt: 0 }
  }
 },
 {
  $group: {
   _id: {
    $switch: {
     branches: [
      { case: { $lt: ["$cibilScore", 500] }, then: "0-499" },
      { case: { $lt: ["$cibilScore", 600] }, then: "500-599" },
      { case: { $lt: ["$cibilScore", 650] }, then: "600-649" },
      { case: { $lt: ["$cibilScore", 700] }, then: "650-699" },
      { case: { $lt: ["$cibilScore", 750] }, then: "700-749" },
      { case: { $lt: ["$cibilScore", 800] }, then: "750-799" },
      { case: { $lt: ["$cibilScore", 900] }, then: "800-899" }
     ],
     default: "900+"
    }
   },

   total: { $sum: 1 },

   approved: {
    $sum: {
     $cond: [
      { $eq: ["$breHistory.breStatus", "APPROVED"] },
      1,
      0
     ]
    }
   }
  }
 },
 {
  $project: {
   _id: 0,
   cibilRange: "$_id",
   total: 1,
   approved: 1,

   approvalRate: {
    $divide: [
     {
      $floor: {
       $multiply: [
        {
         $divide: [
          "$approved",
          { $cond: [{ $gt: ["$total", 0] }, "$total", 1] }
         ]
        },
        10000
       ]
      }
     },
     100
    ]
   }
  }
 },
 {
  $sort: { cibilRange: 1 }
 }
]

3. BRE Approval Rate by Loan Amount - Applications

[
 {
  $match: {
   isDeleted: { $ne: true },
   "breHistory.brePulled": true,
   requestedLoanAmount: { $gt: 0 }
  }
 },
 {
  $group: {
   _id: {
    $switch: {
     branches: [
      { case: { $lt: ["$requestedLoanAmount", 10000] }, then: "0-9999" },
      { case: { $lt: ["$requestedLoanAmount", 20000] }, then: "10000-19999" },
      { case: { $lt: ["$requestedLoanAmount", 30000] }, then: "20000-29999" },
      { case: { $lt: ["$requestedLoanAmount", 50000] }, then: "30000-49999" },
      { case: { $lt: ["$requestedLoanAmount", 100000] }, then: "50000-99999" },
      { case: { $lt: ["$requestedLoanAmount", 500000] }, then: "100000-499999" }
     ],
     default: "500000+"
    }
   },

   total: { $sum: 1 },

   approved: {
    $sum: {
     $cond: [
      { $eq: ["$breHistory.breStatus", "APPROVED"] },
      1,
      0
     ]
    }
   }
  }
 },
 {
  $project: {
   _id: 0,
   amountRange: "$_id",
   total: 1,
   approved: 1,

   approvalRate: {
    $divide: [
     {
      $floor: {
       $multiply: [
        {
         $divide: [
          "$approved",
          { $cond: [{ $gt: ["$total", 0] }, "$total", 1] }
         ]
        },
        10000
       ]
      }
     },
     100
    ]
   }
  }
 },
 {
  $sort: { amountRange: 1 }
 }
]

4. BRE Pull Volume by Day - Applications

[
  {
    "$match": {
      "breHistory.brePulled": true,
      "breHistory.brePulledAt": { "$exists": true },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": {
        "$dateToString": {
          "format": "%Y-%m-%d",
          "date": "$breHistory.brePulledAt"
        }
      },
      "brePulls": { "$sum": 1 }
    }
  },
  { "$sort": { "_id": 1 } },
  {
    "$project": {
      "date": "$_id",
      "brePulls": 1,
      "_id": 0
    }
  }
]

5. BRE Status Breakdown - Applications

[
  {"$match": {"breHistory.brePulled": true, "isDeleted": {"$ne": true}}},
  {"$group": {"_id": {"$ifNull": ["$breHistory.breStatus", "NULL/PENDING"]}, "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"breStatus": "$_id", "count": 1, "_id": 0}}
]

6. BSA-BRE Combined Funnel - Applications

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,

      "Application Created": { "$sum": 1 },

      "BRE Pulled": {
        "$sum": {
          "$cond": [
            { "$eq": ["$breHistory.brePulled", true] },
            1,
            0
          ]
        }
      },

      "BSA Initiated": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$breHistory.brePulled", true] },
                { "$eq": ["$breHistory.bsaInitiated", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "BSA Completed": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$breHistory.brePulled", true] },
                { "$eq": ["$breHistory.bsaInitiated", true] },
                { "$eq": ["$breHistory.bsaStatus", "COMPLETED"] }
              ]
            },
            1,
            0
          ]
        }
      },

      "BRE Approved": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$breHistory.brePulled", true] },
                { "$eq": ["$breHistory.bsaStatus", "COMPLETED"] },
                { "$eq": ["$breHistory.breStatus", "APPROVED"] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Proceed To Bank": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$breHistory.brePulled", true] },
                { "$eq": ["$breHistory.bsaStatus", "COMPLETED"] },
                { "$eq": ["$breHistory.breStatus", "PROCEED TO BANK"] }
              ]
            },
            1,
            0
          ]
        }
      }
    }
  },
  {
    "$project": {
      "stages": [
        { "stage": "Application Created", "count": "$Application Created" },
        { "stage": "BRE Pulled", "count": "$BRE Pulled" },
        { "stage": "BSA Initiated", "count": "$BSA Initiated" },
        { "stage": "BSA Completed", "count": "$BSA Completed" },
        { "stage": "BRE Approved", "count": "$BRE Approved" },
        { "stage": "Proceed To Bank", "count": "$Proceed To Bank" }
      ]
    }
  },
  { "$unwind": "$stages" },
  {
    "$project": {
      "_id": 0,
      "Stage": "$stages.stage",
      "Count": "$stages.count"
    }
  }
]

7. BSA Status Funnel - Applications

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,

      "applications": {
        "$sum": 1
      },

      "brePulled": {
        "$sum": {
          "$cond": [
            { "$eq": ["$breHistory.brePulled", true] },
            1,
            0
          ]
        }
      },

      "bsaInitiated": {
        "$sum": {
          "$cond": [
            { "$eq": ["$breHistory.bsaInitiated", true] },
            1,
            0
          ]
        }
      },

      "bsaSuccess": {
        "$sum": {
          "$cond": [
            { "$eq": ["$breHistory.bsaStatus", "COMPLETED"] },
            1,
            0
          ]
        }
      },

      "disbursed": {
        "$sum": {
          "$cond": [
            { "$eq": ["$status", "DISBURSED"] },
            1,
            0
          ]
        }
      }
    }
  },
  {
    "$project": {
      "stages": [
        { "stage": "Application Created", "count": "$applications", "order": 1 },
        { "stage": "BRE Pulled", "count": "$brePulled", "order": 2 },
        { "stage": "BSA Initiated", "count": "$bsaInitiated", "order": 3 },
        { "stage": "BSA Success", "count": "$bsaSuccess", "order": 4 },
        { "stage": "Loan Disbursed", "count": "$disbursed", "order": 5 }
      ]
    }
  },
  { "$unwind": "$stages" },
  {
    "$project": {
      "_id": 0,
      "stage": "$stages.stage",
      "count": "$stages.count",
      "order": "$stages.order"
    }
  },
  {
    "$sort": { "order": 1 }
  }
]

8. CIBIL Score Distribution - Applications

[
  {
    $match: {
      cibilScore: { $gt: 0 },
      isDeleted: { $ne: true }
    }
  },
  {
    $addFields: {
      cibilRange: {
        $switch: {
          branches: [
            { case: { $lt: ["$cibilScore", 300] }, then: "0-299" },
            { case: { $lt: ["$cibilScore", 500] }, then: "300-499" },
            { case: { $lt: ["$cibilScore", 600] }, then: "500-599" },
            { case: { $lt: ["$cibilScore", 650] }, then: "600-649" },
            { case: { $lt: ["$cibilScore", 700] }, then: "650-699" },
            { case: { $lt: ["$cibilScore", 750] }, then: "700-749" },
            { case: { $lt: ["$cibilScore", 800] }, then: "750-799" },
            { case: { $lt: ["$cibilScore", 900] }, then: "800-899" }
          ],
          default: "900+"
        }
      }
    }
  },
  {
    $group: {
      _id: "$cibilRange",

      count: { $sum: 1 },

      approved: {
        $sum: {
          $cond: [
            { $in: ["$status", ["APPROVED", "DISBURSED", "CLOSED"]] },
            1,
            0
          ]
        }
      },

      rejected: {
        $sum: {
          $cond: [
            { $eq: ["$status", "REJECTED"] },
            1,
            0
          ]
        }
      }
    }
  },
  {
    $project: {
      _id: 0,
      cibilRange: "$_id",
      count: 1,
      approved: 1,
      rejected: 1
    }
  },
  {
    $sort: {
      cibilRange: 1
    }
  }
]

9. Fraud Score Distribution - Applications

[
 {
  $match: {
   isDeleted: { $ne: true },
   fraudScore: { $gt: 0 }
  }
 },
 {
  $group: {
   _id: {
    $switch: {
     branches: [
      { case: { $lt: ["$fraudScore", 10] }, then: { label: "0-9", order: 1 } },
      { case: { $lt: ["$fraudScore", 20] }, then: { label: "10-19", order: 2 } },
      { case: { $lt: ["$fraudScore", 30] }, then: { label: "20-29", order: 3 } },
      { case: { $lt: ["$fraudScore", 40] }, then: { label: "30-39", order: 4 } },
      { case: { $lt: ["$fraudScore", 50] }, then: { label: "40-49", order: 5 } },
      { case: { $lt: ["$fraudScore", 60] }, then: { label: "50-59", order: 6 } },
      { case: { $lt: ["$fraudScore", 70] }, then: { label: "60-69", order: 7 } },
      { case: { $lt: ["$fraudScore", 80] }, then: { label: "70-79", order: 8 } },
      { case: { $lt: ["$fraudScore", 90] }, then: { label: "80-89", order: 9 } },
      { case: { $lt: ["$fraudScore", 100] }, then: { label: "90-99", order: 10 } }
     ],
     default: { label: "100+", order: 11 }
    }
   },
   count: { $sum: 1 }
  }
 },
 {
  $sort: {
   "_id.order": 1
  }
 },
 {
  $project: {
   _id: 0,
   fraudBucket: "$_id.label",
   count: 1
  }
 }
]

10. Risk Score Distribution - Applications

[
 {
  $match: {
   isDeleted: { $ne: true },
   riskScore: { $gt: 0 }
  }
 },
 {
  $group: {
   _id: {
    $switch: {
     branches: [
      { case: { $lt: ["$riskScore", 10] }, then: { label: "0-9", order: 1 } },
      { case: { $lt: ["$riskScore", 20] }, then: { label: "10-19", order: 2 } },
      { case: { $lt: ["$riskScore", 30] }, then: { label: "20-29", order: 3 } },
      { case: { $lt: ["$riskScore", 40] }, then: { label: "30-39", order: 4 } },
      { case: { $lt: ["$riskScore", 50] }, then: { label: "40-49", order: 5 } },
      { case: { $lt: ["$riskScore", 60] }, then: { label: "50-59", order: 6 } },
      { case: { $lt: ["$riskScore", 70] }, then: { label: "60-69", order: 7 } },
      { case: { $lt: ["$riskScore", 80] }, then: { label: "70-79", order: 8 } },
      { case: { $lt: ["$riskScore", 90] }, then: { label: "80-89", order: 9 } },
      { case: { $lt: ["$riskScore", 100] }, then: { label: "90-99", order: 10 } }
     ],
     default: { label: "100+", order: 11 }
    }
   },
   count: { $sum: 1 }
  }
 },
 {
  $sort: {
   "_id.order": 1
  }
 },
 {
  $project: {
   _id: 0,
   riskBucket: "$_id.label",
   count: 1
  }
 }
]