1. Overview - Apimetrics

[
  {
    "$group": {
      "_id": null,
      "totalAPICalls": { "$sum": 1 },
      "totalAPICost":  { "$sum": { "$ifNull": ["$cost", 0] } },
      "uniqueCustomers": { "$addToSet": "$customerId" }
    }
  },
  {
    "$addFields": {
      "uniqueCustomerCount": { "$size": "$uniqueCustomers" }
    }
  },
  {
    "$addFields": {
      "totalAPICost": {
        "$toLong": { "$add": ["$totalAPICost", 0.5] }
      },
      "avgCallsPerCustomer": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$totalAPICalls", { "$max": ["$uniqueCustomerCount", 1] }] }, 10] }, 0.5] } },
          10
        ]
      },
      "avgCostPerCustomer": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$totalAPICost", { "$max": ["$uniqueCustomerCount", 1] }] }, 100] }, 0.5] } },
          100
        ]
      }
    }
  },
  {
    "$project": {
      "totalAPICalls": 1,
      "totalAPICost": 1,
      "uniqueCustomers": "$uniqueCustomerCount",
      "avgCallsPerCustomer": 1,
      "avgCostPerCustomer": 1,
      "_id": 0
    }
  }
]

2. API Cost Per Successful Verification - Apimetrics

[
  {
    "$group": {
      "_id": {
        "provider": "$provider",
        "service": "$service"
      },
      "totalCalls":   { "$sum": 1 },
      "successCalls": { "$sum": { "$cond": [{ "$eq": ["$success", true] },  1, 0] } },
      "failedCalls":  { "$sum": { "$cond": [{ "$eq": ["$success", false] }, 1, 0] } },
      "totalCost":    { "$sum": { "$ifNull": ["$cost", 0] } }
    }
  },
  {
    "$addFields": {
      "successRate": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$successCalls", { "$max": ["$totalCalls", 1] }] }, 1000] }, 0.5] } },
          10
        ]
      },
      "totalCost": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": ["$totalCost", 100] }, 0.5] } },
          100
        ]
      },
      "costPerCall": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$totalCost", { "$max": ["$totalCalls", 1] }] }, 100] }, 0.5] } },
          100
        ]
      },
      "costPerSuccess": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$divide": ["$totalCost", { "$max": ["$successCalls", 1] }] }, 100] }, 0.5] } },
          100
        ]
      },
      "wastedSpend": {
        "$divide": [
          { "$toLong": { "$add": [{ "$multiply": [{ "$multiply": ["$totalCost", { "$divide": ["$failedCalls", { "$max": ["$totalCalls", 1] }] }] }, 100] }, 0.5] } },
          100
        ]
      }
    }
  },
  {
    "$sort": { "wastedSpend": -1 }
  },
  {
    "$project": {
      "provider":       "$_id.provider",
      "service":        "$_id.service",
      "totalCalls":     1,
      "successCalls":   1,
      "failedCalls":    1,
      "successRate":    1,
      "totalCost":      1,
      "costPerCall":    1,
      "costPerSuccess": 1,
      "wastedSpend":    1,
      "_id": 0
    }
  }
]