# Whatsapp Design

## Functional Requirements

1. Group Chat (100 users)
2. Send/Receive messages 
3. User should be able to receive the messages while they are not online.
4. User should be able to send /receive the media in their messages.

## Non-Functional Requirements

1. Messages should delivered to the users with low latency.
2. Guaranteed delivery of messages.
3. System should be able to handle billions of users.
4. Messages should be stored on centralized server no longer than necessary . 
5. The system should be resilient against failures of individual components.

## Core Entities 
- User
- Chats
- Messages
- Client(User may have mupliple devices)

## API or system Interface

//->CreateChat()
{
    "participants":[],
    name:"",    
}->{
    "chatId":"" 
}

//->SendMessage()
{
    "chatId":"",
    "message":"",
    "attachmentId":""
}
-> "Success | Failure"

//->CreateAttachment()
{
    "body":...,
    "hash":""
}
-> {
    "attachmentId":""
}

//->ModifyChatParticipants
{
    "chatId":"",
    "userId":"",
    "operation":"add | remove"
}
-> "Success | Failure"

// <- newMessage
{
    "chatId": "",
    "userId": ""
    "message": "",
    "attachments": []
} -> "RECEIVED"


//<- ChatUpdate 

{
    "chatId": "",
    "participants": [],
} -> "RECEIVED"


## High Level Design

1) User should be able to start group chat with multiple participants.(limit 100 users)
2) User should be able to send/receive messages.
3) User should be able to receive the messages while they are not online.
4) User should be able to send /receive the media in their messages. 

---

### Requirement #1: Group Chat Creation (Max 100 Users)

#### Architecture Overview

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Client     │         │  L4 Load     │         │  WebSocket   │
│  (Mobile/    │◄───────►│  Balancer    │◄───────►│   Service    │
│   Web)       │         │  (TCP/IP)    │         │              │
└──────────────┘         └──────────────┘         └──────┬───────┘
                                                          │
                                                          ▼
                                                  ┌──────────────┐
                                                  │  DynamoDB    │
                                                  │  - Chats     │
                                                  │  - ChatPart. │
                                                  └──────────────┘
```

#### Key Components

| Component | Purpose | Why? |
|-----------|---------|------|
| **L4 Load Balancer** | Route WebSocket connections | Maintains persistent TCP connections |
| **WebSocket Service** | Handle real-time messaging | Bi-directional, low-latency communication |
| **DynamoDB** | Store chat metadata | Fast key-value lookups, auto-scaling |

#### Database Design

##### Table 1: Chats

```javascript
TableName: "Chats"
Primary Key: chatId (Partition Key)

Schema:
{
    "chatId": "chat_12345",              // Unique identifier
    "chatType": "GROUP",                 // GROUP or DIRECT
    "name": "Weekend Plans",             // Group name
    "createdBy": "user_alice",           // Creator
    "createdAt": "2025-12-28T07:51:25Z", // Timestamp
    "participantCount": 4,               // Cache for validation
    "maxParticipants": 100               // Hard limit
}

Access Pattern:
└─ getChatById(chatId) → O(1)
```

##### Table 2: ChatParticipants

```javascript
TableName: "ChatParticipants"
Primary Key (Composite):
├─ Partition Key: chatId
└─ Sort Key: participantId

Schema:
{
    "chatId": "chat_12345",
    "participantId": "user_alice",
    "role": "ADMIN",                     // ADMIN or MEMBER
    "joinedAt": "2025-12-28T07:51:25Z"
}

Storage Structure:
Partition: chat_12345
├─ user_alice
├─ user_bob
├─ user_charlie
└─ user_dave

Access Patterns:
└─ getAllParticipants(chatId) → O(1) range query
```

##### Global Secondary Index: ParticipantIdIndex

```javascript
GSI Name: "ParticipantIdIndex"
Keys:
├─ Partition Key: participantId
└─ Sort Key: chatId
Projection: ALL

Purpose: 
Enable reverse lookup - find all chats for a given user

Storage Structure:
Partition: user_alice
├─ chat_12345
├─ chat_67890
└─ chat_99999

Access Pattern:
└─ getAllChatsForUser(participantId) → O(1) range query
```

#### Create Chat Flow

```
Client            L4 LB        WS Service      DynamoDB
  │                 │              │               │
  │──WebSocket─────>│              │               │
  │                 │──Route──────>│               │
  │                 │              │               │
  │──createChat()──────────────────>               │
  │  {participants:[...]}          │               │
  │                 │              │─Validate─────>│
  │                 │              │ (count<=100)  │
  │                 │              │               │
  │                 │              │─BEGIN TXN────>│
  │                 │              │               │
  │                 │              │─INSERT Chat──>│
  │                 │              │               │
  │                 │              │─INSERT Part.─>│
  │                 │              │  (alice, bob, │
  │                 │              │   ... x100)   │
  │                 │              │               │
  │                 │              │<─COMMIT TXN───│
  │                 │              │               │
  │<─{chatId:"123"}────────────────│               │
  │                 │              │               │
  │                 │              │─Notify────────│
  │                 │              │ (async)       │
```

#### Transaction Implementation

```javascript
// ACID Transaction - All or Nothing
const transaction = {
    TransactItems: [
        // 1. Create Chat record
        {
            Put: {
                TableName: 'Chats',
                Item: {
                    chatId: 'chat_12345',
                    name: 'Weekend Plans',
                    createdBy: 'user_alice',
                    participantCount: 100,
                    maxParticipants: 100
                }
            }
        },
        // 2-101. Create Participant records (up to 100)
        {
            Put: {
                TableName: 'ChatParticipants',
                Item: {
                    chatId: 'chat_12345',
                    participantId: 'user_alice',
                    role: 'ADMIN'
                }
            }
        }
        // ... up to 100 participants
    ]
};

await dynamoDB.transactWrite(transaction).promise();
```

#### Query Patterns & Performance

| Use Case | Query | Complexity | Index Used |
|----------|-------|------------|------------|
| Get chat details | `chatId = "chat_123"` | O(1) | Primary Key |
| Get all participants | `chatId = "chat_123"` | O(1) | Primary Key (range) |
| Get user's all chats | `participantId = "user_alice"` | O(1) | GSI |
| Validate 100 limit | Read `participantCount` | O(1) | Primary Key |

#### Key Design Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **DynamoDB** | Fast key-value, auto-scaling | Cost at scale |
| **Composite Primary Key** | Efficient range queries | Can't query by participantId |
| **GSI on participantId** | Reverse lookup (user's chats) | Extra storage cost |
| **Separate Participants Table** | Scalability (100 participants) | Join complexity |
| **participantCount cache** | Fast validation | Denormalization |
| **L4 Load Balancer** | WebSocket support | Sticky sessions needed |
| **Transactions** | Data consistency | Slightly slower writes |
| **100 user limit** | Prevent notification storm | UX limitation |

---