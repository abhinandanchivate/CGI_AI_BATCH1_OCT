# How GenAI Pre-training Works — From Tokens to Weights

Let's understand the complete pre-training process using one simple example: teaching AI to understand sentences about tea.

## Step 1 — Collect training data

Imagine we give the AI model 1,000 sentences:

- Rahul drinks a cup of tea.
- Priya drinks a cup of coffee.
- My mother prepares hot tea.
- Tea is made using water and milk.
- Rahul likes tea with sugar.

This is called the training dataset.

## Step 2 — Tokenization: Break sentences into tokens

Consider:

"Rahul drinks a cup of tea."

The model converts the sentence into smaller pieces called tokens.

Original sentence

Rahul drinks a cup of tea.

Tokens and IDs

Rahul

ID: 101

drinks

ID: 205

a

ID: 12

cup

ID: 310

of

ID: 42

tea

ID: 508

.

ID: 9

For simplicity, every word is shown as one token. Real tokenizers may split a word into multiple tokens. IDs are illustrative.

Why do we need tokens?

Because the model processes numerical representations, rather than directly performing calculations on written words.

## Step 3 — Convert tokens into embeddings

Each token ID is mapped to a list of numbers called an embedding.

For example:

| Token  | Token ID | Simplified embedding |
| ------ | -------- | -------------------- |
| Rahul  | 101      | [0.2, 0.5, 0.1]      |
| drinks | 205      | [0.6, 0.3, 0.4]      |
| tea    | 508      | [0.8, 0.2, 0.7]      |

Embeddings help the model represent and learn relationships between tokens.

These numbers are illustrative. Real embeddings usually contain many more dimensions, and their values are learned during training.

## Step 4 — The model predicts the next token

The model receives:

"Rahul drinks a cup of \_\_\_\_"

It tries to predict the next token.

Example prediction (hypothetical)

tea

40%

coffee

35%

water

15%

other tokens

10%

For this example, the correct next token in the original sentence is "tea".

## Step 5 — Calculate the error (loss)

The model assigned only 40% probability to the correct token.

To calculate the loss, a typical language model uses cross-entropy:

\\[ \text{Loss}=-\ln(0.40)=0.916 \\]

This indicates how poorly the model predicted the correct token. Lower loss is better.

## Step 6 — Calculate and update weights

Imagine one of the model's weights is initially:

\\[ W=0.30 \\]

The training algorithm calculates a gradient, which tells the model how changing that weight would affect the loss.

Suppose:

- Current weight = 0.30
- Gradient = −0.20
- Learning rate = 0.10

The simplified gradient-descent formula is:

\\[ W\_{\text{new}}=W\_{\text{old}}-(\text{Learning Rate}\times\text{Gradient}) \\]

\\[ W\_{\text{new}}=0.30-(0.10\times-0.20) \\]

\\[ \boxed{W\_{\text{new}}=0.32} \\]

So the weight changes from 0.30 to 0.32.

This weight is a numerical parameter inside the network, not a manually defined importance score for the word "tea". The gradient is calculated using backpropagation.

## Step 7 — Repeat across millions or billions of tokens

The process continues:

<img width="828" height="618" alt="image" src="https://github.com/user-attachments/assets/d40a27a2-0b58-4188-adc5-a70b40bcbd33" />


## Step 8 — Test the pre-trained model

After pre-training, suppose the user enters:

"Rahul drinks a cup of"

The model uses its learned weights to generate a likely continuation, such as:

"tea."

The model has learned patterns from many examples. It can also generate text it has never seen in exactly that form.

## Final summary for participants

| Step                 | What happens                     | Simple meaning                   |
| -------------------- | -------------------------------- | -------------------------------- |
| 1. Data              | Collect sentences                | Give AI material to learn        |
| 2. Tokens            | Split text into pieces           | Break text into units            |
| 3. Token IDs         | Assign numerical IDs             | Identify the pieces              |
| 4. Embeddings        | Convert IDs into vectors         | Represent tokens numerically     |
| 5. Prediction        | Predict next token               | Make a guess                     |
| 6. Loss              | Compare against correct token    | Measure the error                |
| 7. Weights           | Update neural network parameters | Improve future predictions       |
| 8. Repeat            | Train on many batches            | Learn more patterns              |
| 9. Pre-trained model | Save trained parameters          | Ready for further tuning and use |

One-line explanation:

> GenAI pre-training is a repeated cycle of converting text into tokens, predicting the next token, calculating the error, and updating neural network weights to improve future predictions.

Remember this sequence:

Data → Tokens → Token IDs → Embeddings → Transformer → Prediction → Loss → Backpropagation → Weight Update → Repeat.
