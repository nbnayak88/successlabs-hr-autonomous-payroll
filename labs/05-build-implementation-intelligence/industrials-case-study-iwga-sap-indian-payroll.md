# Industrials Case Study - IWGA
## The Origin Story — SAP Indian Payroll for Beginners

**Learning ecosystem:** SuccessLabs Academy  
**Experience:** The Origin, powered by the Discovery Lab  
**Domain:** SAP HCM — Indian Payroll  
**Audience:** Beginners  
**Case context:** Indra Wings Global Airlines (IWGA)  
**Status:** Learning scenario; validate assumptions against the source case study before treating them as business facts.

> **Case-study note:** This learning artifact contextualizes wage-type concepts for the IWGA airline scenario. The Google Doc source should be reviewed and exact business facts reconciled before using this as a definitive representation of the original case. Employee names, amounts, company code, and calculations below are illustrative teaching assumptions.

---

## 1. The opening story

It is month-end at Indra Wings Global Airlines (IWGA). The Indian payroll team must calculate employee remuneration while handling different employee categories, compensation structures, attendance, absences, allowances, and changes during the month.

The payroll manager asks:

- How can each earning and deduction be represented consistently?
- How can the team determine which employees qualify for particular allowances?
- How should salary be calculated when an employee joins halfway through a payroll period?
- How can the right wage components contribute to the right calculation totals?
- How can payroll results be explained and validated?

**Learning mission:** Discover how SAP Indian Payroll configuration answers these questions, starting with wage types.

## 2. IWGA workforce scenario

For discovery purposes, consider four employee personas. These are illustrative archetypes, not confirmed IWGA compensation policies.

- **Pilot:** fixed remuneration and potentially applicable flying-related payments or allowances, subject to policy.
- **Cabin crew member:** salary and potentially eligible duty-related allowances, with relevant time or duty data affecting payments where configured.
- **Ground operations executive:** monthly salary and potentially applicable shift-related payments or overtime, depending on the role and policy.
- **Corporate HR employee:** a useful starting persona for recurring salary components, allowances, and partial-period calculations.

The employee groups may share the same payroll engine without sharing identical wage types, eligibility, valuation, or processing rules. Do not assume that every member of a job category qualifies for a particular allowance.

## 3. Chapter 1 — Wage Types: What is IWGA paying for?

- IWGA needs consistent representations for Basic Pay, HRA, Special Allowance, overtime, bonuses, and applicable deductions.
- Each component has a distinct business meaning and may need its own wage type.
- A pilot's flying-related payment and a ground employee's shift allowance may represent different business rules and require separate treatment.
- Distinguish wage types entered manually, wage types originating from employee master data, and wage types generated during payroll processing.
- Wage type texts and groups help administrators organise, recognise, and maintain wage types.

**Syllabus mapping:**  
1.1 Wage Type Overview  
1.2 Wage Type Creation, Deletion, Completeness Check  
1.3 Wage Type Texts  
1.4 Wage Type Groups  
1.5 Wage Types in Wage Type Groups

**Discovery activity:** Ask each learner to propose five wage types IWGA might need. Have the group explain the business purpose of each before reviewing configuration.

**Key question:** If Basic Pay and HRA are both earnings, why should SAP treat them as separate wage types?

## 4. Chapter 2 — Wage Type Properties: How should SAP treat the payment?

- Naming a wage type is not enough; SAP needs configuration that determines how the wage type behaves.
- Basic Pay, HRA, and overtime may require different processing and evaluation behaviour.
- Processing classes and evaluation classes support configured handling during payroll processing and evaluation.
- Incorrect configuration can omit an amount from a required calculation or include it where it does not belong.
- A wage type's name alone does not establish its complete calculation behaviour.

**Syllabus mapping:**  
2.1 Wage Type Characteristics  
2.2 Processing Cumulation and Evaluation Classes

**Discovery activity:** Compare Basic Pay with an illustrative overtime wage type. Ask what additional rules SAP would need before it could calculate the overtime amount.

**Key question:** What could go wrong if a wage type is configured with an incorrect processing class?

## 5. Chapter 3 — Wage Type Permissibility: Who can receive or enter it?

