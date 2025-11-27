### _Users Service_

1. **Get Users / Setting** - yes, users-service may need data from auth0 (through auth-service...? maybe overkill) to complete the get request. If it is overkill, then no.

2. **Get User by `user_id`** - no. single service is involved.

3. **Update user / users** - yes, auth-service should tell auth0 what to update (or just the user-service). If it is overkill, then no.

4. **Update setting** - no. will broadcast updated settings to all of the services, but it's done at the background.

5. **Send connection status through _WebSocket Service_** - no. single service is involved.

6. **Get User connection status** - depends on the get list settings flow (see below, at list service, flow no. 9)

7. **Remove User** - yes...? multiple services are involved, but only when they all finished can the delete request be resolved.

8. **Logout** - maybe? depends on whatever's going on with auth0 at the backend (if there is need for something here)

### _Lists Service_

1. **Get Lists** - _SSE_ - no. it's literally an SSE.

2. **Get List by ID** - no. single service is involved.

3. **Create new list** - yes, depends on chat finishing his part before the list is opened... or maybe not?

4. **Update List / Item / List Setting** - just like above.

5. **Share List** - no. literally broadcasting the message to more then one consumer.

6. **Leave List** - yes. depends on chat (but if I can take the user of the chat room after I said everything is alright, then no)

7. **Remove List** - same as the above.

8. **Close List** - same as the above...

9. **Get list settings** - absolutely YES.

### _Chat Service_

1. **Send message** this is such a complicated flow, that I've made an SVG to make the diagram and it still a bit rough.
   But, I think that yes...? look below:

   1. _Front_ sends message through WS Connection
      \
      &nbsp;&nbsp; ↓
   2. WS Connection provides the message to the _Chat Service_, while broadcasting it to all of the room members
      \
      &nbsp;&nbsp; ↓
   3. _Chat Service_ writes all of the messages in _Redis_
      \
      &nbsp;&nbsp; ↓
   4. The _Front_ of all of the listeners receives the message, and ping the _WebSocket Service_
      \
      &nbsp;&nbsp; ↓
   5. The _WebSocket Service_ provides the ping messages to the _Chat Service_
      \
      &nbsp;&nbsp; ...
   6. For every user who sees the message (enters viewable area in _Front_), _Front_ sends `seen` WS Message
      \
      &nbsp;&nbsp; ↓
   7. The _WebSocket Service_ provides the `seen` messages to _Chat Service_
      \
      &nbsp;&nbsp; ...
   8. When _Chat Service_ sees that all of the listeners received the message,\
      it sends the sender a `received by all` WS Message.
      \
      &nbsp;&nbsp; ...
   9. When _Chat Service_ sees that all of the listeners saw the message,\
      it sends the sender a `saw by all` WS Message.

2. **Edit message** - no, the ws service is telling the chat service what to edit but it's being broadcasted while the chat is saving the message, to save time.

### _DB Writer Service_ - reads from kafka anyway...
