+++
title = "RAG Explained"
date = "2025-02-23T12:34:51+01:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "DarkBones"
authorTwitter = "" #do not include @
cover = ""
tags = []
keywords = []
description = ""
showFullContent = false
readingTime = true
hideComments = true
draft = true
+++
[brief executive summary about rag]

## RAG 

Large Language Models are great for many things. They can code, write emails, hallucinate ingredients for the perfect sandwich you asked it to compose, and even write articles (though I still prefer doing that myself). But for some things, they just don't work. Because LLMs take a long time to train, they don't "know" about recent events. If you ask an LLM about what happened last week, it will either tell you it can't, tell you what happened the week before its training started, or just make stuff up.

Some LLMs get around this by pulling in real-time data from external sources before generating a response. This approach is called RAG—short for Retrieval-Augmented Generation—where the model retrieves relevant information and augments your prompt before answering.

## RAG - Oversimplified

But how does RAG work under the hood? Instead of researching on our own, let's ask our favorite LLM:
[comic where LLM says rag is a piece of cloth]

Okay, so clearly this model doesn't know what Retrieval-Augmented Generation is. That's okay. We can ask our friend Bob:
[comic where user asks Bob]

Interesting, Bob also didn't know the answer but was able to retrieve it. Let's break down what happened:
1. We asked Bob to tell us about RAG.
2. Bob went to the library and asked the librarian for information about RAG.
3. The librarian told Bob where he can find it.
4. Bob retrieves the information as per the instructions of the librarian.
5. Now Bob sounds like he knew it all along. Thanks, Bob.

When we break it down like this, it sure looks like Bob is doing the work of a RAG agent. So, let's think about how a RAG agent works.

## RAG - Simplified

Let's transform our interaction with Bob into an actual RAG system:
- Bob becomes the RAG System.
- The librarian becomes an Embedder.
- The library becomes a Vector Database.

[comic where user asks RAG agent]

Instead of the user prompting the LLM directly, they prompt the RAG system instead. The RAG system goes to the embedder, and repeats the user's prompt and gets a vector in return. This "vector" is a numeric representation of the prompt. The idea is that data that shares similarities with the prompt will share similarities in their vector representations.

Basically, we can use this vector to find relevant data in the vector database. We give the vector of the user's prompt to the database, and it returns the most "similar" (relevant) data.

The RAG system then simply "augments" the user's prompt with this relevant information.

And that's it in a nutshell. Retrieve, Augment, and then Generate. RAG.

But of course we cannot retrieve data from a database if that data doesn't exist. So how do we add it? It's very straight-forward, actually. We start out with the same process, but instead of using the vector to find relevant information, we store the information along with its vector representation.

[comic of inserting into the knowledge base]

<!-- TODO: Rethink this part -->
If you're here just for the bigger picture, congratulations. Now you have it. If you're a fellow neckbeard, let's talk a bit more about vectors and embedders. Fair warning: strong language ahead. Words like "cosine similarity" and concepts such as "high-dimensional spaces".

## What is a Vector?

In the simplest term, a vector is a set of coordinates that tell you how to move from A to B. Look at this graph.

[nickelback look at this graph]

This graph has two dimensions. This means that each point; A, B, C, and D can be described using a two-digit coordinate system. The first of the two digits tell us how far from the origin (0) to move to the right, and the second digit tells us how far to move up. So to get to A, our vector is `[3, 7]`, and to get to D, our vector is `[3, 0]`.

This principle holds up just as well in 3 dimensions. In order to get from your desk to your coffee machine for a refill, you have to go from your current location a certain distance in the x, y, and z axes, making it a 3-digit coordinate system.

We can't really think about or understand beyond 3 dimensions, but computers don't have this limitation. The math works just the same way. 4 dimensions? No problem, we'll use a 4-digit coordinate system. 100 dimensions? 100-digit coordinate system.

The embedder I use works with a coordinate system that has 768 digits. When you're done trying to visualize what 768-dimensional space looks like, let's go back to our lovely, easy-to-draw, 2d graphs.

### Why Are Vectors Useful?

By themselves, vectors are just n-dimensional coordinates that point to a point in n-dimensional space. But what makes them useful is the things they point to. You probably don't think about GPS coordinates on a daily basis. But as long as you're on earth, your exact position can be described with these coordinates.

In the same way, vectors are coordinates not to places, but to information. A specialized LLM, an embedder, is trained on a large corpus of text to figure out similarities and to place these pieces of information somewhere in n-dimensional space such that similar topics tend to be grouped together. Like, when you go to a social event, you're likely to stick with your friends, colleagues, or at least a group of like-minded people.

[graph of related words]

This graph shows how words that are similar in meaning tend to get grouped together in this n-dimensional space. Modern embedders (like **BERT**) don't use single-word embeddings anymore, but generate *contextual embeddings*.

This clustering of similar concepts in vector space is what makes embeddings so powerful. But early embedding models, like **Word2Vec**, had a major blindspot that modern models have worked hard to fix.

#### Quick Tech Tangent

If you've been working on AI systems for as long as I have, you might be familiar with **Word2Vec**. While groundbreaking when it came out in 2013, it has a major flaw: it assigns a **single vector** to each word, no matter the context.

Take the word *"bat"*.
- Are we talking about the **flying mammal**? Then it should be near *"mammal"*, *"cave"*, and *"nocturnal"*.
- Or do we mean a **baseball bat**? Then it belongs near *"ball"*, *"pitch"*, and *"base"* (but what *base*? Military?)
- And what if we're in the world of **fiction**? Then *"bat"* relates to *"vampire"* and *"transformation"*.

**Word2Vec** can't tell the difference. It picks one and sticks with it.

One thing I find particularly fascinating with Word2Vec is that, since words are now represented by numbers, you can actually do arithmetic on them.

You can make equations like 
`kind - man + woman = queen`

It's wild, but it works (most of the time).

**Tangent over.**

### How are Vectors Used?

Now that we know what vectors are, taking the next leap is relatively simple. In essence, we embed information we want the LLM to know about, and when we ask a question about that information, our very question should be close to the information when it's embedded. The vector database returns the `n` number of most-related pieces of content it has, `n` being some configured number. Along with the `n` pieces of content, it also returns its *cosine similarity* with your prompt.

In short, the cosine similarity refers to the angle between two vectors. The smaller this angle, the more relevant the data is to your prompt.

[drawing of cosine similarity

## Limitations of RAG

Typically, we don't store and retrieve entire documents in our vector database. If we did, we'd fill up the entire context window of the LLM just with a large document. If we configured our system to return the 10 most relevant pieces of information, and they're all articles of this size, your computer quickly turns into a space heater. This is why we split up the information into chunks of some configured size (E.g. 1000 characters).

But splitting information into chunks introduces a new problem. Just like Word2Vec can't figure out the context from a single word, RAG often can't figure out the context from a single chunk, especially if that chunk comes from somewhere in the middle of a document.

[comic about LLM returning information about rag-time]

Here's a problem I ran into recently. I keep a detailed work diary where I list out all my professional achievements. It's super handy during performance review season. But when I ask my RAG system what I achieved at my current company, it will confidently include things I achieved at my previous company and the one before. And, because I write this diary in first person, and I also have information from other sources, not written by me, also in the first person, it has no clue what to do with words like "I" and "me", so it starts telling me about all the cool stuff I did away from the computer (that's how I could tell those weren't my achievements, I don't leave my computer).

**Want to know how I fixed this mess?** In [the next article], I walk you through how I made my RAG system **context-aware**. You can even steal my code and set it up on your own computer with a couple of commands.
