# A2A OrderBot Mule Application

OrderBot is a Mule 4 Agent-to-Agent (A2A) service that captures pizza restaurant orders through natural, multi-turn conversations. It registers an A2A card named `a2a-orderbot`, listens for task requests, preserves per-session history, and defers response generation to a Large Language Model (LLM) that is prompted with the restaurant menu.

## System Overview

The runtime is composed of the following building blocks:

- **`a2a-orderbot-flow`** (in `src/main/mule/implementation.xml`) – Accepts A2A tasks, normalizes payloads, sets the system prompt, and orchestrates calls to the LLM.
- **Conversation history store** – A persistent Mule Object Store (`ConversationHistoryStore`) keyed by `sessionId` keeps prior assistant/user messages so that each turn remains contextual.
- **HTTP integrations** – The flow exposes an HTTP listener for incoming A2A calls and reaches out to the configured LLM router via HTTPS to obtain completions.
- **Configuration layers** – Shared listener/request settings live in `global.xml`, while environment-specific properties are provided through YAML files in `src/main/resources/config`.
- **Logging** – `src/main/resources/log4j2.xml` controls log levels and appenders for troubleshooting the flow while running locally or in CloudHub.

### Message lifecycle

1. The `a2a:task-listener` receives the inbound request and extracts the user utterance plus a `sessionId` (or generates one when missing).
2. A DataWeave script seeds the prompt with the pizza menu and conversational guidelines before combining it with any stored history.
3. The flow POSTs the assembled messages to `/dev-llm-router-v1/api/v1/chat/completions` on the external router with the required credentials.
4. The chosen assistant response is appended to the history and persisted for future turns.
5. A normalized A2A message structure is returned to the caller, including HTTP headers that enable cross-origin testing clients.

## Project Layout

```
a2a-orderbot/
├── LICENSE
├── README.md
├── pom.xml
├── src/
│   ├── main/
│   │   ├── mule/
│   │   │   ├── global.xml
│   │   │   └── implementation.xml
│   │   ├── resources/
│   │   │   ├── config/
│   │   │   │   ├── a2a-orderbot.yaml
│   │   │   │   └── a2a-orderbot.dev.yaml
│   │   │   └── log4j2.xml
└── exchange-docs/
```

## Configuration

The application loads configuration from `src/main/resources/config/a2a-orderbot.yaml` plus an optional overlay matching the value of the `mule.env` runtime property (for example `a2a-orderbot.dev.yaml`).

| Property | Default | Description |
| --- | --- | --- |
| `http.host` | `0.0.0.0` | Interface that exposes the HTTP listener used by the A2A server. |
| `http.port` | `8081` | Port for incoming A2A requests (listener path `/a2a-orderbot`). |
| `llm.host` | – | Hostname for the LLM router that proxies completions. Define in the environment-specific YAML. |
| `llm.port` | – | Port for the LLM router (typically `443` for HTTPS). |
| `llm.router.client-id` | – | Credential passed in the `client-id` header for LLM requests. Provide via secure property placeholder or environment variable. |
| `llm.router.client-secret` | – | Credential passed in the `client-secret` header. Provide securely at runtime. |

> **Note:** Credentials are intentionally omitted from the repository. Supply them through a secure property file (for example `a2a-orderbot.secure.yaml`) or by setting JVM/Mule properties such as `-Dllm.router.client-id=...`.

## Local Development

1. **Install prerequisites**
   - Mule runtime 4.8.x or newer (via Anypoint Studio 7.x or the Mule Maven plugin).
   - Access to an HTTPS endpoint that implements `/dev-llm-router-v1/api/v1/chat/completions`.
   - Valid credentials for the LLM router to satisfy the `client-id` and `client-secret` headers.

2. **Configure properties**
   - Update `src/main/resources/config/a2a-orderbot.yaml` if you need a different listener host/port.
   - Duplicate `a2a-orderbot.dev.yaml` (or create another `<env>` file) to point at your LLM router and to reference where credentials will be sourced.
   - Provide secure values for `llm.router.client-id` and `llm.router.client-secret` using the method of your choice (Secure Properties, Runtime Manager, or JVM arguments).

3. **Run the app**
   - In Anypoint Studio, right-click the project and choose **Run project**.
   - For Maven execution, run `mvn mule:run -Dmule.env=dev` from the repository root after setting the necessary properties.
   - The listener becomes available at `http://<http.host>:<http.port>/a2a-orderbot` and registers the agent card metadata defined in `implementation.xml`.

## Exercising the Flow

Send an A2A-style payload with the most recent user message and an optional `sessionId` to reuse previous context. The snippet below demonstrates a minimal `curl` request:

```bash
curl -X POST "http://localhost:8081/a2a-orderbot" \
  -H "Content-Type: application/json" \
  -d '{
        "message": {
          "parts": [ { "kind": "text", "text": "I would like a large pepperoni pizza." } ]
        },
        "configuration": {
          "sessionId": "demo-session"
        }
      }'
```

The service forwards the compiled conversation to the LLM, appends the assistant reply to the persisted history, and responds with a JSON body where `parts[0].text` holds the assistant message.

## Troubleshooting

- Use Anypoint Studio's console or the configured Log4j2 appenders to inspect runtime logs.
- Clear the `ConversationHistoryStore` (object store) if you need to reset a conversation between tests.
- Ensure the external LLM router allows the configured `client-id`/`client-secret` pair and supports streaming responses when requested by the card.

## License

This project is licensed under the [MIT License](./LICENSE).
