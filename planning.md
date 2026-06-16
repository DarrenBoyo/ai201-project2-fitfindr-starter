# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:** Searches the secondhand listings dataset foritems that match the users's request.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): Natural language description of what the user wants.
- `size` (str): Desired size or fit.
- `max_price` (float): Highest price the user is willing to pay.

  
**What it returns:** A list of matching listing dictionaries, including id, title, description, category, style_tags, size, condition, price, colors, brand and platform
<!-- Describe the return value — what fields does a result contain? -->

**What happens if it fails or returns nothing:** The agent explains that no strong matches were found, broadens the search if possible, or asks the user to relax filters such as size, price, color, or style.
<!-- What should the agent do if no listings match? -->

---

### Tool 2: suggest_outfit

**What it does:** Checks how a selected listing fits with the user’s wardrobe and suggests an outfit.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): The listing selected from search_listings.
- `wardrobe` (dict): The user’s wardrobe, containing an items list.

**What it returns:** An outfit recommendation using the new item plus compatible wardrobe pieces, with a short explanation of why the outfit works.
<!-- Describe the return value -->

**What happens if it fails or returns nothing:** The agent gives a general styling suggestion based only on the new item, or explains that it needs wardrobe data to make a personalized outfit.
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->

---

### Tool 3: create_fit_card

**What it does:** Creates a shareable outfit description for the final recommendation.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (dict): The outfit recommendation from suggest_outfit, including the new item, selected wardrobe pieces, styling notes, and reasoning.


**What it returns:** A concise fit card with the item, outfit pieces, styling description, price/value note, and recommendation.
<!-- Describe the return value -->

**What happens if it fails or returns nothing:** The agent returns a plain-text outfit summary instead of a formatted fit card.
<!-- What should the agent do if the outfit data is incomplete? -->

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

### Tool 4: evaluate_price

**What it does:**
Evaluates whether a listing seems like a good deal.

**Input parameters:**
listing (dict): The secondhand listing being evaluated.
max_price (float): The user’s budget.

**What it returns:**
A price judgment such as “good deal,” “fair price,” or “overpriced,” with a short reason.

**What happens if it fails or returns nothing:**
The agent skips the price score and only reports the listing’s actual price.


## Planning Loop

**How does your agent decide which tool to call next?** The agent first reads the user’s request and extracts constraints like item type, style, size, color, and budget. It calls search_listings first, then evaluates the best results, passes the strongest item into suggest_outfit with the wardrobe, and finally calls create_fit_card to generate the final user-facing recommendation. If any step fails, the agent either retries with broader constraints, uses a fallback response, or asks the user for missing information.
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->

The agent stores important information in a shared state object as each tool runs. For example, the user’s request is parsed into search constraints, `search_listings` returns matching listings, the best listing is passed into `suggest_outfit` as `new_item`, and the outfit result is passed into `create_fit_card` for the final response.


## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|---|---|---|
| search_listings | No results match the query | Broaden the search by relaxing style, size, or price constraints, then try again. If there are still no results, explain that no matches were found. |
| suggest_outfit | Wardrobe is empty | Create a general outfit suggestion using only the new item instead of personalized wardrobe pieces. |
| create_fit_card | Outfit input is missing or incomplete | Return a plain-text recommendation instead of a formatted fit card. |
---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

```
+------------------+
|    User Query    |
+------------------+
          |
          | description, size, budget, style preferences
          v
+------------------+
|   Planning Loop  |
+------------------+
          |
          | stores user constraints
          v
+------------------------------+
| Session State                |
|------------------------------|
| query                        |
| selected_item                |
| outfit_suggestion            |
| fit_card                     |
+------------------------------+
          |
          v
+----------------------------------------------+
| search_listings(description, size, max_price)|
+----------------------------------------------+
          |
          | results=[]
          +------------------------------+
          |                              |
          v                              |
 [ERROR] No listings found               |
          |                              |
          v                              |
 Return fallback response <--------------+
          |
          |
          | results=[item1, item2, ...]
          v
+------------------------------+
| Session State Update         |
| selected_item = results[0]   |
+------------------------------+
          |
          v
+--------------------------------------+
| evaluate_price(selected_item)        |
+--------------------------------------+
          |
          | price evaluation
          v
+------------------------------+
| Session State Update         |
| value_score = "good deal"    |
+------------------------------+
          |
          v
+--------------------------------------+
| suggest_outfit(selected_item,        |
| wardrobe)                            |
+--------------------------------------+
          |
          | wardrobe empty?
          +------------------------------+
          |                              |
          v                              |
 [WARNING] Empty wardrobe               |
          |                              |
          v                              |
 Generate generic outfit <--------------+
          |
          |
          v
+------------------------------+
| Session State Update         |
| outfit_suggestion            |
+------------------------------+
          |
          v
+--------------------------------------+
| create_fit_card(outfit_suggestion)   |
+--------------------------------------+
          |
          | missing outfit?
          +------------------------------+
          |                              |
          v                              |
 [WARNING] Incomplete outfit            |
          |                              |
          v                              |
 Return plain-text summary <------------+
          |
          |
          v
+------------------------------+
| Session State Update         |
| fit_card                     |
+------------------------------+
          |
          v
+------------------+
| Final Response   |
+------------------+
```

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:** Implement `search_listings`, `suggest_outfit`, and `create_fit_card` separately. Each tool should have clear inputs, outputs, and fallback behavior.

**Milestone 4 — Planning loop and state management:**

Build the agent loop that decides which tool to call next based on the current state. The loop should pass outputs from one tool into the next and handle errors without crashing.

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
The agent extracts constraints from the user request:
- description: vintage graphic tee
- max_price: 30
- style preferences: baggy jeans, chunky sneakers

Then it calls:

`search_listings(description="vintage graphic tee", size="", max_price=30)`

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
`search_listings` returns matching listings, such as a black vintage-style graphic tee for $24.

The agent chooses the best match based on price, style tags, and condition.

**Step 3:**
<!-- Continue until the full interaction is complete -->
The agent calls:

`suggest_outfit(new_item=selected_listing, wardrobe=example_wardrobe)`

The tool matches the tee with wardrobe pieces like baggy jeans and chunky white sneakers.

**Step 4:**
The agent calls:

`create_fit_card(outfit=outfit_result)`

This creates a polished final recommendation.

### Final output to user:
I found a black vintage-style graphic tee for $24. It fits your style because it matches your baggy jeans and chunky sneakers, and it stays under your $30 budget.

Fit card: Pair the tee with baggy dark-wash jeans, chunky white sneakers, and a black crossbody bag for a relaxed streetwear look. I’d recommend this piece because it is affordable, versatile, and easy to style with clothes you already wear.
**Final output to user:**
<!-- What does the user actually see at the end? -->
