# Automated Testing of GraphQL Subscriptions over WebSockets

## 1. Overview of GraphQL Subscriptions

While GraphQL Queries and Mutations follow standard stateless HTTP request-response patterns, **GraphQL Subscriptions** maintain persistent, bidirectional connections (typically over WebSockets) allowing servers to push real-time updates to clients whenever specific server events occur.

Common use cases include:
- Real-time stock ticker updates and order book changes
- Multi-user collaborative document editing
- Live customer chat and support messages
- System alerting and workflow state updates

From an SQA perspective, testing GraphQL Subscriptions requires verifying:
- Proper connection handshakes and authentication over the `graphql-transport-ws` subprotocol.
- Event filtering and argument parameters (e.g., subscribing only to messages in a specific channel).
- Event delivery order and data schema conformance.
- Reconnection resilience and connection cleanup to avoid memory leaks on the server.

```
┌────────────────────────────────────────────────────────┐
│                        Client                          │
└───────┬───────────────────┬───────────────────▲────────┘
        │                   │                   │
  1. WS Connect       2. Subscribe        3. Subscription
  (ConnectionInit)    (orderCreated)         Event Stream
        │                   │                   │
        ▼                   ▼                   │
┌───────────────────────────────────────────────┴────────┐
│               GraphQL Server (Subscription Engine)     │
│   - Validates JWT in connection_init payload           │
│   - Registers subscriber in PubSub broker              │
│   - Streams JSON payloads upon mutation triggers       │
└────────────────────────────────────────────────────────┘
```

---

## 2. Protocols: `graphql-ws` vs Legacy `subscriptions-transport-ws`

| Protocol Feature | Legacy (`subscriptions-transport-ws`) | Modern Standard (`graphql-ws`) |
| :--- | :--- | :--- |
| **Subprotocol Header** | `graphql-ws` | `graphql-transport-ws` |
| **Init Message Type** | `connection_init` | `connection_init` |
| **Ack Message Type** | `connection_ack` | `connection_ack` |
| **Subscribe Type** | `start` | `subscribe` |
| **Data Payload Type**| `data` | `next` |
| **Completion Type** | `complete` | `complete` |
| **Active Maintenance**| Deprecated / Archived | Active (W3C / GraphQL Foundation) |

---

## 3. Automated Test Suite with TypeScript & `graphql-ws`

Below is an automated test suite verifying subscription delivery using Jest and the official `graphql-ws` client.

