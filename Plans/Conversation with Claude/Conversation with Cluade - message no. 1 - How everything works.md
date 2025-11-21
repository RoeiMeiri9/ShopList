<img src="../../Architecture/Old/V1/ShopList Architecture v1.drawio.png"/>

Take a look over this graph. It is the current architecture of ShopList.

As you can see, there are many arrows and services. I'll list some of the more abscure once to make sure you understood what's going on. The rest you can propably guess.

But first, a few words about shoplist.

### ShopList Overview

1.  Its an app that allows users to create, share and edit shop lists.

2.  Multiple users can collaborate on a single list. They can add, remove and check rows.

3.  When adding new rows to an existing list, or when adding a title to the list, based on current data, AI powered suggestions will be displayed on the list for the user to quickly add them.

4.  The users have a built in chat interface where they can discuss what to put on the list.

5.  When a user says he will add bears and steaks, he'll be able to do so from the chat.

6.  Later, will be added a statistics service that will give some statistics about - what will be bought tommorrow morning. (B2B feature).

Note: pay close attention to the colors of the arrows.

### Flows

1. **Front-users** - used for auth (there will be additional Auth0 path when I'll understand it), for additional knowledge about me and other users, settings.

2. **Front-chat** - used for chat.

3. **Font-Lists** - used to get all lists, list items, manipulating list, share list (there might be additional work about this when I'll work on the share).
   not included here - an arrow from the list to the chat, to notify it that such chat should be opened when new list is created. (maybe using rabbit?)

4. **Front-WS** - used as the main point through which all of the WS messages will come from/to.

5. **Users-WS** - Only to display which user is active.

6. **Chat-Rabbit-Message Processor** - When new message arrives, the message should be processed to show if the message includes some key-words (products were mantioned).

7. **Message Processor-Redis-AI-ProductListDB** - The message is processed by dropping words from the black-list (stored in Redis and the DB), scans for good words, asks AI what from the rest of the words are products, stores new keywords in DB/Cache, and sends all of the relevant keywords to the WS in a json.

8. **Chat-RabbitMQ-Message Processor** - When an item that was suggested to the user in the chat (after the processing above was done) if it is being used, store it in the rabbit, and the message processor turns up the score of it. If it's dismissed, it turns it down. When a score belongs to a word reaches to -5, its being remove from product-list and being stored in the black list.

9. **Message Processor-Product List** - each X time, push changes to DB. (Maybe there should be a different service just for doing that?)

10. **Lists-RabbitMQ-AI Suggestion** - When an item that was suggested by an AI was modified, it's being sent to the Rabbit, then the AI Suggestion service will modify the suggestion on the DB. If other users modified an item I added from an AI Generated Suggestion, it will not be modified.

11. All of the DBs are using the Users DB as the reference for the users.

12. Chat and List are connected, as every list has it's chat.
