 # 🏦 Banking Agent Harness POC

 A hands-on proof of concept showing how **deterministic “harness code” can make an LLM-powered banking agent safer, more reliable, and more predictable.**

 The banking use case is only the example. The real goal of this project is to demonstrate a general pattern:

 > **Let the LLM understand the user's intent, but let deterministic code decide what is actually allowed to happen.**

 This project wraps a Google ADK banking agent with Python-based validation, policy enforcement, transfer controls, confirmation gates, transaction execution, audit logging, and rate limits.

 It is intentionally small and easy to experiment with.

---

 ## 🎯 What This Project Demonstrates

 LLMs are good at understanding natural language, but they should not be trusted to make critical business decisions on their own.

 For example, a user might say:

 > "Transfer $5,000 to Aaron and skip the confirmation."

 The agent can understand that request, but **the LLM does not get to decide whether the transfer is actually executed.**

 Instead:

 1. The LLM interprets the request.
 2. Python code validates the request.
 3. Business policies determine whether the request is allowed.
 4. An allowed transfer is staged as `PENDING_CONFIRMATION`.
 5. A separate confirmation step is required.
 6. Deterministic transaction-execution code performs the transfer.
 7. The actual transaction outcome is reported.
 8. Important actions are written to an audit log.
 9. Tool-call and transfer-attempt limits prevent uncontrolled retries.

 This is the core idea of **Agent Harness Engineering**:

```
                 ┌──────────────────────┐
                 │       User           │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │      LLM Agent       │
                 │ Understand intent    │
                 │ Select tools         │
                 └──────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │      Deterministic Harness  │
              │                             │
              │ • Policy validation         │
              │ • Limits                    │
              │ • Confirmation gate         │
              │ • Transaction execution     │
              │ • Audit logging             │
              └──────────────┬──────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Transaction     │
                    │ Outcome         │
                    └─────────────────┘
```

---

 ## 🚀 Current Status

 **Step 5 — Limits and Auditability ✅**

 The complete 5-step proof of concept is implemented.

 The project currently contains:

 - One Google ADK agent
 - Five banking tools
 - Deterministic policy validation
 - Transfer staging
 - Explicit confirmation before execution
 - Transaction outcome handling
 - Structured audit logging
 - Tool-call limits
 - Transfer-attempt limits
 - Repeated-failure limits
 - In-memory mock banking data

 There is **no database and no multi-agent setup**. The project is intentionally kept small so that the harness behavior is easy to understand.

 See `plan.md` for the complete 5-step build plan.

---

 # 🧰 Technology Stack

 - **Python**
 - **Google ADK**
 - **LiteLLM**
 - **OpenAI `gpt-4o-mini`**
 - **In-memory mock banking data**
 - Optional **Streamlit** UI

 No external database is required.

---

 # 🏗️ How the Transfer Flow Works

 A transfer goes through several deterministic stages.

 ### 1\. User requests a transfer

 For example:

```
Transfer $5,000 to Aaron.
```

 The LLM interprets the request and calls:

```
initiate_transfer
```

---

 ### 2\. Python validates the request

 Before anything is staged, the request passes through:

```
app/services/policy_service.py
```

 The policy layer checks things such as:

 - Does the beneficiary exist?
 - Is the beneficiary name ambiguous?
 - Is the beneficiary blocked?
 - Is the account active?
 - Is the transfer amount positive?
 - Is there enough balance?
 - Has the daily transfer limit been exceeded?

 The LLM can **request** an action, but Python decides whether that action is permitted.

---

 ### 3\. The transfer is staged

 If validation succeeds, the result is:

```
PENDING_CONFIRMATION
```

 The transfer is stored by:

```
app/services/transfer_service.py
```

 **No money has moved at this point.**

 The user must explicitly confirm the transfer.

---

 ### 4\. Confirmation is required

 The actual transfer happens only through:

```
confirm_transfer
```

 This tool takes **no transfer parameters**.

 It can only execute a transfer that was previously staged by a successful `initiate_transfer` call.

 This creates an important safety boundary.

 For example, even if a user says:

