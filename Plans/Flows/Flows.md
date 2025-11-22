# ShopList - Flows

## Overview

ShopList allows customers to collaborate on writing and managing Shop Lists.\
As a B2B service, ShopList provides real-data based statistics. (Note - ask Eden to write something here....)

For that reason, ShopList is divided into two parts:

1. B2C - The known part of ShopList.
2. B2B - Displaying statistics, based on users activities.

> [!NOTE] All events are logged in Kafka.
> Each event has `type`, `data` and `state` to them.\
> When failing service is reinitiated, it will look over all of the `"state": "started"` events, and retry them.
>
> For brevity, kafka logs are omitted from flow diagrams unless it plays a special role beyond standard logging.

> [!NOTE] From Redis to DB
> All writes on Redis are followed with notifications in Kafka for the DB Writer Service to push update
>
> For brevity, update {{DB}} logs are omitted from flow diagram unless it plays a special role beyond standard role.
>
> See [DB Writer Service](#db-writer) for implementation details.

> [!NOTE]
> For best visibility, open this MD file through Visual Studio Code, using the [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)

---

## 1. B2C Flows

This part is divided into 3 categories:

1. **Base App** - consisting of `Auth Service`, `Users Service`, `Lists Service`, `Chat Service`, `DB Writer Service`
2. **Message Process** - `Message Analyzer Service`, `Product History Manager Service`, `Scoring Service`
3. **AI Suggestions** - `AI Suggestions Service`, `Keyword Manager Service`

> [!NOTE] All REST request must have JWT
> All REST API requests pass through Ngnix, which sends them to the Auth Service for JWT token validation.\
> Valid requests are modified to have `user_id` instead of JWT before forwarding to downstream services.
>
> For brevity, Ngnix is omitted from flow diagrams unless it plays a service-specific role beyond standard authentication.
>
> See [Auth Service: JWT to `user_id` Flow](#authentication) for implementation details.

## 1.1. Base App

### Auth Service

Manages Login and JWTs

1. **Login**

   1. Front redirect to Auth0\
      &nbsp;&nbsp; ↓
   2. User Login\
      &nbsp;&nbsp; ↓
   3. JWT is sent to the Auth Service\
      &nbsp;&nbsp; ↓
   4. _#FIGURE OUT WHATS NEXT_

&nbsp;

2. **JWT to `user_id`**
   1. Front sends `REST` request\
      &nbsp;&nbsp; ↓
   2. Ngnix extracts JWT\
      &nbsp;&nbsp; ↓
   3. Auth Service validates JWT, returns `user_id`\
      &nbsp;&nbsp; ↓
   4. Ngnix adds `user_id` to headers\
      &nbsp;&nbsp; ↓
   5. Destination service receives request

---

### Users Service

Manages users, user data, user settings

> [!NOTE]
> This service will be divided into Users Manager and Settings Service, when more settings will be added.

---

### Lists Service

1. **Get Lists** - SSE

   1. Front sends `GET` request\
      &nbsp;&nbsp; ↓
   2. Lists are fetched from Redis.\
      &nbsp;&nbsp; ↓
   3. For every list, items are fetched from Redis - 5 at most.\
      &nbsp;&nbsp; ↓
   4. Lists Service publishes event to Kafka, consumed by Chat Service\
      &nbsp;&nbsp; ↓
   5. **First response is sent to Front.**\
      _Content of Response:_

   ```JSON
   [
     {
       "id": "",
       "name": "",
       "update time": "",
       "silent": true,
       "items": [
         {
           "id": "",
           "content": "",
           "checked": false
         }
       ]
     }
   ]
   ```

   6. Displayed in Front
      ***
   7. Chat Service read event from Kafka\
      &nbsp;&nbsp; ↓
   8. Chat Service fetches amount of unread messages for every list, and last message\
      &nbsp;&nbsp; ↓
   9. **Second response is sent to the front.**\
      _Content of Response:_

   ```JSON
   [
     {
       "id": "",
       "unread_messages": 0,
       "last_message": ""
     }
   ]
   ```

   10. Front display new data.\
       &nbsp;&nbsp; ↓
   11. SSE Closes.\
        _Later updates are based on WS activities._

&nbsp;

2. **Get List by ID**

   1. Front sends `GET` request\
      &nbsp;&nbsp; ↓
   2. List is fetched from Redis based on ID.\
      &nbsp;&nbsp; ↓
   3. List returns to Front, with all metadata and items metadata.\
      &nbsp;&nbsp; ↓
   4. Front displays list

&nbsp;

3. **Update List / Item**

   1. Front sends `POST` request\
      &nbsp;&nbsp; ↓
   2. List / Item is updated on Redis.\
      **For items** - when version number is lower than what's on Redis, new item is created instead of modification of existing item.\
      &nbsp;&nbsp; ↓
   3. WebSocket Service fetches the event from Kafka\
      &nbsp;&nbsp; ↓
   4. Update is broadcasted to all room members\
      &nbsp;&nbsp; ↓
   5. Front displays update.
      > [!Note]
      >
      > For the user who made the update, it displayed before being broadcasted. If there is a problem, silent retry and backup in localhost.

&nbsp;

4. **Share List**

   > [!Note] SMS not part of MVP
   > Apart from mentioning the SMS Service, the current flow does not include the SMS path.

   1. Front sends `POST` request with all userIDs and unrecognized phone numbers\
      &nbsp;&nbsp; ↓
   2. Lists Service adds members to list\
      &nbsp;&nbsp; ↓
   3. Lists Service publishes event to Kafka, consumed by:

      - Chat Service
      - SMS Service - _planned_

      &nbsp;&nbsp; ↓

   4. Chat Service enters users to equivalent chat\
      &nbsp;&nbsp; ↓
   5. WebSocket Service adds users to room\
      &nbsp;&nbsp; ↓
   6. Lists Service publish event to Kafka, consumed by WebSocket Service with following data:

   ```JSON
    {
      "user_id": [""],
      "list": {
        "id": "",
        "name": "",
        "update time": "",
        "silent": false,
        "items": [
          {
            "id": "",
            "content": "",
            "checked": false
          }
        ]
      }
    }
   ```

   7. WebSocket Service consumes event from Kafka, sends message to all users under `user_id`\
      &nbsp;&nbsp; ↓
   8. Front displays new list

&nbsp;

5. **Remove List**
   1. Front sends `DELETE` request\
      &nbsp;&nbsp; ↓
   2. Lists Service

---

### Chat Service

> [!Note]
> Some of the flows are scattered across different services, as they are part of bigger flows.

---

### DB Writer Service

All Services are writing and reading from Redis. Only the DB Writer Service writes the DB, and when Redis is re-instantiate, this service puts relevant data in Redis.

The DB Writer Service updates the DB every 30sec.

> [!NOTE]
> This service may not be part of MVP
