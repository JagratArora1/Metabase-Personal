1. Total Customers - Customers (Same query for Active Customers, Blocked Customers, KYC Completed, with Active Loans, Avg Cibil Score)

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,
      "totalCustomers": { "$sum": 1 },
      "activeCustomers": {
        "$sum": {
          "$cond": [
            { "$eq": ["$isActive", true] },
            1,
            0
          ]
        }
      },
      "blockedCustomers": {
        "$sum": {
          "$cond": [
            { "$eq": ["$isBlocked", true] },
            1,
            0
          ]
        }
      },
      "kycCompleted": {
        "$sum": {
          "$cond": [
            { "$eq": ["$kycStatus", "SUCCESS"] },
            1,
            0
          ]
        }
      },
      "withActiveLoans": {
        "$sum": {
          "$cond": [
            { "$gt": ["$activeLoanCount", 0] },
            1,
            0
          ]
        }
      },
      "avgCibilScore": {
        "$avg": {
          "$cond": [
            { "$gt": ["$cibilScore", 0] },
            "$cibilScore",
            null
          ]
        }
      }
    }
  },
  {
    "$project": {
      "_id": 0
    }
  }
]

2. CIBIL Score Distribution - Customers

[
  {
    "$match": {
      "cibilScore": { "$gt": 0 },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "bucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$cibilScore", 300] }, "then": { "label": "<300", "order": 1 } },
            { "case": { "$lt": ["$cibilScore", 400] }, "then": { "label": "300-400", "order": 2 } },
            { "case": { "$lt": ["$cibilScore", 500] }, "then": { "label": "400-500", "order": 3 } },
            { "case": { "$lt": ["$cibilScore", 550] }, "then": { "label": "500-550", "order": 4 } },
            { "case": { "$lt": ["$cibilScore", 600] }, "then": { "label": "550-600", "order": 5 } },
            { "case": { "$lt": ["$cibilScore", 650] }, "then": { "label": "600-650", "order": 6 } },
            { "case": { "$lt": ["$cibilScore", 700] }, "then": { "label": "650-700", "order": 7 } },
            { "case": { "$lt": ["$cibilScore", 750] }, "then": { "label": "700-750", "order": 8 } },
            { "case": { "$lt": ["$cibilScore", 800] }, "then": { "label": "750-800", "order": 9 } },
            { "case": { "$lt": ["$cibilScore", 850] }, "then": { "label": "800-850", "order": 10 } },
            { "case": { "$lt": ["$cibilScore", 900] }, "then": { "label": "850-900", "order": 11 } }
          ],
          "default": { "label": "900+", "order": 12 }
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
      "cibilRange": "$_id.label",
      "order": "$_id.order",
      "count": 1
    }
  },
  {
    "$sort": { "order": 1 }
  }
]

3. Employment Type Distribution - Customers

[
  {
    $match: {
      isDeleted: { $ne: true }
    }
  },
  {
    $project: {
      employmentType: {
        $cond: [
          {
            $or: [
              { $eq: ["$employmentType", null] },
              { $not: ["$employmentType"] }
            ]
          },
          "Unknown",
          "$employmentType"
        ]
      }
    }
  },
  {
    $group: {
      _id: "$employmentType",
      count: { $sum: 1 }
    }
  },
  {
    $sort: { count: -1 }
  },
  {
    $project: {
      employmentType: "$_id",
      count: 1,
      _id: 0
    }
  }
]

4. Gender Distribution - Customers

[
  {
    $match: {
      isDeleted: { $ne: true }
    }
  },
  {
    $project: {
      gender: {
        $cond: [
          {
            $or: [
              { $eq: ["$gender", null] },
              { $eq: ["$gender", ""] },
              { $not: ["$gender"] }
            ]
          },
          "Unknown",
          "$gender"
        ]
      }
    }
  },
  {
    $group: {
      _id: "$gender",
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      gender: "$_id",
      count: 1,
      _id: 0
    }
  }
]

5. Monthly Income Distribution - Customers

[
  {
    "$match": {
      "monthlyIncome": { "$gt": 0 },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "incomeBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$monthlyIncome", 10000] }, "then": { "label": "<10k", "order": 1 } },
            { "case": { "$lt": ["$monthlyIncome", 15000] }, "then": { "label": "10k-15k", "order": 2 } },
            { "case": { "$lt": ["$monthlyIncome", 20000] }, "then": { "label": "15k-20k", "order": 3 } },
            { "case": { "$lt": ["$monthlyIncome", 25000] }, "then": { "label": "20k-25k", "order": 4 } },
            { "case": { "$lt": ["$monthlyIncome", 30000] }, "then": { "label": "25k-30k", "order": 5 } },
            { "case": { "$lt": ["$monthlyIncome", 40000] }, "then": { "label": "30k-40k", "order": 6 } },
            { "case": { "$lt": ["$monthlyIncome", 50000] }, "then": { "label": "40k-50k", "order": 7 } },
            { "case": { "$lt": ["$monthlyIncome", 75000] }, "then": { "label": "50k-75k", "order": 8 } },
            { "case": { "$lt": ["$monthlyIncome", 100000] }, "then": { "label": "75k-100k", "order": 9 } }
          ],
          "default": { "label": "100k+", "order": 10 }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$incomeBucket",
      "count": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "incomeRange": "$_id.label",
      "order": "$_id.order",
      "count": 1
    }
  },
  {
    "$sort": { "order": 1 }
  }
]

