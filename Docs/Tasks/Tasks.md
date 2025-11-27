Tasks:

1. Message Processor Overload
   **Issue:** The Message Processor handles too many responsibilities (NLP, AI calls, scoring, blacklist management, Redis updates, DB updates)
   **Suggestion:** Split into:
   - Message Analyzer Service (NLP + AI product detection)
   - Scoring Service (handles voting on suggestions)
   - Keyword Manager Service (manages blacklist/product list)

   $\color{green} Done!$

2. Add kafka. Maybe replace Rabbit with it.\
   Every new process should be stored there, and every event that is related to one of the process will be stored next to it. When process is DONE, remove it. No need to remember everything forever. Complete log should be stored later in DataDog.

   $\color{green} Done!$

3. Add DataDog to store logs, which are the same events from kafka but that are not deleted until next month or so.

   $\color{orange} WIP$

4. Go through the conversation with claude, and write down all of the tasks listed there.\
   Look through message no. 2, and no. 3.
   https://claude.ai/share/b840b907-9d24-42f1-8d36-b0fcde9a1bda

   $\color{red} TODO - Make  -  sure  -  you  -  did  -  it!$

5. Add a new service - keyword manager. It will merge the products list with the preferences of everybody.

   $\color{green} Done!$

6. **12\. Statistics Service (Future)**\
   Good foresight to plan for it\
   **Suggestion:** Use event streaming (Kafka?) to feed it historical data\
   **Consider:** Read replicas for the DBs so analytics don't impact production

   $\color{red} TODO$

7. Document Flows

   $\color{orange} WIP$

8. Add a sign up page in Figma

   $\color{red} TODO$

9. Read all of the conversation [here](https://chatgpt.com/share/69278556-74c0-8006-9154-70dc7853256e) and pull tasks from the conversation.
   and [here](https://chatgpt.com/c/692898d9-eedc-8327-af59-00897a56b42d)
