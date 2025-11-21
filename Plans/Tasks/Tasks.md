Tasks:

1. Message Processor Overload
   **Issue:** The Message Processor handles too many responsibilities (NLP, AI calls, scoring, blacklist management, Redis updates, DB updates)
   **Suggestion:** Split into:

   - Message Analyzer Service (NLP + AI product detection)
   - Scoring Service (handles voting on suggestions)
   - Keyword Manager Service (manages blacklist/product list)

2. Add kafka. Maybe replace Rabbit with it.\
   Every new process should be stored there, and every event that is related to one of the process will be stored next to it. When process is DONE, remove it. No need to remember everything forever. Complete log should be stored later in DataDog.

3. Add DataDog to store logs, which are the same events from kafka but that are not deleted until next month or so.

4. Change the Product List DB to a redis persistence.

5. Go through the conversation with claude, and write down all of the tasks listed there.\
   Look through message no. 2, and no. 3.
   https://claude.ai/share/b840b907-9d24-42f1-8d36-b0fcde9a1bda

6. Add a new service - orchestrator. It will merge the products list with the preferences of everybody.

7. **12\. Statistics Service (Future)**\
   Good foresight to plan for it\
   **Suggestion:** Use event streaming (Kafka?) to feed it historical data\
   **Consider:** Read replicas for the DBs so analytics don't impact production

8. Document Flows