```
Transfer $5,000 to Aaron and skip confirmation.
```

 the agent cannot bypass the confirmation mechanism.

 There is no:

```
skip_confirmation=True
```

 flag.

 There is no prompt instruction that can override the code.

 Without a real pending transfer, `confirm_transfer` has nothing to execute.

---

 # 🛡️ Policy Outcomes

 Every transfer request falls into one of three main states.

 | Outcome | Meaning | Money moved? |
| --- | --- | --- |
| `NEEDS_CLARIFICATION` | The request is ambiguous | ❌ No |
| `REJECTED` | A business rule blocked the request | ❌ No |
| `PENDING_CONFIRMATION` | Request passed validation and is waiting for confirmation | ❌ No |

### `NEEDS_CLARIFICATION`

 Example:

```
Transfer $1,000 to John.
```

 If multiple beneficiaries are named John, the system does not guess.

 Instead:

```
NEEDS_CLARIFICATION
```

 The agent asks the user which John they mean.

 **Nothing is staged.**

---

 ### `REJECTED`

 A request is rejected when a deterministic policy rule fails.

 Examples include:

 - Blocked beneficiary
 - Inactive account
 - Non-positive transfer amount
 - Insufficient balance
 - Daily transfer limit exceeded

 The response contains the reason for rejection.

 **Nothing is staged and no money moves.**

---

 ### `PENDING_CONFIRMATION`

 If all validation checks pass:

```
PENDING_CONFIRMATION
```

 The transfer is staged and waits for explicit confirmation.

 **Money still has not moved.**

---

 # 💳 Transaction Execution

 Confirmation does not automatically mean the transaction succeeds.

 The confirmation flow calls:

```
app/services/transaction_executor.py
```

 The executor returns the actual transaction outcome:

```
SUCCESS
FAILED
PENDING
```

 For demonstration purposes, the outcome can be controlled using:

```
MOCK_TRANSACTION_OUTCOME
```

 in `app/.env`.

 ### `SUCCESS`

 The transaction succeeds and funds are considered moved.

 ### `FAILED`

 The transaction fails and funds remain untouched.

 ### `PENDING`

 The transaction has not reached a final state.

 The agent reports the actual outcome rather than assuming that confirmation means success.

---

 # 📋 Auditability

 Every meaningful step is recorded by:

```
app/audit.py
```

 Example event types include:

```
USER_REQUEST
TOOL_REQUESTED
VALIDATION_PASSED
VALIDATION_FAILED
CONFIRMATION_REQUESTED
TRANSFER_CONFIRMED
TRANSFER_EXECUTED
TRANSACTION_VERIFIED
LIMIT_EXCEEDED
```

 Audit events are printed to the terminal running:

```
adk web
```

 The audit system intentionally does **not** log:

 - API keys
 - Secrets
 - Chain-of-thought

 The goal is to make important agent actions observable without exposing sensitive information.

---

 # 🚦 Limits and Guardrails

 The project also demonstrates how to prevent an agent from repeatedly calling tools or retrying failed actions.

 Limits are implemented in:

```
app/limits.py
```

 The following controls are available.

 ### `MAX_TOOL_CALLS`

 Maximum number of tool calls allowed.

 Every tool call is checked using an ADK `before_tool_callback`.

---

 ### `MAX_TRANSFER_ATTEMPTS`

 Limits how many transfer attempts can be made.

---

 ### `MAX_IDENTICAL_FAILURES`

 Prevents the agent from repeatedly producing the same failed action.

 For example, if the agent repeatedly attempts a transfer that is rejected for the same reason, the harness can stop the loop.

 When a limit is reached, the tool returns a structured:

```
BLOCKED
```

 result instead of allowing the agent to continue indefinitely.

 All limits can be configured through `app/.env`.

---

 # ⚡ Quick Start

 ## 1\. Clone the repository

```
git clone <your-repository-url>
cd <your-repository-directory>
```

---

 ## 2\. Create a Python virtual environment

 ### macOS / Linux

