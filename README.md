# README

## Building a Chatbot for Mental Health Conversations Using DSPy and Qdrant

This project demonstrates how to build a chatbot for mental health conversations using the DSPy (Deep Structured Prediction) library and the Qdrant vector database.

### Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Dataset](#dataset)
- [Setting Up Qdrant](#setting-up-qdrant)
- [Building the RAG Pipeline](#building-the-rag-pipeline)
- [Implementing the Coprocessor](#implementing-the-coprocessor)
- [Testing the RAG Pipeline](#testing-the-rag-pipeline)
- [Explanation of Key Components](#explanation-of-key-components)
- [Results](#results)
- [References](#references)

### Introduction

The Retrieval-Augmented Generation (RAG) pipeline is a technique that combines a language model with a retrieval system to generate high-quality responses. In this project, we use DSPy to define the architecture and behavior of the RAG pipeline, and Qdrant as the vector database to store and retrieve relevant information.

### Installation

Create a `requirements.txt` file with the following dependencies:

```
qdrant-client[fastembed]
datasets
dspy-ai
transformers
torch
accelerate
```

Install the dependencies using pip:

```bash
pip install -r requirements.txt
```

### Dataset

We use the Mental Health Conversation Dataset from Hugging Face, which contains 172 rows of mental health conversations.

```python
from datasets import load_dataset

dataset = load_dataset('heliosbrahma/mental_health_chatbot_dataset')
df_pandas = dataset['train'].to_pandas()
documents = df_pandas['text'].to_list()
ids = list(range(1, len(documents) + 1))
```

### Setting Up Qdrant

Launch a local instance of the Qdrant vector database and initialize a client object:

```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)
client.delete_collection(collection_name="Mental Health")
client.add(collection_name="Mental Health", documents=documents, ids=ids)
```

### Building the RAG Pipeline

Load the language model and create a retriever model combined with the Qdrant vector database:

```python
import dspy

# Configure language model
llm = dspy.HFModel(model='mistralai/Mistral-7B-Instruct-v0.3')

# Create a Qdrant retriever model
from dspy.retrieve.qdrant_rm import QdrantRM
qdrant_retriever_model = QdrantRM("Mental Health", client, k=10)

# Configure DSPy settings
dspy.settings.configure(lm=llm, rm=qdrant_retriever_model)
```

Define the RAG pipeline using DSPy:

```python
class RAG(dspy.Module):
    def __init__(self, num_passages=3):
        super().__init__()
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate_answer = dspy.ChainOfThought("context, question -> answer")

    def forward(self, question):
        context = self.retrieve(question).passages
        prediction = self.generate_answer(context=context, question=question)
        return dspy.Prediction(context=context, answer=prediction.answer)
```

### Implementing the Coprocessor

Refine the answer generated by the RAG pipeline with a Coprocessor module:

```python
class Coprocessor(dspy.Module):
    def __init__(self):
        super().__init__()
        self.rag = RAG()

    def forward(self, question):
        # Retrieve relevant passages
        context = self.rag.retrieve(question).passages

        # Generate a draft answer using the RAG
        draft_answer = self.rag.generate_answer(context=context, question=question).answer

        # Refine the answer using a Coprocessor
        refined_answer = self.refine_answer(draft_answer, context)

        return dspy.Prediction(context=context, answer=refined_answer)

    def refine_answer(self, draft_answer, context):
        # Implement your custom logic to refine the answer using the draft answer and the retrieved context
        refined_answer = draft_answer + " (Refined by Coprocessor)"
        return refined_answer
```

### Testing the RAG Pipeline

Test the RAG pipeline with the Coprocessor:

```python
# Test the RAG with Coprocessor
coprocessor = Coprocessor()
example_query = "Tell me about the panic attack?"
response = coprocessor(example_query)
print(response.answer)
```

### Explanation of Key Components

- **RAG (Retrieval-Augmented Generation)**: Combines a language model with a retrieval system to generate high-quality responses by integrating relevant information from a knowledge base.
- **DSPy**: A library that provides a high-level and intuitive way to define the architecture and behavior of machine learning models, including RAG pipelines.
- **Qdrant**: A vector database that stores and searches for high-dimensional data, such as the embeddings used in language models.

### Results

Example conversation with the chatbot:

```
Agent: Hello, thank you for contacting MindWell mental health support. My name is Sam, how may I assist you today?
Customer: Tell me about the panic attack?
Agent: I'd be happy to provide information about panic attacks. To better assist you, may I ask if you're inquiring for yourself or someone else, and if you've experienced what you think might be a panic attack?
Customer: I'm asking for myself. I think I might have had one recently, but I'm not sure.
Agent: Thank you for sharing that. A panic attack is an intense surge of fear or extreme discomfort that reaches a peak within minutes. Can you tell me about some of the symptoms you experienced during your recent episode?
Customer: My heart was racing, I felt short of breath, and I was really dizzy. It came on suddenly and was pretty scary.
Agent: I see. Those symptoms are indeed common during panic attacks. Rapid heartbeat, shortness of breath, and dizziness are typical, along with other possible symptoms like sweating, trembling, and a feeling of losing control. How long did these sensations last?
Customer: It felt like forever, but it was probably about 10 minutes.
Agent: That timing is consistent with panic attacks, which usually peak within 10 minutes. It's important to note that while panic attacks are intense and distressing, they're not physically dangerous. Would you like me to provide some information on coping strategies and treatment options?
Customer: Yes, please. I'd like to know how to handle this if it happens again.
Agent: Certainly. There are several strategies that can help manage panic attacks. These include deep breathing exercises, progressive muscle relaxation, and cognitive techniques to challenge anxious thoughts. Additionally, professional treatments like cognitive-behavioral therapy and, in some cases, medication can be very effective. Would you like me to explain some of these techniques in more detail?
```

### References

- [Stanford DSPy - Qdrant](https://github.com/stanfordnlp/dspy)
- [dspy/dspy/retrieve/qdrant_rm.py at main · stanfordnlp/dspy · GitHub](https://github.com/stanfordnlp/dspy/blob/main/dspy/retrieve/qdrant_rm.py)
- [​​Signature Optimizer | DSPy](https://github.com/stanfordnlp/dspy)

This README provides a comprehensive guide to setting up and running the chatbot for mental health conversations using DSPy and Qdrant. Follow the instructions step by step to reproduce the project and understand the key components involved.
