1. Total / Calculator Usage Rate / Eligibility Usage Rate / FAQ View Rate - User Journeys

[
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "usedCalculator": {"$sum": {"$cond": [{"$eq": ["$usedEmiCalculator", true]}, 1, 0]}},
    "usedEligibility": {"$sum": {"$cond": [{"$eq": ["$usedEligibilityChecker", true]}, 1, 0]}},
    "visitedFaqs": {"$sum": {"$cond": [{"$eq": ["$visitedFaqs", true]}, 1, 0]}},
    "viewedMultipleProducts": {"$sum": {"$cond": [{"$eq": ["$viewedMultipleProducts", true]}, 1, 0]}}
  }},
  {"$project": {
    "_id": 0,
    "total": 1,
    "calculatorUsageRate": {"$multiply": [{"$divide": ["$usedCalculator", {"$max": ["$total", 1]}]}, 100]},
    "eligibilityUsageRate": {"$multiply": [{"$divide": ["$usedEligibility", {"$max": ["$total", 1]}]}, 100]},
    "faqViewRate": {"$multiply": [{"$divide": ["$visitedFaqs", {"$max": ["$total", 1]}]}, 100]}
  }}
]

2. Average Time Spent on Application - User Journeys

[
  {"$match": {"totalTimeSpent": {"$gt": 0}}},
  {"$group": {
    "_id": "$journeyStatus",
    "avgTimeMinutes": {"$avg": {"$divide": ["$totalTimeSpent", 60]}},
    "count": {"$sum": 1}
  }},
  {"$project": {"journeyStatus": "$_id", "avgTimeMinutes": 1, "count": 1, "_id": 0}}
]

3. Device Type Distribution - User Journeys

[
  {"$match": {"sessions": {"$exists": true, "$ne": []}}},
  {"$project": {"isMobile": {"$arrayElemAt": ["$sessions.deviceInfo.isMobile", 0]}}},
  {"$group": {
    "_id": {"$cond": ["$isMobile", "Mobile", "Desktop"]},
    "count": {"$sum": 1}
  }},
  {"$project": {"device": "$_id", "count": 1, "_id": 0}}
]

4. Rage Click Hotspots - User Journeys

[
  {"$match": {"rageClicks": {"$exists": true, "$ne": []}}},
  {"$unwind": "$rageClicks"},
  {"$group": {
    "_id": "$rageClicks.element",
    "totalRageClicks": {"$sum": "$rageClicks.count"},
    "occurrences": {"$sum": 1}
  }},
  {"$sort": {"totalRageClicks": -1}},
  {"$limit": 20},
  {"$project": {"element": "$_id", "totalRageClicks": 1, "occurrences": 1, "_id": 0}}
]

5. Bot/Suspicious Traffic - User Journeys

[
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "bots": {"$sum": {"$cond": [{"$eq": ["$isBot", true]}, 1, 0]}},
    "suspicious": {"$sum": {"$cond": [{"$eq": ["$isSuspicious", true]}, 1, 0]}}
  }},
  {"$project": {
    "_id": 0,
    "total": 1,
    "bots": 1,
    "botRate": {"$multiply": [{"$divide": ["$bots", {"$max": ["$total", 1]}]}, 100]},
    "suspicious": 1,
    "suspiciousRate": {"$multiply": [{"$divide": ["$suspicious", {"$max": ["$total", 1]}]}, 100]}
  }}
]

6. Trust Signal Interactions - User Journeys

[
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "clickedRbiLogo": {"$sum": {"$cond": [{"$eq": ["$clickedRbiLogo", true]}, 1, 0]}},
    "clickedSecurityBadge": {"$sum": {"$cond": [{"$eq": ["$clickedSecurityBadge", true]}, 1, 0]}},
    "viewedPrivacyPolicy": {"$sum": {"$cond": [{"$eq": ["$viewedPrivacyPolicy", true]}, 1, 0]}},
    "viewedTerms": {"$sum": {"$cond": [{"$eq": ["$viewedTerms", true]}, 1, 0]}},
    "expandedFaq": {"$sum": {"$cond": [{"$eq": ["$expandedFaq", true]}, 1, 0]}}
  }},
  {"$project": {"_id": 0}}
]

7. Form Paste Detection (Fraud Signal) - User Journeys 

[
  {"$match": {"formBehavior": {"$exists": true}}},
  {"$group": {
    "_id": null,
    "total": {"$sum": 1},
    "panPaste": {"$sum": {"$cond": [{"$eq": ["$formBehavior.pasteDetected.panNumber", true]}, 1, 0]}},
    "aadhaarPaste": {"$sum": {"$cond": [{"$eq": ["$formBehavior.pasteDetected.aadhaarNumber", true]}, 1, 0]}},
    "accountPaste": {"$sum": {"$cond": [{"$eq": ["$formBehavior.pasteDetected.accountNumber", true]}, 1, 0]}},
    "ifscPaste": {"$sum": {"$cond": [{"$eq": ["$formBehavior.pasteDetected.ifscCode", true]}, 1, 0]}},
    "autoFillDetected": {"$sum": {"$cond": [{"$eq": ["$formBehavior.autoFillDetected", true]}, 1, 0]}}
  }},
  {"$project": {"_id": 0}}
]

8. Tab Switch Frequency Distribution - User Journeys

[
  {
    "$match": {
      "formBehavior.tabSwitchCount": { "$gt": 0 }
    }
  },
  {
    "$addFields": {
      "rangeId": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 3] },  "then": 1 },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 5] },  "then": 2 },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 10] }, "then": 3 },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 20] }, "then": 4 },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 50] }, "then": 5 }
          ],
          "default": 6
        }
      },
      "range": {
        "$switch": {
          "branches": [
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 3] },  "then": "1-2" },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 5] },  "then": "3-4" },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 10] }, "then": "5-9" },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 20] }, "then": "10-19" },
            { "case": { "$lt": ["$formBehavior.tabSwitchCount", 50] }, "then": "20-49" }
          ],
          "default": "50+"
        }
      }
    }
  },
  {
    "$group": {
      "_id": { "rangeId": "$rangeId", "range": "$range" },
      "count": { "$sum": 1 }
    }
  },
  { "$sort": { "_id.rangeId": 1 } },
  {
    "$project": {
      "_id": 0,
      "range": "$_id.range",
      "count": 1
    }
  }
]