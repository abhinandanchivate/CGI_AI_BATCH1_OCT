## Principle 5 — Next Token Prediction in Generative AI

Next Token Prediction means GenAI predicts the most suitable next token based on the tokens that came before it.

It is one of the fundamental principles behind how Large Language Models (LLMs) generate text.

### 1. Simple non-technical example

Imagine you are teaching a child to complete sentences.

You say:

> "Rahul drinks a cup of \_\_\_\_\_\_."

The child thinks of possible answers:

- Tea
- Coffee
- Milk
- Water
- Rice

The child knows from experience that tea, coffee, and milk are more likely to complete this sentence than rice.

GenAI follows a somewhat similar process, but instead of thinking like a human, it uses mathematical patterns learned during training.

### 2. How GenAI predicts the next token

Let's use your banking example.

Input: "The customer visited the bank to deposit"

Step 1 — Read the existing tokens

The

customer

visited

the

bank

to

deposit

?

Step 2 — Calculate next-token probabilities

money

62%

cash

19%

funds

10%

a

6%

other

3%

Illustrative probabilities created for teaching. These are not actual predictions from a particular GenAI model.

Step 3 — Select a token

Selected next token: money

"The customer visited the bank to deposit money"

The model may select the highest-probability token or sample another possible token, depending on its generation settings.

### 3. Interactive example — Predict the next token

Choose a sentence

Bank deposit

Drinking tea

Loan approval

The customer visited the bank to deposit \_\_\_\_\_\_

Which next token is most likely?

money

cash

documents

flowers

### 4. How GenAI generates a complete sentence

GenAI doesn't normally predict a whole sentence at once. It predicts one token, adds that token to its context, and then predicts the next.

1. Input: The customer

   Predict next token → visited
2. Updated: The customer visited

   Predict next token → the
3. Updated: The customer visited the

   Predict next token → bank
4. Updated: The customer visited the bank

   Predict next token → to
5. Updated: The customer visited the bank to

   Predict next token → deposit
6. Updated: The customer visited the bank to deposit

   Predict next token → money

Generated sentence: The customer visited the bank to deposit money.

This is a simplified illustration. In real LLMs, tokens are not necessarily complete words. A word can be represented by one token or several tokens, and punctuation and spaces may also be included in tokens.

### 5. Attention vs Next Token Prediction

| Attention                                     | Next Token Prediction                         |
| --------------------------------------------- | --------------------------------------------- |
| Helps the model use relevant context          | Predicts what token should come next          |
| Connects relationships between tokens         | Assigns probabilities to possible next tokens |
| Example: Bank is related to deposit and money | Example: After "deposit", predict "money"     |
| Helps construct context-aware representations | Uses those representations to generate text   |

How they work together:

Input

The customer visited the bank to deposit

Attention mechanism

Uses relationships between customer, bank and deposit

Model processing and prediction

Calculates scores and probabilities for possible next tokens

Next token: money

### 6. How does the model learn this during pre-training?

During pre-training, it processes large amounts of tokenized text and repeatedly practices prediction.

For example:

Training text: "The customer deposited money."

The model is given the earlier tokens and tries to predict the next one.

1. Prediction: The model predicts "cash".
2. Actual target: The training text contains "money".
3. Loss calculation: The training system calculates how much probability the model assigned to the actual next token.
4. Backpropagation: The training algorithm calculates gradients and updates model weights to improve future predictions.

After repeating this across massive amounts of training text, the model learns patterns involving language, facts, grammar, relationships and context.

### Trainer-ready explanation

> "GenAI works somewhat like the predictive text feature on your mobile keyboard, but it is far more sophisticated. When I type 'The customer visited the bank to deposit', the model examines the available context and calculates which token is likely to come next. It may select 'money'. It then repeats this prediction process until the response is complete."

Key takeaway: GenAI generates text by repeatedly predicting the next token using learned patterns and the available context. The process can produce remarkably coherent answers, but predicting plausible text does not guarantee factual correctness.
