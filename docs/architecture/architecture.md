---

## Architecture

The customer-support workflow follows this high-level architecture:

```text
                         Customer Message
                                │
                                ▼
                     ┌────────────────────┐
                     │   RequestClassifier │
                     │   Amazon Bedrock    │
                     └──────────┬─────────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │ RequestTypeCondition │
                    │   Conditional Router │
                    └──────────┬───────────┘
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
           BUG                FAQ          Other Request
             │                 │                 │
             ▼                 ▼                 ▼
     FormatBugResponse      FAQAnswer       OtherRequest
             │                 │                 │
             ▼                 │                 │
       BugReportAgent          │                 │
             │                 │                 │
             ▼                 │                 │
       Create Bug Report       │                 │
             │                 │                 │
             ▼                 │                 │
        DynamoDB               │                 │
             │                 │                 │
             └─────────────────┴─────────────────┘
                               │
                               ▼
                        Customer Response
