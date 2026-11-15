# Runtime Intake

## When to ask

Use this catalog only after inspecting the request and evidence. Ask an entry
when its `ask_when` condition is true and the answer would materially affect
the result. Skip answered or irrelevant entries. Prioritize safety, scope,
platform or framework basis, evidence quality, then output preference.

## Tool adaptation

- Claude: select at most three neutral entries, project each to only `question`,
  `header`, and `options`, then add `multiSelect: false`; do not send `id` or
  `ask_when`.
- Codex: select at most three neutral entries and project each to only `id`,
  `header`, `question`, and `options`; do not send `ask_when` or `multiSelect`.
- Fallback: ask the same questions in concise plain text with a free-text
  `Other` path.
- Never request secrets.

## Question catalog

```json
{
  "questions": [
    {
      "id": "ipsecb_oob",
      "ask_when": "Console or out-of-band access to both hub nodes and the spoke is not confirmed.",
      "header": "OOB Access",
      "question": "Is console or out-of-band access available on every device?",
      "options": [
        {
          "label": "All devices reachable (Recommended)",
          "description": "Proceed, because a bad commit can be recovered from the console."
        },
        {
          "label": "Some devices only",
          "description": "Stop until every hub node and the spoke has out-of-band access."
        },
        {
          "label": "No out-of-band access",
          "description": "Stop, because a commit that cuts off management may not revert in time."
        }
      ]
    },
    {
      "id": "ipsecb_goal",
      "ask_when": "The optimisation goal for this run has not been chosen.",
      "header": "Goal",
      "question": "What should this IPsec build optimise for?",
      "options": [
        {
          "label": "Least planned loss (Recommended)",
          "description": "Turn on process-packet-on-backup with floating statics, accepting slower tunnel BGP recovery and possible replay log messages."
        },
        {
          "label": "Fastest BGP recovery",
          "description": "Leave the flag off and use floating statics, accepting more hub-to-spoke loss on planned failover."
        },
        {
          "label": "Quiet logs",
          "description": "Turn the flag on and disable anti-replay on the spoke, removing replay protection on this tunnel."
        }
      ]
    },
    {
      "id": "ipsecb_window",
      "ask_when": "No hub node has managed-services ipsec yet and the change timing is not agreed.",
      "header": "SRG Restart",
      "question": "When can the SRG restart caused by adding managed-services ipsec happen?",
      "options": [
        {
          "label": "Maintenance window (Recommended)",
          "description": "Push the hub during an agreed window, backup node first."
        },
        {
          "label": "During initial MNHA build",
          "description": "The pair carries no production traffic yet, so the restart is harmless."
        },
        {
          "label": "No window available",
          "description": "Stop before Phase A, because the active node withdrew its VIP for minutes in testing."
        }
      ]
    },
    {
      "id": "ipsecb_commit",
      "ask_when": "The connected Junos MCP server's commit-confirmed support is unknown.",
      "header": "Commit Mode",
      "question": "Does the connected Junos MCP server support commit confirmed?",
      "options": [
        {
          "label": "Check the tool list (Recommended)",
          "description": "Inspect the server's tools and map them using the MCP server notes."
        },
        {
          "label": "Commit confirmed available",
          "description": "Push every file with a confirm timer and confirm after verification."
        },
        {
          "label": "No commit confirmed",
          "description": "Hand the cutover and the uplink-down test to the user at the CLI."
        }
      ]
    },
    {
      "id": "ipsecb_test",
      "ask_when": "The failover test scope after the build is absent.",
      "header": "Failover",
      "question": "Which failover tests should follow the build?",
      "options": [
        {
          "label": "Planned and unplanned (Recommended)",
          "description": "Measure a baseline, then planned switchover and uplink-down tests in both directions."
        },
        {
          "label": "Planned only",
          "description": "Measure a baseline and planned switchover and failback in both directions."
        },
        {
          "label": "Build only",
          "description": "Stop after the tunnel and cutover are verified."
        }
      ]
    }
  ]
}
```