```typescript
import { createClient, Client } from 'graphql-ws';
import WebSocket from 'ws';
import axios from 'axios';

describe('GraphQL Subscriptions Automated Verification', () => {
    let wsClient: Client;
    const WS_URL = 'ws://localhost:4000/graphql';
    const HTTP_URL = 'http://localhost:4000/graphql';

    beforeEach(() => {
        // Initialize client with modern graphql-transport-ws protocol
        wsClient = createClient({
            url: WS_URL,
            webSocketImpl: WebSocket,
            connectionParams: {
                authorization: 'Bearer valid_qa_token_jwt',
            },
        });
    });

    afterEach(async () => {
        if (wsClient) {
            await wsClient.dispose();
        }
    });

    test('should receive published event when mutation is executed', async () => {
        const receivedEvents: any[] = [];
        let unsubscribe: () => void = () => {};

        // Subscription query
        const subscriptionQuery = `
            subscription OnOrderCreated($market: String!) {
                orderCreated(market: $market) {
                    id
                    symbol
                    price
                    quantity
                    status
                }
            }
        `;

        // Step 1: Establish subscription and listen for events
        const eventReceivedPromise = new Promise<void>((resolve, reject) => {
            const timeout = setTimeout(() => reject(new Error('Subscription event timed out')), 5000);

            unsubscribe = wsClient.subscribe(
                {
                    query: subscriptionQuery,
                    variables: { market: 'CRYPTO-BTC' },
                },
                {
                    next: (event) => {
                        receivedEvents.push(event.data);
                        clearTimeout(timeout);
                        resolve();
                    },
                    error: (err) => reject(err),
                    complete: () => {},
                }
            );
        });

        // Step 2: Trigger mutation that produces the event
        const mutationQuery = `
            mutation CreateOrder($input: OrderInput!) {
                createOrder(input: $input) {
                    id
                    status
                }
            }
        `;

        const mutationResponse = await axios.post(HTTP_URL, {
            query: mutationQuery,
            variables: {
                input: {
                    market: 'CRYPTO-BTC',
                    symbol: 'BTC/USD',
                    price: 65000.50,
                    quantity: 1.5,
                },
            },
        });

        expect(mutationResponse.status).toBe(200);
        expect(mutationResponse.data.data.createOrder.status).toBe('CREATED');

        // Step 3: Wait for subscription event to arrive over WebSocket
        await eventReceivedPromise;

        // Step 4: Validate subscription payload contents
        expect(receivedEvents).toHaveLength(1);
        const eventData = receivedEvents[0].orderCreated;
        expect(eventData.symbol).toBe('BTC/USD');
        expect(eventData.price).toBe(65000.50);
        expect(eventData.status).toBe('CREATED');

        // Step 5: Clean up subscription
        unsubscribe();
    });

    test('should reject connection when authorization token is missing', async () => {
        const unauthClient = createClient({
            url: WS_URL,
            webSocketImpl: WebSocket,
            connectionParams: {
                authorization: '', // Invalid / empty token
            },
        });

        await expect(new Promise((resolve, reject) => {
            unauthClient.subscribe(
                { query: 'subscription { orderCreated(market: "ALL") { id } }' },
                {
                    next: resolve,
                    error: reject,
                    complete: () => {},
                }
            );
        })).rejects.toBeDefined();

        await unauthClient.dispose();
    });
});
```

---

## 4. Key QA Verification Scenarios for Subscriptions

| Scenario | Test Action | Expected Result |
| :--- | :--- | :--- |
| **Parameter Filtering** | Subscribe to channel `A`; trigger mutation in channel `B` | Subscriber to `A` receives zero events |
| **Authentication Expiry** | Expire JWT token mid-stream | Server closes socket with close code `4401 Unauthorized` |
| **High Event Throughput** | Server fires 1,000 mutations in 5 seconds | Client receives all 1,000 events without dropped packets or socket crash |
| **Abrupt Client Disconnect** | Force kill client TCP socket | Server cleanly deregisters subscriber from Redis pub/sub without leaking memory |

---

## 5. SQA Interview Questions & Answers

### Q1: Why do GraphQL Subscriptions typically use WebSockets instead of HTTP Long Polling or Server-Sent Events (SSE)?
> **Answer**:
> WebSockets provide a low-latency, bidirectional full-duplex TCP channel that allows clients to initialize connections, authenticate dynamically, send multiple concurrent subscription operations, and terminate individual subscriptions over a single socket connection. 
> While **Server-Sent Events (SSE)** are increasingly popular for unidirectional streaming over HTTP/2, WebSockets have historically been the industry standard for GraphQL subscriptions.

### Q2: What is the risk of not testing subscription disposal/unsubscription on the client?
> **Answer**:
> When clients navigate between pages in Single Page Applications (SPAs) without calling `unsubscribe()` or disposing the WebSocket client, the server keeps active event listeners registered in memory and in the backend PubSub broker (e.g., Redis). Over time, this causes **memory leaks**, exhausted connection pools, and degraded broker throughput.

---

## 6. Key Takeaways & Best Practices

- Migrate automated test suites and clients to the modern `graphql-ws` library using the `graphql-transport-ws` subprotocol.
- Always assert negative authorization checks during the `connection_init` handshake phase.
- Use explicit Promise timeouts in automated tests to catch silent subscription deadlocks.
