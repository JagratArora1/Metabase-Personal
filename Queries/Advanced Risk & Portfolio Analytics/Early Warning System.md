1. Balance Monitor - Salary Detection & Early Warning - Balance Monitors

[
  {
    "$match": {
      "snapshots.0": { "$exists": true }
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
    "$project": {
      "loanNumber": "$loan.loanNumber",
      "loanStatus": "$loan.status",
      "dpd": "$loan.dpd",
      "consentStatus": "$consent.consentStatus",
      "snapshotCount": { "$size": "$snapshots" },
      "latestBalance": { "$arrayElemAt": ["$snapshots.currentBalance", -1] },
      "previousBalance": { "$arrayElemAt": ["$snapshots.currentBalance", -2] },
      "salaryDetected": {
        "$anyElementTrue": {
          "$map": {
            "input": "$snapshots",
            "as": "s",
            "in": "$$s.salaryDetected"
          }
        }
      },
      "alertCount": { "$size": { "$ifNull": ["$alerts", []] } },
      "totalOutstanding": "$loan.totalOutstanding",
      "monitoringEnabled": "$monitoring.enabled"
    }
  },
  {
    "$addFields": {
      "balanceDropPct": {
        "$cond": {
          "if": {
            "$and": [
              { "$gt": [{ "$ifNull": ["$previousBalance", 0] }, 0] },
              { "$gt": [{ "$ifNull": ["$latestBalance", 0] }, 0] }
            ]
          },
          "then": {
            "$divide": [
              {
                "$toLong": {
                  "$add": [
                    {
                      "$multiply": [
                        { "$divide": [
                          { "$subtract": ["$previousBalance", "$latestBalance"] },
                          "$previousBalance"
                        ]},
                        1000
                      ]
                    },
                    0.5
                  ]
                }
              },
              10
            ]
          },
          "else": 0
        }
      },
      "balanceToOutstandingRatio": {
        "$cond": {
          "if": { "$gt": [{ "$ifNull": ["$totalOutstanding", 0] }, 0] },
          "then": {
            "$divide": [
              {
                "$toLong": {
                  "$add": [
                    { "$multiply": [
                      { "$divide": [
                        { "$ifNull": ["$latestBalance", 0] },
                        "$totalOutstanding"
                      ]},
                      100
                    ]},
                    0.5
                  ]
                }
              },
              100
            ]
          },
          "else": 0
        }
      }
    }
  },
  {
    "$sort": { "balanceToOutstandingRatio": 1 }
  },
  {
    "$limit": 50
  }
]

2. Loans Due in Next 7 Days (Upcoming Maturities) - Loans

[
  {
    "$match": {
      "status": "ACTIVE",
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "customer"
    }
  },
  {
    "$unwind": { "path": "$customer", "preserveNullAndEmptyArrays": true }
  },
  {
    "$addFields": {
      "startDate": {
        "$dateFromString": {
          "dateString": {"$concat": [{{day_start}}, "T18:30:00.000+00:00"]}
        }
      },
      "endDate": {
        "$dateFromString": {
          "dateString": {"$concat": [{{day_end}}, "T18:29:59.000+00:00"]}
        }
      }
    }
  },
  {
    "$match": {
      "$expr": {
        "$and": [
          { "$gte": ["$dueDate", "$startDate"] },
          { "$lte": ["$dueDate", "$endDate"] }
        ]
      }
    }
  },
  {
    "$project": {
      "loanNumber": 1,
      "customerName": { "$ifNull": ["$customer.fullName", "Unknown"] },
      "mobile": "$customer.mobile",
      "principalAmount": 1,
      "totalOutstanding": 1,
      "dueDate": 1,
      "disbursementDate": 1,
      "tenure": 1,
      "remindersSent": { "$ifNull": ["$reminderTracking.totalRemindersSent", 0] },
      "lastReminderType": { "$ifNull": ["$reminderTracking.lastReminderType", "None"] },
      "_id": 0
    }
  },
  {
    "$sort": { "dueDate": 1 }
  }
]

3. E-Mandate Coverage vs Collection Success - Loans

[
  {
    "$match": {
      "isDeleted": { "$ne": true },
      "status": { "$in": ["ACTIVE", "OVERDUE", "CLOSED"] }
    }
  },
  {
    "$lookup": {
      "from": "mandates",
      "localField": "_id",
      "foreignField": "loanId",
      "as": "mandate"
    }
  },
  {
    "$addFields": {
      "mandateStatus": {
        "$cond": {
          "if": { "$eq": [{ "$size": "$mandate" }, 0] },
          "then": "No Mandate",
          "else": {
            "$cond": {
              "if": {
                "$anyElementTrue": {
                  "$map": {
                    "input": "$mandate",
                    "as": "m",
                    "in": { "$eq": ["$$m.status", "active"] }
                  }
                }
              },
              "then": "Active Mandate",
              "else": "Inactive Mandate"
            }
          }
        }
      }
    }
  },
  {
    "$group": {
      "_id": "$mandateStatus",
      "count": { "$sum": 1 },
      "avgDPD": { "$avg": { "$ifNull": ["$dpd", 0] } },
      "overdueCount": { "$sum": { "$cond": [{ "$eq": ["$status", "OVERDUE"] }, 1, 0] } },
      "closedCount": { "$sum": { "$cond": [{ "$eq": ["$status", "CLOSED"] }, 1, 0] } },
      "totalOutstanding": { "$sum": "$totalOutstanding" }
    }
  },
  {
    "$addFields": {
      "avgDPD": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$avgDPD", 10] }, 0.5] } },
          10
        ]
      },
      "overdueRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$overdueCount", { "$max": ["$count", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "closureRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$closedCount", { "$max": ["$count", 1] }] }, 1000] }, 0.5] } },
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
      "mandateStatus": "$_id",
      "count": 1,
      "avgDPD": 1,
      "overdueRate": 1,
      "closureRate": 1,
      "overdueCount": 1,
      "closedCount": 1,
      "totalOutstanding": 1,
      "_id": 0
    }
  }
]

4. Concentration Risk - Top 10 Customers by Outstanding - Loans

[
  {
    "$match": {
      "status": { "$in": ["ACTIVE", "OVERDUE"] },
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": "$customerId",
      "totalOutstanding": { "$sum": "$totalOutstanding" },
      "loanCount": { "$sum": 1 },
      "maxDPD": { "$max": { "$ifNull": ["$dpd", 0] } },
      "totalPrincipal": { "$sum": "$principalAmount" }
    }
  },
  {
    "$lookup": {
      "from": "customers",
      "localField": "_id",
      "foreignField": "_id",
      "as": "customer"
    }
  },
  {
    "$unwind": { "path": "$customer", "preserveNullAndEmptyArrays": true }
  },
  {
    "$project": {
      "customerId": { "$toString": "$_id" },
      "customerName": { "$ifNull": ["$customer.fullName", "Unknown"] },
      "mobile": { "$ifNull": ["$customer.mobile", "N/A"] },
      "loanCount": 1,
      "totalPrincipal": 1,
      "totalOutstanding": 1,
      "maxDPD": 1,
      "_id": 0
    }
  },
  {
    "$sort": { "totalOutstanding": -1 }
  },
  {
    "$limit": 10
  }
]