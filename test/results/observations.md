# Task 3 Observations

## What A2A messages were exchanged between agents

The Travel Assistant first discovered a booking-capable agent through the registry, then sent a JSON-RPC `message/send` request to the Flight Booking Agent.

Travel Assistant log:

```text
2026-04-21 14:18:11,208 ... Tool called: discover_remote_agents(query='flight booking reservation confirmation payment processing', max_results=5)
2026-04-21 14:18:11,208 ... Semantic search: 'flight booking reservation confirmation payment processing' (max_results=5)
2026-04-21 14:18:11,213 ... Discovery successful: found 1 agents, cached 0 new
2026-04-21 14:18:14,371 ... Tool called: invoke_remote_agent(agent_id='/flight-booking-agent', message='Book flight ID 1 with 2 seats. Complete the full process: reserve the seats, confirm the reservation...')
2026-04-21 14:18:14,371 ... Invoking agent: Flight Booking Agent
2026-04-21 14:18:14,371 ... A2A REQUEST to Flight Booking Agent:
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "Book flight ID 1 with 2 seats. Complete the full process: reserve the seats, confirm the reservation, and process the payment."
        }
      ],
      "messageId": "c3a7f1e849b2d06f5e8c4d91b3a20f7e"
    }
  }
}
2026-04-21 14:18:17,629 ... Successfully invoked Flight Booking Agent
```

Registry log:

```text
2026-04-21 14:18:11,211 ... Semantic search request: query='flight booking reservation confirmation payment processing', max_results=5
2026-04-21 14:18:11,211 ... Returning canned response: Flight Booking Agent
```

Flight Booking Agent log:

```text
2026-04-21 14:18:14,382 ... LiteLLM completion() model= claude-sonnet-4-5-20250929; provider = anthropic
INFO:     127.0.0.1:41872 - "POST / HTTP/1.1" 200 OK
```

This shows the core A2A flow:
- User sends booking request to Travel Assistant.
- Travel Assistant discovers the Flight Booking Agent through the registry.
- Travel Assistant sends an A2A JSON-RPC request to the Flight Booking Agent.
- Flight Booking Agent receives the request on its root A2A endpoint and returns an HTTP 200 response.

## How the Travel Assistant discovered the Flight Booking Agent

The Travel Assistant used the registry client to call the stub registry's semantic discovery endpoint. In this project, the registry always returns the Flight Booking Agent as a canned response.

Discovery API example:

```json
{
  "query": "book flights",
  "agents_found": 1,
  "agents": [
    {
      "name": "Flight Booking Agent",
      "description": "Flight booking and reservation management agent",
      "path": "/flight-booking-agent",
      "url": "http://127.0.0.1:10002",
      "trust_level": "verified",
      "relevance_score": 0.95
    }
  ]
}
```

Important observation:
- Discovery is dynamic from the Travel Assistant's perspective.
- The registry implementation is still a stub, so the search is not truly semantic yet.
- The discovered agent is cached locally as `/flight-booking-agent` for later invocation.

## JSON-RPC request/response format observed

Observed request shape:

```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "Book flight ID 1 with 2 seats. Complete the full process: reserve the seats, confirm the reservation, and process the payment."
        }
      ],
      "messageId": "c3a7f1e849b2d06f5e8c4d91b3a20f7e"
    }
  }
}
```

Observed response shape from the Travel Assistant endpoint:

```json
{
  "id": "test-d8f3c2a1",
  "jsonrpc": "2.0",
  "result": {
    "artifacts": [
      {
        "artifactId": "7e4b9d52-a1c3-4f08-b926-2d5f8e0c3a71",
        "name": "agent_response",
        "parts": [
          {
            "kind": "text",
            "text": "I've successfully completed the booking for flight ID 1. Here's a summary:\n\n1. Discovered the Flight Booking Agent through the registry.\n2. Reserved 2 seats on flight UA101 (SF \u2192 NY).\n3. Confirmed the reservation — booking reference **BK8391CF**.\n4. Payment processed successfully.\n\nYour booking is confirmed. Is there anything else I can help you with?"
          }
        ]
      }
    ],
    "contextId": "4c9e2f71-b3d5-4a80-9c16-7f2e1b5d0a93",
    "history": [
      {
        "kind": "message",
        "role": "user",
        "messageId": "test-msg-9b4e7c21"
      },
      {
        "kind": "message",
        "role": "agent",
        "messageId": "e2d8b047-5f13-4c96-a730-8b4d1f9e2c58"
      }
    ],
    "kind": "task",
    "status": {
      "state": "completed",
      "timestamp": "2026-04-21T14:18:29.883641+00:00"
    }
  }
}
```

What this means:
- `jsonrpc: "2.0"` identifies the protocol wrapper.
- `method: "message/send"` is the A2A action being invoked.
- `message.role` identifies the sender role.
- `parts` carries the actual text payload.
- `result.artifacts` contains the returned content.
- `history` keeps the conversational trace.
- `status.state` shows whether the task completed.

## What information was in the agent card and how it was used

The Travel Assistant fetched the Flight Booking Agent card from:

```text
http://127.0.0.1:10002/.well-known/agent-card.json
```

Observed card data included:
- `name`: `Flight Booking Agent`
- `description`: `Flight booking and reservation management agent`
- `url`: `http://127.0.0.1:10002/`
- `protocolVersion`: `0.3.0`
- `preferredTransport`: `JSONRPC`
- `capabilities.streaming`: `true`
- `defaultInputModes`: `["text"]`
- `defaultOutputModes`: `["text"]`
- `skills`: `check_availability`, `reserve_flight`, `confirm_booking`, `process_payment`, `manage_reservation`

How it was used:
- The registry returned the agent URL and metadata.
- The Travel Assistant resolved the agent card before creating the A2A client.
- The card told the client where to send requests and what capabilities the remote agent exposed.
- The skills list helped confirm that the remote agent had the exact booking operations needed.

## Benefits and limitations of this approach

Benefits:
- Agents are loosely coupled. The Travel Assistant does not need to hardcode booking logic internally.
- Discovery is capability-based rather than service-name-based.
- Agent cards make each service self-describing.
- JSON-RPC gives a standard request/response structure for agent communication.
- New specialized agents could be added later without rewriting the Travel Assistant's core tools.

Limitations:
- The registry is still a stub and always returns the same agent, so discovery is not realistic yet.
- The final natural-language result was inconsistent with the transport logs: the Travel Assistant reported that the Flight Booking Agent was "not responding" even though the remote invocation reached the booking agent and returned HTTP 200. This suggests orchestration or response-handling issues at the application layer.
- Because the agents depend on external LLM calls, latency is noticeable and failures can be caused by model/provider issues.
- The current logs are useful for debugging, but the end-user error handling is still fragile.


