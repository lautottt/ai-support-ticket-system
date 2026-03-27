# AI Support Ticket Processing System

## Overview

This project is an AI-powered system designed to automate the processing of customer support tickets.

It analyzes incoming messages, detects multiple issues, classifies and prioritizes them, and generates both internal actions and a final response for the customer.

---

## Problem

Customer support teams often handle repetitive and multi-topic inquiries manually.

This leads to:
- Time-consuming workflows
- Inconsistent responses
- Delays in customer service

---

## Solution

This system automates the full workflow:

1. Extracts multiple issues from a single ticket
2. Identifies user intent and key entities
3. Classifies issues (billing, account, technical)
4. Assigns priority levels
5. Generates recommended actions
6. Creates structured summaries
7. Generates individual responses
8. Combines everything into a final response

---

## Key Features

- Multi-issue detection
- Structured JSON outputs
- Validation layer at each step
- Centralized error handling
- Modular architecture
- Final response composer

---

## Reliability

The system includes validation and control mechanisms at each stage to ensure data integrity and consistency.

- Schema validation for structured outputs
- Controlled error handling
- Fallback mechanisms for incomplete or invalid data

---

## Architecture

Ticket Input  
→ Issue Extraction  
→ Validation  
→ Classification  
→ Validation  
→ Summary Generation  
→ Validation  
→ Action Generation  
→ Validation  
→ Customer Response  
→ Final Response Composer  
→ Output

---

## Example

### Input

"I can't log into my account, I was charged twice this month, and the app crashes when I try to open it."

### Output

```json
{
  "status": "success",
  "data": [
    {
      "issue": "Unable to log into account",
      "category": "account",
      "priority": "high"
    },
    {
      "issue": "Double charge for subscription",
      "category": "billing",
      "priority": "high"
    },
    {
      "issue": "App crashes on startup",
      "category": "technical",
      "priority": "high"
    }
  ],
  "final_response": "Thank you for your message. You are unable to log into your account due to invalid credentials, and we will initiate a password reset process. Additionally, you have reported a double charge, and we will review and process the refund if confirmed. Lastly, the app is crashing, and we will investigate the issue further. Thank you for your patience."
},

*Note: Full structured output includes validation, actions, and detailed processing per issue.*