6. Customer Tier Distribution - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": {"$ifNull": ["$customerTier", "Not Set"]}, "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"tier": "$_id", "count": 1, "_id": 0}}
]

7. Risk Category Distribution - Customers

[
  {"$match": {"riskCategory": {"$exists": true, "$ne": null}, "isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$riskCategory", "count": {"$sum": 1}}},
  {"$project": {"riskCategory": "$_id", "count": 1, "_id": 0}}
]

8. Age Distribution - Customers

[
  {
    "$match": {
      "dateOfBirth": { "$exists": true, "$ne": null },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$project": {
      "age": {
        "$floor": {
          "$divide": [
            { "$subtract": [ new Date(), "$dateOfBirth" ] },
            31557600000
          ]
        }
      }
    }
  },
  {
    "$match": {
      "age": { "$gte": 18, "$lte": 70 }
    }
  },
  {
    "$project": {
      "ageBucket": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$age", 22] }, "then": { "label": "18-22", "order": 1 } },
            { "case": { "$lt": ["$age", 25] }, "then": { "label": "22-25", "order": 2 } },
            { "case": { "$lt": ["$age", 28] }, "then": { "label": "25-28", "order": 3 } },
            { "case": { "$lt": ["$age", 30] }, "then": { "label": "28-30", "order": 4 } },
            { "case": { "$lt": ["$age", 35] }, "then": { "label": "30-35", "order": 5 } },
            { "case": { "$lt": ["$age", 40] }, "then": { "label": "35-40", "order": 6 } },
            { "case": { "$lt": ["$age", 45] }, "then": { "label": "40-45", "order": 7 } },
            { "case": { "$lt": ["$age", 50] }, "then": { "label": "45-50", "order": 8 } },
            { "case": { "$lt": ["$age", 55] }, "then": { "label": "50-55", "order": 9 } },
            { "case": { "$lt": ["$age", 60] }, "then": { "label": "55-60", "order": 10 } },
            { "case": { "$lt": ["$age", 70] }, "then": { "label": "60-70", "order": 11 } }
          ],
          "default": { "label": "70+", "order": 12 }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$ageBucket",
      "count": { "$sum": 1 }
    }
  },
  {
    "$project": {
      "_id": 0,
      "ageRange": "$_id.label",
      "order": "$_id.order",
      "count": 1
    }
  },
  {
    "$sort": { "order": 1 }
  }
]

9. City-wise Customer Distribution (Top 20) - Customers

[
  {
    $match: {
      isDeleted: { $ne: true }
    }
  },
  {
    $project: {
      city: {
        $cond: [
          {
            $or: [
              { $eq: ["$currentAddress.city", null] },
              { $eq: ["$currentAddress.city", ""] },
              { $not: ["$currentAddress.city"] }
            ]
          },
          "Unknown",
          "$currentAddress.city"
        ]
      }
    }
  },
  {
    $group: {
      _id: "$city",
      count: { $sum: 1 }
    }
  },
  {
    $sort: { count: -1 }
  },
  {
    $project: {
      city: "$_id",
      count: 1,
      _id: 0
    }
  }
]

10. State-wise Customer Distribution - Customers

[
  {"$match": {"currentAddress.state": {"$exists": true, "$ne": null}, "isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$state", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"state": "$_id", "count": 1, "_id": 0}}
]

11. Salary Mode Distribution - Customers

[
  {
    $match: {
      isDeleted: { $ne: true }
    }
  },
  {
    $project: {
      salaryMode: {
        $cond: [
          {
            $or: [
              { $eq: ["$salaryMode", null] },
              { $eq: ["$salaryMode", ""] },
              { $not: ["$salaryMode"] }
            ]
          },
          "Unknown",
          "$salaryMode"
        ]
      }
    }
  },
  {
    $group: {
      _id: "$salaryMode",
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      salaryMode: "$_id",
      count: 1,
      _id: 0
    }
  }
]

12. Contact Verification Status - Customers

[
  {
    $match: {
      isDeleted: { $ne: true }
    }
  },
  {
    $project: {
      status: {
        $cond: [
          {
            $or: [
              { $eq: ["$contactVerificationStatus", null] },
              { $eq: ["$contactVerificationStatus", ""] },
              { $not: ["$contactVerificationStatus"] }
            ]
          },
          "Unknown",
          "$contactVerificationStatus"
        ]
      }
    }
  },
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  },
  {
    $sort: { count: -1 }
  },
  {
    $project: {
      status: "$_id",
      count: 1,
      _id: 0
    }
  }
]