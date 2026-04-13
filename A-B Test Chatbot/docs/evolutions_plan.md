# Project Evolution Plan

Based on the full codebase review, the current implementation of the Telegram A/B Test bot in `workflows/telegram-ab-test.json` has several gaps compared to the initial assignment requirements. This Evolution Plan outlines immediate fixes (Phase 1) and future enhancements (Phase 2).

## Phase 1: Immediate Fixes (Compliance with Assignment Requirements)

### 1. Fix Group Assignment Logic (Statsig Integration)
**Current State**: The workflow makes an HTTP POST request to Statsig API (`Statsig Initialize`), but the `Extract Group` node ignores the API response and instead uses hardcoded random assignment: `const group = Math.random() < 0.5 ? "control" : "test";`.
**Evolution Action**: 
*   Update the `Extract Group` node logic to parse the actual JSON response from Statsig (`$node["Statsig Initialize"].json.value`). If the gate returns `true`, assign "test", otherwise "control".

### 2. Implement the Complete 3-Step Guided Onboarding
**Current State**: The Test Group branch only sends a single specific message (`Test Step 1`), leaving the users without a complete 3-step flow.
**Evolution Action**: 
*   Expand the n8n logic (using conversational context or multiple webhooks/routers) to handle subsequent steps:
    *   **Step 1**: Goal Selection (Completed - implemented)
    *   **Step 2**: Dietary Restrictions Check (To be implemented)
    *   **Step 3**: Final Profile Setup / Confirmation (To be implemented)

### 3. Comprehensive Event Logging
**Current State**: Only the initial `assignment` event is being logged to the Supabase database (`Supabase Log Assignment1` node). 
**Evolution Action**: 
*   Add event logging nodes after **each** step in the onboarding flow, not just the initial assignment.
*   Log specific events such as `onboarding_step_completed`, `onboarding_finished`, and any ongoing `message_sent` events to track long-term retention. 

---

## Phase 2: Long-Term Enhancements

### 1. User State Management
*   Implement a robust cache (like Redis) or database-backed state machine within n8n to track which step a user is currently on natively, ensuring they receive the proper prompt even if they delay their responses.

### 2. Guardrail Mechanics & Quality of Life
*   Detect Telegram "bot blocked" errors (HTTP 403 Forbidden) effectively within the workflow to track our guardrail metric: `Bot Block Rate`.
*   Introduce an "Opt-out" or "Skip" command during the onboarding sequence.# Evaluation Plan: Telegram Chatbot A/B Test

This document outlines the rigorous evaluation framework for the Telegram Chatbot Onboarding A/B Test. The test compares a Control Group (generic welcome message) against a Test Group (guided 3-step onboarding flow).

## 1. Primary Metric
*The leading indicator strongly correlated with long-term retention.*

*   **Day-1 Activation Rate**: The percentage of new users who complete their first "core action" (e.g., sending a meaningful query, interacting with a menu, or completing the profile) within 24 hours of starting the bot.
    *   *Why?* Users who actively engage shortly after arriving are significantly more likely to become long-term retained users compared to those who only receive a welcome message and leave.

## 2. Guardrail Metrics
*Metrics to detect unintended negative effects or friction introduced by the new flow.*

*   **Bot Block/Unsubscribe Rate**: The percentage of users who block or delete the bot within the first 7 days.
    *   *Why?* A 3-step onboarding might feel spammy or too demanding, leading users to immediately block the bot.
*   **Drop-off Rate During Onboarding**: The percentage of users in the Test Group who start the 3-step flow but never finish it.
    *   *Why?* If this is too high, it indicates the onboarding steps are too complex or invasive.

## 3. Secondary Metrics
*Supporting metrics that provide deeper behavioral context.*

*   **Onboarding Completion Rate**: Percentage of assigned Test Group users who successfully complete all 3 onboarding steps.
*   **Day-7 Return Rate**: The percentage of users who send at least one message or interact with the bot exactly 7 days after their first interaction.
*   **Average Messages per User (First 7 Days)**: Volume of engagement to see if guided onboarding creates more conversive/active users.

## 4. Pre-Committed Decision Framework
*Outlining how results will be interpreted and acted upon after reaching statistical significance (e.g., 95% confidence level).*

| Scenario | Primary Metric (Activation) | Guardrail (Block Rate) | Decision & Next Steps |
| :--- | :--- | :--- | :--- |
| **Clear Winner** | Statistically significant **increase** | Neutral or **decrease** | **Roll out** the 3-step onboarding to 100% of new users. |
| **Negative Trade-off** | Statistically significant **increase** | Statistically significant **increase** | **Iterate**. The onboarding works but is too aggressive. Reduce friction (e.g., condense to 2 steps) and run a follow-up test. |
| **Flat / Losing** | Neutral or **decrease** | Neutral or **increase** | **Discard** the feature. Keep the Control group's generic welcome message or formulate a new onboarding hypothesis. |
| **Harmless but Ineffective** | Neutral | Neutral | **Discard**. If the feature adds complexity to the product/codebase without moving the needle, do not ship it. |

## 5. Experiment Sizing & Duration
*   **Duration**: The test will run until the required sample size is reached to detect a Minimum Detectable Effect (MDE) of X% at a 95% statistical significance level. A minimum of 1-2 full weeks is recommended to account for day-of-week seasonality.
