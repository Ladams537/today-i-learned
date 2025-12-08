See how this software is just perfect for demos. Can spin something up in a day or two then get market validation immediately if the idea is worth pursuing further.
LangGraph API url allows you to modify graphs from within your own ide.

## Nodes
* Tools
* Reducers

## Memory
* Checkpointers create threads to parts of the conversation -> appends previous messages. Token expense goes up quickly
* Multiple Schemas

### Filtering and Trimming Messages
* Separating out threads is easy

### Summarising Messages
* LineGraph supports checkpointers with both SQLite (good starting point to understand how it works) and PostGres (for production)
* After certain number of messages, all old messages are 'nuked' and a summary is made

## Human in the Loop
* Breakpoints
* Streaming: updates (updates only -> AI message) and values (full-state)
* **Use Cases:**
  * Approval - allow user to accept an action
  * Debugging - rewinding graph
  * Editing - modify the state
* Shows the daisychaining of some key ideas:
  * Breakpoint at assistant node
  * Thread_id
  * Reducer append
  * LLM sees updated message
  * Goes to tools, back to assistant, breakpoint -> approve
* Dummy nodes to allow for systematic human feedback
* **Dynamic Breakpoints**
* Time-Travel -> this is what is used for debugging

## Building Your Own Assistant
### Parallelisation
* a -> b + c (parallel) -> d
* if two nodes in parallel that write to the same node -> must use reducer to aggregate those results
* parallel branches run as part of the same step. **Will wait until they ALL finish until next step.**
* linegraph decides how to order nodes when they are in the same step
  * can have more control over this by using a custom reducer
  * e.g. sorting reducer managing order as they are aggregated
* **Use Case:** searching multiple sources at the same time

### Sub-Graphs
* Example: system that accepts logs
  * Summarise
  * Find failure modes
* As long as what you want to communicate is a key in both the parent and the child -> they can effectively communicate
* Many keys will not need reducers as they won't be present in the output schema of each subgraph
* **Make traces much more readable**

### Map-Reduce
* **Map:** Break a task into smaller sub-tasks, processing each sub-task in parallel
* **Reduce:** Aggregate the results across all of the completed sub-tasks
* E.g. create a set of jokes for each subject -> choose the best one from the list
  * Automatically generates all the edges based on the size of the list subjects -> saves a lot of manual work
* **Able to decouple state of map node from overall map state**

### Research Assistant
* **Goal**
  * Source Selection
    * User chooses any set of input sources for their research
  * Planning
    * System generates a team of AI analysts, each focused on one sub-topics
    * Human-in-the-loop will refine before research begins
  * LLM Utilisation
    * analyst will conduct in-depth research with an AI-expert using selected sources
    * multi-turn conversations to extract insights as shown in `STORM` paper
    * captured in sub-graphs
  * Research Process
    * experts gather information in parallel
    * all interviews conducted simultaneously with map-reduce
  * Output Format
    * synthesised into final report
    * customisable prompts for the report structures

## Long-Term Memory
1. Within session
  * Chat history
  * Checkpointers
2. Across session -> present in every session with the user
  * Semantic -> Facts; Episodic -> Memories; Procedural -> Instructions (CoALA paper does good work on mapping this)
  * Semantic: profile or list (management vs retrieval issues)
  * Episodic: prior reasoning trajectories
  * Procedural: conversation + old instructions -> LLM -> new instructions
* When do you want to update memory?
  * **Hot-path** -> during runtime (can effect UX / latency)
  * **Background** -> as a separate process (frequency of memory writing needs to be tuned)
* **Put** -> save objects to store; **Search** -> retrieve objects; **Get** -> retrieve by namespace and key
* Can pass **Store** as a parameter into your functions



  
