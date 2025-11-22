# CoList - Flows

## Overview

CoList is a collaborative platform that allows users to manage and share shopping lists.\
CoList also offers a B2B service, providing statistics and insights based on real user data

To serve both individual and business needs, CoList is organized into two segments:

1. **B2C** - The consumer-facing part of CoList.
2. **B2B** - Provides analytics and insights derived from user activity.

Respectively, the flows below are organized into those categories too.

&nbsp;

## Notes

Before continue, make sure you read the following notes:

> [!NOTE]
> **All events are logged in Kafka**
>
> Each event has `type`, `data` and `state` to them.\
> When failing service is reinitiated, it will look over all of the `"state": "started"` events, and retry them.
>
> For brevity, kafka logs are omitted from flow diagrams unless it plays a special role beyond standard logging.

> [!NOTE]
> **From Redis to DB**
>
> All writes on Redis are followed with notifications in Kafka for the DB Writer Service to push update
>
> For brevity, update DB logs are omitted from flow diagram unless it plays a special role beyond standard role.\
> See [DB Writer Service](#db-writer-service) for implementation details.

> [!NOTE]
>
> By default, `GET` requests return the full data.\
> When tailored response is required, the flow defines a response template.

> [!NOTE]
> For best visibility, open this MD file through Visual Studio Code, using the [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)

&nbsp;

## Architecture

![ShopList Architecture v1.1](../../Architecture/Current/ShopList-Architecture-v1.1.drawio.svg)

&nbsp;

## 1. B2C Flows

This part is organized into 3 categories:

1. **Base App** - consisting of `Auth Service`, `Users Service`, `Lists Service`, `Chat Service`, `DB Writer Service`
2. **Message Process** - `Message Analyzer Service`, `Product History Manager Service`, `Scoring Service`
3. **AI Suggestions** - `AI Suggestions Service`, `Keyword Manager Service`

> [!NOTE]
> **All REST request must have JWT**
>
> All REST API requests pass through Ngnix, which sends them to the Auth Service for JWT token validation.\
> Valid requests are modified to have `user_id` instead of JWT before forwarding to downstream services.
>
> For brevity, Ngnix is omitted from flow diagrams unless it plays a service-specific role beyond standard authentication.
>
> See [Auth Service: JWT to `user_id` Flow](#jwt-to-userid) for implementation details.

&nbsp;

## 1.1. Base App

### Auth Service

Manages Login and JWTs

1. **Login**
   1. Front redirect to Auth0
      \
      &nbsp;&nbsp; ↓
   2. User Login
      \
      &nbsp;&nbsp; ↓
   3. JWT is sent to the Auth Service
      \
      &nbsp;&nbsp; ↓
   4. _#FIGURE OUT WHATS NEXT_

&nbsp;

2. **JWT to `user_id`**<span id="jwt-to-userid"></span>
   1. Front sends `REST` request
      \
      &nbsp;&nbsp; ↓
   2. Ngnix extracts JWT
      \
      &nbsp;&nbsp; ↓
   3. Auth Service validates JWT, returns `user_id`
      \
      &nbsp;&nbsp; ↓
   4. Ngnix adds `user_id` to headers
      \
      &nbsp;&nbsp; ↓
   5. Destination service receives request

&nbsp;

### Users Service

Manages users, user data, user settings

> [!NOTE]
> This service will be divided into Users Manager and Settings Service, when more settings will be added.

> [!NOTE]
> As Auth Service is yet to be realized, flows might change in the future.

1. **Get Users / Setting**
   1. Front sends a `GET` request
      \
      &nbsp;&nbsp; ↓
   2. Users Service fetches users details from Redis.
      \
      &nbsp;&nbsp; ↓
   3. If type of detail is stored in Auth0, Auth Service fetches them.
      \
      &nbsp;&nbsp; ↓
   4. Front receives users details in a list

&nbsp;

2. **Get User by `user_id`**
   1. Front sends a `GET` request
      \
      &nbsp;&nbsp; ↓
   2. Users Service fetches user details from Redis.
      \
      &nbsp;&nbsp; ↓
   3. Front receives users details in an object

&nbsp;

3. **Update user / users / settings / setting**
   1. Front sends a `PUT` request
      \
      &nbsp;&nbsp; ↓
   2. Users Service update user details in Redis.
      \
      &nbsp;&nbsp; ↓
   3. DB Writer Service updates details in DB (not affecting flow)
      \
      &nbsp;&nbsp; ↓
   4. Auth Service updates details in Auth0
      \
      &nbsp;&nbsp; ↓
   5. Front receives users details in an object

&nbsp;

4. **Send connection status through WebSocket Service**

   > [!NOTE]
   >
   > This is part 1 of the combined flow for getting user status
   1. When user enters the app, the Front receives all user details (see flow above).
      \
      &nbsp;&nbsp; ↓
   2. If allowed, and when WS connection is open, Front sends a WS Message that the user's connected
      \
      &nbsp;&nbsp; ↓
   3. WebSocket Service broadcast the message through all of the rooms the user is part of
      \
      &nbsp;&nbsp; ↓
   4. Front receives message, and ignores it unless the user is currently displayed in the chat settings

&nbsp;

5. **Get User connection status**

   This flow is part of a bigger flow of getting list settings.\
   See [Lists Service flows](#list-settings) for details

&nbsp;

### Lists Service

1. **Get Lists** - _SSE_

   This flow is used in the Home Page and focused on minimal metadata and preview-display specific data
   1. Front sends `GET` request
      \
      &nbsp;&nbsp; ↓
   2. Lists are fetched from Redis.
      \
      &nbsp;&nbsp; ↓
   3. For every list, items are fetched from Redis - 5 at most.
      \
      &nbsp;&nbsp; ↓
   4. Lists Service publishes event to Kafka, consumed by Chat Service
      \
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
   7. Chat Service read event from Kafka
      \
      &nbsp;&nbsp; ↓
   8. Chat Service fetches amount of unread messages for every list, and last message
      \
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

   10. Front display new data.
       \
      &nbsp;&nbsp; ↓
   11. SSE Closes.\
       _Later updates are based on WS activities._

&nbsp;

2. **Get List by ID**
   1. Front sends `GET` request
      \
      &nbsp;&nbsp; ↓
   2. List is fetched from Redis based on ID.
      \
      &nbsp;&nbsp; ↓
   3. List returns to Front, with all metadata and items metadata.
      \
      &nbsp;&nbsp; ↓
   4. Front displays list

&nbsp;

3. **Update List / Item / List Setting**
   1. Front sends `POST` request
      \
      &nbsp;&nbsp; ↓
   2. List / Item is updated on Redis.\
      **For items** - when version number is lower than what's on Redis, new item is created instead of modification of existing item.
      \
      &nbsp;&nbsp; ↓
   3. WebSocket Service fetches the event from Kafka
      \
      &nbsp;&nbsp; ↓
   4. Update is broadcasted to all room members
      \
      &nbsp;&nbsp; ↓
   5. Front displays update.
      > [!NOTE]
      >
      > For the user who made the update, it displayed before being broadcasted. If there is a problem, silent retry and backup in localhost.

&nbsp;

4. **Share List**

   > [!NOTE]
   > **SMS not part of MVP**
   >
   > Apart from mentioning the SMS Service, the current flow does not include the SMS path.
   1. Front sends `POST` request with list of `user_id` and unrecognized phone numbers
      \
      &nbsp;&nbsp; ↓
   2. Lists Service adds members to list
      \
      &nbsp;&nbsp; ↓
   3. Lists Service publishes event to Kafka, consumed by:
      - Chat Service
      - SMS Service - _planned_

      &nbsp;&nbsp; ↓

   4. Chat Service enters users to equivalent chat
      \
      &nbsp;&nbsp; ↓
   5. WebSocket Service adds users to room
      \
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

   7. WebSocket Service consumes event from Kafka, sends message to all users under `user_id`
      \
      &nbsp;&nbsp; ↓
   8. Front displays new list

&nbsp;

5. **Remove List**
   1. Front sends `DELETE` request
      \
      &nbsp;&nbsp; ↓
   2. Lists Service removes list and items from Redis
      \
      &nbsp;&nbsp; ↓
   3. Chat Service removes chat and all of the message from Redis
      \
      &nbsp;&nbsp; ↓
   4. DB Writer Service updates DB accordingly (not affecting flow).
      \
      &nbsp;&nbsp; ↓
   5. WebSocket Service removes all of the users from the equivalent room

&nbsp;

6. **Close List**

   This is different from removing the list. It leaves the list as it was.
   1. Front sends `PUT` request
      \
      &nbsp;&nbsp; ↓
   2. Lists Service changes the `state` in the `list_id` to `closed`
      \
      &nbsp;&nbsp; ↓
   3. WebSocket Service keeps the users in the room, but it stops accepting any WS Message that is not "list is not open anymore"

&nbsp;

7. **Get list settings**<span id="list-settings"></span>
   1. Front sends a `GET` request with list of users with undefined connection status
      \
      &nbsp;&nbsp; ↓
   2. Lists Service fetches data from Redis
      \
      &nbsp;&nbsp; ↓
   3. Users Service fetches connection status for each user in the list settings
      \
      &nbsp;&nbsp; ↓
   4. Lists Service send response to front
      \
      &nbsp;&nbsp; ↓
   5. Front displays data

&nbsp;

### Chat Service

> [!NOTE]
> Some of the flows are scattered across different services, as they are part of bigger flows.

&nbsp;

### DB Writer Service

All Services are writing and reading from Redis. Only the DB Writer Service writes the DB, and when Redis is re-instantiate, this service puts relevant data in Redis.

The DB Writer Service updates the DB every 30sec.\
For that reason, this is an async operation.

> [!NOTE]
> This service may not be part of MVP