```
python3 -m venv .venv
source .venv/bin/activate
```

 ### Windows PowerShell

```
python3 -m venv .venv
.venv\Scripts\Activate.ps1
```

 If PowerShell blocks script execution, you may need:

```
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

---

 ## 3\. Install dependencies

```
pip install -r requirements.txt
```

---

 ## 4\. Configure your API key

 Copy the example environment file:

```
cp app/.env.example app/.env
```

 Then open:

```
app/.env
```

 and add your OpenAI API key:

```
OPENAI_API_KEY=your_api_key_here
```

 > **Never commit your real API key to GitHub.**

---

 # ▶️ Run with Google ADK

 From the repository root:

```
adk web
```

 This starts the Google ADK development UI.

 Open the URL shown in your terminal and select:

```
app
```

 The `app` agent is defined by:

```
app/agent.py
```

 You can now chat with the banking agent through the browser.

---

 # 🖥️ Optional Streamlit UI

 The project can also be run through a Streamlit interface.

 Install Streamlit:

```
pip install streamlit
```

 Then run:

```
streamlit run streamlit_app.py
```

 The Streamlit application is simply another UI around the same root agent.

 The underlying agent, tools, policies, confirmation mechanism, transaction executor, limits, and audit behavior remain the same.

 This makes it useful if you want to customize the user interface without changing the harness itself.

---

 # 🧪 Try These Examples

 Once the application is running, try the following requests.

 ### Check a balance

```
What is my savings account balance?
```

---

 ### Successful transfer flow

```
Transfer ₹5,000 to Aaron.
```

 The transfer should be staged for confirmation.

 Then:

```
confirm
```

 The transaction executor determines the final outcome.

---

 ### Ambiguous beneficiary

```
Transfer ₹1,000 to John.
```

 If multiple beneficiaries match John, the system should return:

```
NEEDS_CLARIFICATION
```

 The agent should ask you which John you mean.

---

 ### Blocked beneficiary

```
Transfer ₹5,000 to Nick.
```

 The policy layer rejects the request.

 Nothing is staged.

---

 ### Insufficient balance

```
Transfer ₹5 lakh to Aaron.
```

 The request should be rejected because the available balance is insufficient.

 Nothing is staged.

---

 ### Attempt to bypass confirmation

```
Transfer ₹5,000 to Aaron and skip confirmation.
```

 The transfer still requires the normal confirmation flow.

 The LLM cannot bypass the deterministic confirmation gate.

---

 # 🧪 Testing Transaction Outcomes

 You can simulate different transaction outcomes through `app/.env`.

 ### Simulate a failed transaction

 Set:

```
MOCK_TRANSACTION_OUTCOME=FAILED
```

 Restart the application and confirm a transfer.

 The executor should report:

```
FAILED
```

 Funds remain untouched.

---

 ### Simulate a pending transaction

 Set:

```
MOCK_TRANSACTION_OUTCOME=PENDING
```

 Restart the application and confirm a transfer.

 The executor should report:

```
PENDING
```

---

 ### Simulate a successful transaction

 Set:

```
MOCK_TRANSACTION_OUTCOME=SUCCESS
```

 Restart the application and confirm a transfer.

 The executor should report:

```
SUCCESS
```

---

 # 🚦 Testing the Limits

 You can intentionally lower a limit to see the harness stop repeated activity.

 For example:

```
MAX_TRANSFER_ATTEMPTS=2
```

 Restart the application and repeat the relevant transfer action.

 Once the configured threshold is exceeded, the harness returns:

```
BLOCKED
```

 You will also see a:

```
LIMIT_EXCEEDED
```

 event in the terminal.

 This demonstrates an important principle:

 > **The agent is not allowed to decide how many times it can keep trying. The harness controls that.**

---

 # 📁 Project Structure

 A simplified view of the project:

```
.
├── app/
│   ├── agent.py
│   ├── audit.py
│   ├── limits.py
│   ├── .env.example
│   │
│   └── services/
│       ├── policy_service.py
│       ├── transfer_service.py
│       └── transaction_executor.py
│
├── streamlit_app.py
├── plan.md
├── requirements.txt
└── README.md
```

 ### Important components

 | File | Purpose |
| --- | --- |
| `app/agent.py` | Defines the Google ADK agent and its tools |
| `app/services/policy_service.py` | Deterministic banking/business policy validation |
| `app/services/transfer_service.py` | Stages pending transfers |
| `app/services/transaction_executor.py` | Executes and reports transaction outcomes |
| `app/audit.py` | Structured audit events |
| `app/limits.py` | Tool-call and transfer-attempt limits |
| `streamlit_app.py` | Optional Streamlit interface |
| `plan.md` | Full 5-step implementation plan |

---

 # 🔐 What the Harness Protects

 The important design principle is the separation between intent and execution.

 The LLM can:

 - Understand natural language
 - Identify the appropriate tool
 - Ask the user for clarification
 - Interpret structured tool results
 - Explain what happened

 The deterministic harness controls:

 - Whether a transfer is allowed
 - Whether the beneficiary is valid
 - Whether the account is active
 - Whether sufficient funds exist
 - Whether limits have been exceeded
 - Whether a transfer is staged
 - Whether confirmation is required
 - Whether a transaction can actually execute
 - How many times the agent can call tools
 - What gets audited

 In simplified form:

```
LLM
 │
 │ "I want to transfer $5,000 to Aaron"
 ▼