- IWGA needs appropriate controls over wage type entry.
- A recurring salary component may belong in recurring payment data, while a one-time bonus may use a different entry process.
- Historical entries may be restricted or allowed according to configured rules.
- Employee category, eligibility, and wage type configuration must be considered together.
- A wage type being technically available does not prove that an employee is eligible to receive it.

**Syllabus mapping:**  
3.1 Permissibility of Wage Types in Infotypes  
3.2 Permissibility of Wage Type Entry in the Past  
3.3 Permissibility of Wage Types for Employees

**Discovery activity:** Propose a flying-related allowance for a corporate HR employee. Ask the group what it would validate before allowing the entry.

**Key question:** Should payroll administrators be able to enter every wage type for every employee? Why or why not?

## 6. Chapter 4 — Wage Type Valuation: How much should IWGA pay?

- Valuation determines how SAP derives the amount of a wage type.
- Some wage types use fixed amounts; others may depend on rates, percentages, time, or configured rules.
- Indirect valuation can derive amounts through configured employee groupings and valuation methods.
- Allowance grouping becomes relevant when different employee populations follow different allowance rules.
- The INVAL variants in the syllabus must be learned through their precise Indian Payroll configuration and examples rather than assumed to be interchangeable formulas.
- Valuation bases identify values used as inputs to other calculations.
- Average-based valuation may use historical remuneration data depending on the configured procedure.

**Syllabus mapping:**  
4.1 Indirect Valuation of Wage Types for India  
4.2 Allowance Grouping  
4.3 Allowance Grouping Definition  
4.4 Wage Type Model for Allowance Grouping  
4.5 INVAL A Amount  
4.6 INVAL B, C and D Percentage  
4.7 INVAL B and C Amount  
4.8 INVAL D Amount  
4.9 INVAL Amount Multiplier  
4.10 Valuation of Time Wage Types  
4.11 Valuation Bases  
4.12 Wage Type-Dependent Constant Valuation Bases  
4.13 Pay Scale-Dependent Constant Valuation Bases  
4.14 Employee-Related Valuation Bases  
4.15 Wage Type Valuation Based on Averages  
4.16 Different Payment

**Discovery activity:** Compare two employee profiles with different configured allowance eligibility. Ask how the same business event could produce different payment amounts.

**Key question:** How does SAP know what an allowance should be worth?

## 7. Chapter 5 — Wage Type Cumulation: How does IWGA calculate totals?

- IWGA needs meaningful totals for payroll calculations, reporting, and subsequent processing.
- Selected wage types can contribute to specific cumulation totals according to configuration.
- Gross remuneration is conceptually different from a statutory contribution base or a tax-related calculation base.
- One wage type may contribute to one total but not another, depending on applicable rules.
- The functions SETCU and ADDCU should be understood in context as payroll functions associated with cumulation processing.

**Syllabus mapping:**  
5.1 Overview  
5.2 Cumulation Type  
5.3 Valid Cumulation Types  
5.4 Cumulation Periods  
5.5 Calendar for Cumulation  
5.6 Cumulation of Wage Types  
5.7 Function SETCU  
5.8 Function ADDCU

**Discovery activity:** Ask whether Basic Pay, HRA, overtime, and a reimbursement should contribute to every payroll total. Have learners state their assumptions and identify the rules that need validation.

**Key question:** Why would SAP need both an individual allowance amount and a total of selected allowances?

## 8. Chapter 6 — Wage Type Factoring: What if the employee works only part of the month?

- An IWGA employee joins mid-month.
- Another employee has an eligible unpaid absence.
- A third employee has a salary change during the payroll period.
- These events may require partial-period calculations, wage type splits, or adjustments to applicable payments.
- The result depends on employee dates, payroll data, configured rules, and wage type behaviour.
- Functions such as PARTT, RAB, PAB, PRINT, and GEN/8 should be introduced as technical mechanisms in payroll processing, not as isolated transaction codes to memorise.

**Syllabus mapping:**  
6.1 Partial Period / Wage Type Split  
6.2 Salary Reduction for Absences  
6.3 Partial Period Factors  
6.4 Partial Period Parameters  
6.5 Function PARTT  
6.6 Function RAB  
6.7 Function PAB  
6.8 Function PRINT  
6.9 Function GEN/8  
6.10 Partial Period Factors Calculation  
6.11 Wage Type Valuation Using Partial Period Factors

