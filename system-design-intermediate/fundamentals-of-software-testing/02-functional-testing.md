---
slug: functional-testing
title: Functional Testing
readTime: 7 min
orderIndex: 2
premium: false
---


# **✅ Functional Testing: "Does Each Feature Work Correctly?"**


### **What is Functional Testing?**

Functional testing verifies that **each individual feature** works according to requirements, testing ONE thing at a time.

Feature: User Login

Test Cases:

1. Valid username \+ valid password → Success ✅

2. Valid username \+ wrong password → Error message ✅

3. Invalid username → Error message ✅

4. Empty fields → Validation error ✅

5. Password with special chars → Success ✅

6. SQL injection attempt → Blocked ✅

### **Real-World Analogy:**

**Like testing each appliance in a kitchen separately:**

Refrigerator:

  ✓ Does it cool to 40°F?

  ✓ Does freezer reach 0°F?

  ✓ Does ice maker work?

  ✓ Do shelves hold weight?

Oven:

  ✓ Does it heat to 350°F?

  ✓ Does timer work?

  ✓ Do burners light?

  ✓ Does auto-shutoff work?

Each appliance tested independently!

### **Functional Testing Example: Password Reset Feature**

Feature Requirements:

- User enters email

- System sends reset link

- Link expires in 24 hours

- User sets new password

- Old password no longer works

Test Cases:

TC-001: Valid Email Reset
Input: registered@email.com
Expected:

  ✅ Success message displayed

  ✅ Email sent with reset link

  ✅ Link works within 24 hours

TC-002: Unregistered Email

Input: notindb@email.com

Expected:

  ✅ Generic success message (security - don't reveal if email exists)

  ✅ No email sent

TC-003: Expired Link

Setup: Generate link, wait 25 hours

Expected:

  ✅ "Link expired" message

  ✅ Option to request new link

TC-004: Link Used Twice

Setup: Use valid link once
Expected:

  ✅ First use: Password changed ✅

  ✅ Second use: "Link already used" error

TC-005: Password Requirements

Input: Various passwords

Expected:

  ✅ "12345" → Rejected (too weak)

  ✅ "MyP@ssw0rd123" → Accepted

  ✅ Password must meet complexity rules

### **Functional vs. Unit Testing:**

Unit Testing (Developer perspective):

"Does this function return correct output?"
```js
 function validateEmail(email) {

    return email.includes('@')

 }

// Test:
 validateEmail("test@test.com") === true ✅
 ````

Functional Testing (User perspective):

"Can the user actually reset their password?"

1. User enters email

2. Clicks submit

3. Checks email inbox

4. Clicks link

5. Enters new password

6. Logs in with new password

All steps work from UI → API → Database → Email ✅

**Key characteristics:**

* 🎯 **Black box:** Test from user perspective, not code perspective

* 📋 **Requirements-based:** Each test maps to a requirement

* 🔍 **One feature at a time:** Isolated testing

* 📊 **Many test cases:** Cover positive, negative, edge cases