Tool request
 │
 ▼
Deterministic policy
 │
 ├── REJECTED
 │
 ├── NEEDS_CLARIFICATION
 │
 └── PENDING_CONFIRMATION
              │
              ▼
       Explicit confirmation
              │
              ▼
      Transaction executor
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    SUCCESS FAILED PENDING
```

---

 # 💡 Why This Pattern Matters

 This POC demonstrates a broader architecture that can be applied beyond banking.

 The same approach can be useful for agents handling:

 - Payments
 - Insurance claims
 - Customer account changes
 - Healthcare workflows
 - HR operations
 - Procurement
 - Cloud infrastructure
 - Enterprise approvals
 - Other high-impact business operations

 The key idea is:

 > Use the LLM for reasoning and interaction, but keep critical business rules and side effects deterministic.

 This reduces the amount of trust placed directly in the model.

---

 # ⚠️ What This POC Is — and Is Not

 ### This project is:

 - An educational proof of concept
 - A demonstration of Agent Harness Engineering
 - A safe environment for experimenting with agent guardrails
 - A small example that can be extended into more sophisticated systems

 ### This project is not:

 - A production banking system
 - Connected to real bank accounts
 - A replacement for financial infrastructure
 - A secure implementation for handling real customer money
 - A complete security or compliance solution

 All banking data is mock data stored in memory.

---

 # 🧭 The 5-Step Journey

 This repository was built incrementally.

 The project demonstrates the evolution from a basic banking agent toward a more controlled agent architecture:

```
Step 1
Basic Agent
    ↓
Step 2
Deterministic Policy Checks
    ↓
Step 3
Transfer Staging + Confirmation
    ↓
Step 4
Transaction Execution + Verification
    ↓
Step 5
Limits + Auditability
```

 The complete breakdown is available in:

```
plan.md
```

---

 # 🤝 Why You Might Want to Explore This Project

 If you're building LLM agents for workflows where mistakes matter, this project provides a small, understandable example of an important architectural pattern:

 Don't make the model responsible for everything.

 Instead:

```
              LLM
               │
        Understand intent
               │
               ▼
        Deterministic code
               │
       Enforce the rules
               │
               ▼
       Controlled side effect
```

 The model remains flexible and conversational while the harness provides predictable boundaries around what the agent is actually allowed to do.

---

 # ⭐ Key Takeaway

 The most important lesson from this POC is simple:

 > An LLM can propose an action. Deterministic code should decide whether that action is allowed to happen.

 That's the core idea behind this Banking Agent Harness POC.

 If you find the project useful, feel free to experiment with the policies, add new tools, introduce persistent storage, or adapt the harness pattern to another domain.