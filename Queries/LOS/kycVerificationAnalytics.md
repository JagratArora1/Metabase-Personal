1. Daily KYC Completions - Customers

[
  {"$match": {"kycStatus": "SUCCESS", "kycCompletedAt": {"$exists": true}, "isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"$dateToString": {"format": "%Y-%m-%d", "date": "$kycCompletedAt"}},
    "count": {"$sum": 1}
  }},
  {"$sort": {"_id": 1}},
  {"$project": {"date": "$_id", "kycCompletions": "$count", "_id": 0}}
]

2. Document Upload Analytics - Documents

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$documentType", "count": {"$sum": 1}}},
  {"$sort": {"count": -1}},
  {"$project": {"documentType": "$_id", "count": 1, "_id": 0}}
]

3. Document Verification Status - Documents

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {
    "_id": {"type": "$documentType", "status": "$status"},
    "count": {"$sum": 1}
  }},
  {"$sort": {"_id.type": 1}},
  {"$project": {"documentType": "$_id.type", "status": "$_id.status", "count": 1, "_id": 0}}
]

4. KYC Completion Funnel - Customers 

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,

      "Registered": { "$sum": 1 },

      "PAN Verified": {
        "$sum": {
          "$cond": [
            { "$eq": ["$isPanVerify", true] },
            1,
            0
          ]
        }
      },

      "Aadhaar Verified": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isPanVerify", true] },
                { "$eq": ["$isAadhaarVerify", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Bank Verified": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isPanVerify", true] },
                { "$eq": ["$isAadhaarVerify", true] },
                { "$eq": ["$isBankDetailsFilled", true] }
              ]
            },
            1,
            0
          ]
        }
      },

      "KYC Complete": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isPanVerify", true] },
                { "$eq": ["$isAadhaarVerify", true] },
                { "$eq": ["$isBankDetailsFilled", true] },
                { "$eq": ["$kycStatus", "SUCCESS"] }
              ]
            },
            1,
            0
          ]
        }
      },

      "Submitted": {
        "$sum": {
          "$cond": [
            {
              "$and": [
                { "$eq": ["$isPanVerify", true] },
                { "$eq": ["$isAadhaarVerify", true] },
                { "$eq": ["$isBankDetailsFilled", true] },
                { "$eq": ["$kycStatus", "SUCCESS"] },
                { "$eq": ["$isSubmit", true] }
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
        { "stage": "Registered", "count": "$Registered" },
        { "stage": "PAN Verified", "count": "$PAN Verified" },
        { "stage": "Aadhaar Verified", "count": "$Aadhaar Verified" },
        { "stage": "Bank Verified", "count": "$Bank Verified" },
        { "stage": "KYC Complete", "count": "$KYC Complete" },
        { "stage": "Submitted", "count": "$Submitted" }
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

5. KYC Status Breakdown - Customers

[
  {"$match": {"isDeleted": {"$ne": true}}},
  {"$group": {"_id": "$kycStatus", "count": {"$sum": 1}}},
  {"$project": {"kycStatus": "$_id", "count": 1, "_id": 0}}
]

6. Manual vs API Verification - Customers

[
  {
    "$match": {
      "isDeleted": { "$ne": true }
    }
  },
  {
    "$group": {
      "_id": null,

      "apiPan": {
        "$sum": { "$cond": [{ "$eq": ["$isPanVerify", true] }, 1, 0] }
      },

      "manualPan": {
        "$sum": { "$cond": [{ "$eq": ["$manualPanVerify.isVerified", true] }, 1, 0] }
      },

      "apiAadhaar": {
        "$sum": { "$cond": [{ "$eq": ["$isAadhaarVerify", true] }, 1, 0] }
      },

      "manualAadhaar": {
        "$sum": { "$cond": [{ "$eq": ["$manualAadhaarVerify.isVerified", true] }, 1, 0] }
      },

      "manualFace": {
        "$sum": { "$cond": [{ "$eq": ["$manualFaceVerify.isVerified", true] }, 1, 0] }
      },

      "manualBank": {
        "$sum": { "$cond": [{ "$eq": ["$manualBankVerify.isVerified", true] }, 1, 0] }
      }
    }
  },
  {
    "$project": {
      "rows": [
        { "VerificationType": "PAN", "Method": "API", "Count": "$apiPan" },
        { "VerificationType": "PAN", "Method": "Manual", "Count": "$manualPan" },

        { "VerificationType": "Aadhaar", "Method": "API", "Count": "$apiAadhaar" },
        { "VerificationType": "Aadhaar", "Method": "Manual", "Count": "$manualAadhaar" },

        { "VerificationType": "Face", "Method": "Manual", "Count": "$manualFace" },

        { "VerificationType": "Bank", "Method": "Manual", "Count": "$manualBank" }
      ]
    }
  },
  { "$unwind": "$rows" },
  {
    "$project": {
      "_id": 0,
      "VerificationType": "$rows.VerificationType",
      "Method": "$rows.Method",
      "Count": "$rows.Count"
    }
  }
]