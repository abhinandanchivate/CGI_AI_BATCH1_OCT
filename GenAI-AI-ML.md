### Assignment: Three Smart Assistants at a Supermarket

**Scenario:**  
Riya manages a supermarket. She wants to reduce billing queues, avoid running out of products, and create promotional messages.

She introduces three capabilities:

1. A system checks how many customers are waiting. If a queue has more than five customers, it recommends opening another billing counter.
2. A system studies previous sales, weekends, and festival demand to predict how many milk packets the store will need tomorrow.
3. A system creates a WhatsApp promotional message using verified product prices and approved discounts.

**Your task**

**1. Identify the technology**

Complete this table using **rule-based AI, ML, or GenAI**.

| Supermarket requirement | Technology | Why does it fit? |
|---|---|---|
| Recommend opening another counter | | |
| Predict tomorrow’s milk demand | | |
| Write a promotional message | | |

*For this exercise, treat the queue rule as one part of a larger AI decision system. A single fixed rule alone is basic automation.*

**2. Explain the process**

For each requirement, write:

- **Input:** What information does the system need?
- **Process:** Does it follow rules, learn patterns, or generate content?
- **Output:** What does it produce?

**3. Apply it to this situation**

The store has the following information:

- Eight customers are waiting at one billing counter.
- Tomorrow is a festival.
- Milk sales during the last three similar festivals were 180, 200, and 220 packets.
- The approved offer is **10% off biscuits**, valid **tomorrow only**.

Answer these questions:

1. What should the queue system recommend?
2. What can the historical milk sales tell the manager? Why is tomorrow’s exact demand still uncertain?
3. Write a promotional message of no more than 40 words using only the approved offer.
4. What could go wrong if GenAI invents a discount?
5. Who should approve the promotional message before it is sent?

**Few more questions/situations**:
1. Conflicting recommendations during festival demand
The ML system predicts that the supermarket will sell 300 milk packets tomorrow. However, the refrigerator can hold only 220 packets, and the supplier cannot accept returns. The manager also knows that road construction may reduce customer visits.
Question: How should AI, ML, and GenAI work together to help the manager decide how much milk to order?
Explain:
- What additional information is needed before accepting the prediction?
- How should the decision system handle storage limits and possible wastage?
- How can GenAI explain the recommendation without presenting the prediction as a certainty?
- Who should make the final decision, and why?

2. A correct prediction but an incorrect promotional message
The ML system predicts high demand for biscuits. The approved offer is 10% off selected biscuit brands tomorrow only. GenAI creates this message:
“Festival celebration! Get 25% off all biscuits throughout the week. Hurry—stock is almost gone!”

The message sounds attractive, and the marketing team wants to send it immediately.
Question: Identify every unsupported or incorrect claim and design a process that prevents this message from reaching customers.
Explain:
- Does a correct ML prediction make the generated message trustworthy?
- Which facts should come from approved business records?
- Which checks can use fixed rules, and which require human judgement?
- Write a corrected promotional message using only verified information.

  
3. Improving one outcome while harming another
The queue system recommends opening another counter whenever more than five customers are waiting. During a festival rush, it moves two employees from shelf replenishment to billing.
Queues become shorter, but shelves remain empty even though products are available in the storeroom. Sales fall, and customers complain.
Question: Why did a reasonable local decision produce a poor overall business result? Redesign the solution using AI, ML, and GenAI.
Explain:
- What should ML predict besides queue length?
- What staffing constraints and competing goals should the AI decision system consider?
- How could GenAI help communicate employee assignments?
- Which measures would show whether the overall solution is successful?
- What should happen when no staffing arrangement can satisfy every requirement?

