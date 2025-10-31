# A2A HelloWorld Server

A simple Mule A2A (Agent-to-Agent) server that responds with "Hi!" message.

## Overview

This Mule application implements an A2A server named `a2a-helloworld` that provides a greeting service. When invoked, it returns a simple "Hi!" payload.

## Configuration

- **Server Name:** a2a-helloworld
- **Port:** 8081
- **Endpoint:** http://localhost:8081/a2a-helloworld/
- **Version:** 1.0.0

## Agent Card

The A2A server exposes the following capability:

- **Skill:** greeting (ID: 1)
  - **Description:** Returns a simple greeting

## How to Run

### Prerequisites

- Anypoint Studio 7.x or later
- Mule Runtime 4.8.0 or later
- A2A Connector installed

### Running the Application

1. Import the project into Anypoint Studio
2. Run the application
3. The server will start on port 8081

### Testing

Send a request to the A2A server:

```bash
curl http://localhost:8081/a2a-helloworld
```

Expected response: `Hi!`

## Project Structure

```
a2a-helloworld/
├── src/
│   └── main/
│       └── mule/
│           ├── a2a-helloworld.xml
│           └── mule-artifact.json
├── pom.xml
└── README.md
```

## Dependencies

- Mule HTTP Connector
- Mule Sockets Connector
- Mule A2A Connector

## Notes

- The HTTP listener is configured to accept connections from any host (0.0.0.0)
- The agent path is set to `/a2a-helloworld`
- The flow uses the `a2a:task-listener` to handle incoming A2A tasks

