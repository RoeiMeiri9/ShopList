# ShopList - Flows

## Overview

ShopList allows customers to collaborate on writing and managing Shop Lists.\
As a B2B service, ShopList provides real-data based statistics. (Note - ask Eden to write something here....)

For that reason, ShopList is divided into two parts:

1. B2C - The known part of ShopList.
2. B2B - Displaying statistics, based on users activities.

> [!NOTE]
> All events are logged in Kafka. Each event has `type`, `data` and `state` to them.\
> When failing service is reinitiated, it will look over all of the `"state": "started"` events, and retry them.
>
> For brevity, kafka logs are omitted from flow diagrams unless it plays a special role beyond standard logging.

## 1. B2C Flows

This part is divided into 3 categories:

1. **Base App** - Auth, Users, Lists, Chat, DB Writer
2. **Message Process** - Message Analyzer, Keyword Manager, Scoring
3. **AI Suggestions** - AI Suggestion, Keyword Manager

> [!NOTE]
> All REST API requests pass through Nginx, which validates the JWT token and replaces it with the authenticated `user_id` before forwarding to downstream services.
>
> For brevity, Nginx is omitted from flow diagrams unless it plays a service-specific role beyond standard authentication.
>
> See [Authentication: JWT to `user_id` Flow](#authentication) for implementation details.

## 1.1. Base App

### Authentication

Manages Login and JWTs

1. **Login**\
   Front redirect to Auth0\
    &nbsp;&nbsp; ↓\
   User Login\
    &nbsp;&nbsp; ↓\
   JWT is sent to the Auth\
    &nbsp;&nbsp; ↓\
    #FIGURE OUT WHATS NEXT

2. **JWT to `user_id`**\
   Front sends REST request\
    &nbsp;&nbsp; ↓\
   Nginx extracts JWT\
    &nbsp;&nbsp; ↓\
   Auth Service validates JWT, returns `user_id`\
    &nbsp;&nbsp; ↓\
   Nginx adds `user_id` to headers\
    &nbsp;&nbsp; ↓\
   Destination Service receives request

### Users

Manages users, user data, user settings

> [!NOTE]
> This service will be divided into Users Manager and Settings Service, when more settings will be added.

### Lists

1. **Get Lists** - SSE\
   Front sends `GET` request\
    &nbsp;&nbsp; ↓\
   Lists are fetched from Redis.\
    &nbsp;&nbsp; ↓\
   For every list, items are fetched from Redis - 5 at most.\
    &nbsp;&nbsp; ↓\
   Lists put on Kafka a request to send the client how many messages were missed for every list\
    &nbsp;&nbsp; ↓\
   **First response is sent to front.**\
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

   &nbsp;&nbsp; ↓\
   Displayed in Front\
    &nbsp;&nbsp; ... ... ...\
   Chat fetches amount of unread messages for every list, and last message\
    &nbsp;&nbsp; ↓\
   **Second response is sent to the front.**\
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

   &nbsp;&nbsp; ↓\
   Front display new data.\
   &nbsp;&nbsp; ↓\
   SSE Closes.\
   Later updates are based on WS activities.

2. **Get List by ID**\
   Front sends `GET` request\
    &nbsp;&nbsp; ↓\
   List is fetched from Redis based on ID.\
    &nbsp;&nbsp; ↓\
   List returns to Front, with all metadata and items metadata.\
    &nbsp;&nbsp; ↓\
   Front displays list

3. **Update List / Item**\
   Front sends `POST` request\
    &nbsp;&nbsp; ↓\
   List / Item is updated on Redis
   &nbsp;&nbsp; ↓\
    Kafka

### Chat

### DB Writer

All Services are writing and reading from Redis. Only the DB Writer writes the DB, and when Redis is re-instantiate, this service puts relevant data in Redis.

> [!NOTE]
> This service may not be part of MVP