**Discovery activity:** Ask learners to calculate a simple half-period salary example by hand, then identify which employee and configuration data SAP needs to calculate it correctly.

**Key question:** Does joining halfway through a month always result in a factor of exactly 0.5? What should be checked first?

## 9. Worked teaching example — Mid-month joiner

**Employee:** Ananya Rao  
**Role:** Ground Operations Executive  
**All values below are hypothetical learning inputs, not verified IWGA payroll data.**

| Wage type example | Illustrative full-month amount |
|---|---:|
| Basic Pay | ₹30,000 |
| HRA | ₹15,000 |
| Special Allowance | ₹10,000 |
| **Illustrative gross** | **₹55,000** |

For this arithmetic exercise only, assume that all three components are eligible for the same illustrative partial-period factor of 0.5.

| Component | Calculation | Illustrative factored amount |
|---|---:|---:|
| Basic Pay | ₹30,000 × 0.5 | ₹15,000 |
| HRA | ₹15,000 × 0.5 | ₹7,500 |
| Special Allowance | ₹10,000 × 0.5 | ₹5,000 |
| **Factored gross** | ₹55,000 × 0.5 | **₹27,500** |

This is a simplified arithmetic example, not a conclusion about actual IWGA entitlement or a universal SAP calculation rule. Actual factoring depends on the configured rules, relevant dates, wage type treatment, and payroll data.

Ask learners to explain:

1. Why are three separate wage types represented?
2. How does SAP know which components are eligible for a factor?
3. What changes if only one of the three components is eligible?
4. What changes if an unpaid absence occurs later in the period?
5. What payroll result checks should take place before accepting the calculation?

## 10. Discovery board for Notion

Capture group learning in a shared database instead of presenting every answer at the start.

| Field | Purpose |
|---|---|
| Business scenario | IWGA payroll problem being investigated |
| Employee persona | Pilot, cabin crew, ground operations, or corporate |
| Wage type or concept | Syllabus topic being explored |
| Learner hypothesis | What the learner thinks SAP should do |
| Evidence or configuration to check | What would validate the hypothesis |
| Outcome | Confirmed, needs investigation, or corrected |
| Open question | What the group should explore next |

Invite each participant to contribute one scenario, one hypothesis, and one question. Preserve corrections and unresolved questions for subsequent sessions.

## 11. Suggested facilitation sequence

1. **Tell the story:** Introduce the month-end payroll challenge at IWGA.
2. **Classify the components:** Have learners identify earnings, deductions, and calculation totals.
3. **Introduce a business change:** A mid-month joiner, unpaid absence, or compensation change.
4. **Connect the problem to the syllabus:** Identify whether the investigation concerns wage type properties, permissibility, valuation, cumulation, or factoring.
5. **Record the discovery:** Capture contributions and open questions in Notion.
6. **Review the result:** Ask learners to explain the business rule and the configuration that would need validation.

## 12. Learning outcomes

By the end of The Origin, a beginner should be able to:

- Explain a wage type using an IWGA business example.
- Distinguish wage type characteristics from processing classes.
- Explain why wage type permissibility matters.
- Describe the difference between valuation, cumulation, and factoring.
- Recognise the role of India-specific INVAL configuration.
- Trace a simple remuneration scenario from source data through payroll calculations.
- Formulate meaningful validation questions before changing SAP configuration.

## 13. Accuracy and case-study controls

Before using this learning artifact as a definitive IWGA case-study deliverable:

- Replace illustrative employee profiles and salary amounts with approved case-study facts where available.
- Validate actual employee categories, allowance policies, eligibility conditions, and payroll-period assumptions.
- Verify the exact technical behaviour of Indian Payroll functions and INVAL variants against the target SAP release and configuration.
- Do not interpret hypothetical amounts as statutory calculations, tax advice, or production payroll rules.

## Closing thought

At Indra Wings, payroll is not simply the act of transferring money to employees. It is the disciplined translation of employment data and compensation policy into a traceable, explainable payroll result.

**The Origin principle:** Start with the employee, the business rule, and the question payroll must answer. Then discover which SAP configuration makes that answer possible.
