# Step Functions Cheatsheet

---

## Core Concepts

- Step Functions = **serverless workflow orchestration** — coordinate Lambda, Glue, DynamoDB, etc.
- Defined as a **state machine** in JSON (Amazon States Language / ASL)
- Handles ordering, retries, error handling, parallelism, and visual monitoring
- Built visually in **Workflow Studio** or by editing the JSON definition

---

## State Machine Types

| Type | Use |
| --- | --- |
| **Standard** | Long-running (up to 1 yr), exactly-once, full history — most ETL/orchestration |
| **Express** | High-volume, short (<5 min), cheaper, at-least-once — event streaming |

---

## Common State Types

| State | Purpose |
| --- | --- |
| `Task` | Do work (invoke Lambda, start Glue job, etc.) |
| `Pass` | Pass/inject data, no work |
| `Choice` | Branch on input (if/else) |
| `Parallel` | Run multiple branches at once (fan-out) |
| `Map` | Loop a sub-workflow over an array |
| `Wait` | Delay for a time / until a timestamp |
| `Succeed` / `Fail` | Terminate the execution |

---

## Invoke a Lambda (Task)

```json
{
  "StartAt": "InvokeLambda",
  "States": {
    "InvokeLambda": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "OutputPath": "$.Payload",
      "Parameters": { "FunctionName": "HelloLambda", "Payload": { "name": "Student" } },
      "End": true
    }
  }
}
```

---

## Fan-Out (Parallel)

```json
{
  "StartAt": "ParallelTasks",
  "States": {
    "ParallelTasks": {
      "Type": "Parallel",
      "Branches": [
        { "StartAt": "TaskA", "States": { "TaskA": { "Type": "Task", "Resource": "arn:aws:states:::lambda:invoke", "Parameters": { "FunctionName": "TaskA" }, "End": true } } },
        { "StartAt": "TaskB", "States": { "TaskB": { "Type": "Task", "Resource": "arn:aws:states:::lambda:invoke", "Parameters": { "FunctionName": "TaskB" }, "End": true } } }
      ],
      "End": true
    }
  }
}
```

- Output is an array — one element per branch

---

## Error Handling — Retry & Catch (Saga Pattern)

```json
"Step2": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": { "FunctionName": "FailTask" },
  "Retry":  [ { "ErrorEquals": ["States.ALL"], "MaxAttempts": 2, "IntervalSeconds": 3 } ],
  "Catch":  [ { "ErrorEquals": ["States.ALL"], "Next": "Rollback" } ],
  "End": true
}
```

- **Retry** re-attempts the same state; **Catch** routes failures to a compensating state
- **Saga pattern** = on failure, run rollback/compensation steps to undo prior work

---

## IAM

Execution role needs permission to invoke targets, e.g.:

```json
{ "Effect": "Allow", "Action": "lambda:InvokeFunction", "Resource": "arn:aws:lambda:us-east-1:<acct>:function:*" }
```

---

## Monitoring

- **Executions** → open one → Visual Graph shows success/failure transitions
- Expand each state for Input/Output JSON, duration, retry attempts
- **View logs in CloudWatch** for task details; enable logging in the Config panel