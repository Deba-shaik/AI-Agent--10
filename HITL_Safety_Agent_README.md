# HITL Safety Agent - README

## Project overview
This n8n workflow demonstrates a human-in-the-loop (HITL) approval gate for AI-drafted customer discount offers. The AI drafts an offer; Slack notifies a reviewer; an n8n Wait node pauses for an authenticated POST callback; an IF node routes the manager decision. No discount transaction is executed by the demonstrated workflow.

## Workflow architecture
When chat message received -> AI Agent (OpenAI Chat Model) -> Slack: Send a message -> Wait (On Webhook Call) -> IF -> Manager_Approved / Manager_Rejected.

## Implemented configuration
- AI Agent: drafts customer discount text; its system instructions prohibit autonomous approval, sending or execution.
- Slack: posts a draft and manager-review request to the configured channel.
- Wait: resumes on POST webhook call, with Header Auth and HTTP 200 immediate response.
- IF: compares `{{ $json.body.decision }}` to the exact string `Approved` (case-sensitive).
- Manager_Approved: emits `approval_status=Approved`, an approval message, and `discount_executed=false`.
- Manager_Rejected: emits `approval_status=Rejected`, a rejection message, and `discount_executed=false`.

## Reproduction / testing
1. Configure valid AI model and Slack credentials; restrict access to reviewer channels.
2. Start a new chat-triggered execution and provide a discount-drafting request.
3. Confirm Slack message and the Wait node's waiting state; copy the fresh execution-specific resume URL.
4. POST JSON `{"decision":"Approved","manager_id":"MGR001","comments":"Discount approved"}` with the matching Header Auth credential; expect IF true branch and Approved output.
5. Start a **new** execution with a **new** resume URL; POST `{"decision":"Rejected","manager_id":"MGR001","comments":"Discount rejected"}`; expect IF false branch and Rejected output.
6. Do not reuse an already completed resume URL: n8n returns HTTP 409 when that execution is finished.

## Observed evidence
- Main workflow: Chat Trigger, AI Agent, Slack, Wait and IF executed; one item reached the Rejected branch in the latest screenshot.
- Earlier Approved output: `approval_status: Approved`, `discount_executed: false`.
- Rejected output: `approval_status: Rejected`, `discount_executed: false`.
- These are functional tests, not proof that a real manager authenticated and clicked an approval UI.

## Safety, limitations and remaining assessment gaps
- The approval/rejection branches are Set/Edit Fields nodes, not real customer transaction actions. No discount was sent or executed.
- Current implementation uses Postman JSON approval callbacks, rather than the native n8n approval form described in the Topic 10 guide.
- The screenshots do not verify a 15,000 ms AI Agent timeout, a 400-token ceiling, or a configured/activated Global_Error_Handler workflow. Capture these separately if required for full Topic 10 coverage.
- Keep a decision audit trail (reviewer identity, decision, comments, timestamp, request ID), verify identity server-side, enforce expiry and replay protection, and restrict allowed reviewers before production use.
- Rotate any tokens exposed in screenshots/chat; signed resume URLs are sensitive. Do not publish raw secrets or full authenticated request logs.

## Submission files
- `HITL_Safety_Agent_README.md`: project setup, tests, limitations.
- `HITL_Safety_Agent_Evidence.pdf`: screenshot evidence with sensitive regions removed.
