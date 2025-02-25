+++
title = "RAG – How I Made My 2nd Brain Talk Back"
date = "2025-02-25T16:55:55+01:00"
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

<!-- TODO -->
[link to previous article explaining rag]

## RAG Problem #1: Context Blindness

To illustrate the problems with RAG, let's say my vector database holds detailed plots of my favorite movieseries. Let's pick two series that couldn't be further from each other: *The Matrix* and *The Lord of the Rings*. For illustration purposes, let's use a small chunk size and no overlap.

We have blindly split our movie plots into chunks and stored them in our vector database, along with their embeddings. We're even clever about it and split it up into sentences instead of `x` characters.

Now watch what might happen if you query the vector database with a question, and what it might return.

```
Input:  Who forged the one ring of power?
Output: [
  "Sauron forged the One Ring in Mount Doom to control Middle-earth.",
  "Neo is believed to be the One who will save humanity.",
  "The One Ring was made to enslave Elves, Dwarves, and Men.",
  "Agent Smith wants to destroy both humans and the Matrix.",
  "Sauron tricked the Elves and secretly made the One Ring."
]
```

```
Input:  Who or what was the Oracle in the Matrix?
Output: [
  "The Oracle is a program in the Matrix that guides Neo on his path."  
  "Galadriel, the Lady of Lothlórien, offers wisdom and foresight to the Fellowship."  
  "The Oracle helps humans understand their choices within the Matrix."  
  "Gandalf is a Maia, sent to Middle-earth to guide and counsel its people."  
  "The Oracle predicts the coming of the One who will end the war."  
]
```

If you give this information to an LLM and use it as context, it might end up answering the questions like this:

**Input:** *Who forged the one ring of power?*  

**LLM Response:**  
*The One Ring was forged by Sauron and Agent Smith to control Middle-earth and enslave Elves, Dwarves, and Men. Neo is believed to be the One who will save humanity, but for some reason, he completely ignores the other races. Sauron tricked the Elves into helping, while Agent Smith just wanted to destroy both humans and the Matrix.*

---

**Input:** *Who or what was the Oracle in the Matrix?*  

**LLM Response:**  
*The Oracle is a program in the Matrix that guides Neo on his path, much like Galadriel, the Lady of Lothlórien, who offers wisdom and foresight to the Fellowship. Gandalf, a Maia, is sent to Middle-earth to guide and counsel. Together, they predict the coming of the One who will end the war, by throwing the Matrix into Mount Doom.*

**What happened here?**

Garbage in, garbage out. The vector database isn't designed to be smart. It's designed to take an input vector, sort its own vectors by most-similar, and return the top `x` items. It doesn't know what either of the movies are. In our second prompt, we even specifically ask about The Matrix. But if you've never seen the Matrix and don't know about the movie, how are you supposed to know which of these chunks belongs to which movie? The first chunk it returns correctly answers the question, as there's literally an *Oracle* in the matrix, so that is the most similar chunk. But an oracle, by concept, offers wisdom and foresight. So, it's kind of right to come to the conclusion that Galadriel is also similar.

## RAG Problem #2: First Person Perspective Confusion

I write a lot in the first person. I write articles, technical documentation, tech talk preparations, emails, and I keep a professional diary with things I'm working on and achievements that I accomplished.

But also, I save a lot of content from other people who write in the first person. Articles I find interesting but don't have the time to read (yet), documentation of tools and applications that I like to be able to refer to. Neatly organized into directories and sub-directories they may be, but they all go into the same knowledge base, my 2nd brain.

But given the chunks *"I designed a robust data anomaly detection system."* and *"I climbed Mount Fuji in the least amount of steps."*, how is it supposed to know which one of those belongs to me (obviously it's the former, but a vector db doesn't know that).

## Solving Context Blindness

It's clear that storing individual chunks isn't going to cut it. Take a random chunk out of even a medium-sized document, and it's hard even for us humans to understand what the context is. So we need to make each chunk *context aware*.

Let's take a chunk from our example:

> The Oracle is a program in the Matrix that guides Neo on his path.

Here's what a *context aware* version of the chunk looks like:

```
<file_summary>
This file contains a detailed explanation of the Matrix film trilogy.
</file_summary>

<chunk_summary>
This chunk is about the Oracle and how she relates to Neo.
</chunk_summary>

<chunk_headers>
# The Matrix, ## The First Movie, ### Notable Characters, #### The Oracle
</chunk_headers>

<chunk_content>
The Oracle is a program in the Matrix that guides Neo on his path.
</chunk_content>
```

Even if you've never heard of The Matrix or The Oracle, this makes it clear how relevant the chunk is to a prompt like: *'Who or what was the Oracle in the Matrix?'*

*But how do we get these summaries and headers?*

We ask the LLM to summarize the entire file by providing the first and last few chunks-since intros and conclusions usually carry the strongest contextual clues. Then, using this file summary as a guide, we have the LLM create a chunk-specific summary, ensuring the chunk is tied back to the broader topic. And finally, we extract the markdown headers and include the chunk's content. We put all the pieces together and store it in a separate database field called *full_context*.

- **This *full_context* field is what we embed.**
- **The original chunk content is what the database returns**

By embedding the chunk along with its broader context, we help the RAG system understand where the chunk fits within the bigger picture. Instead of treating it as an isolated sentence, it now knows it’s part of a document about The Matrix, and that this specific chunk focuses on the Oracle’s role.

By storing the *"full context"* in a separate database field, it allows us to:
1. Check if the chunk we're processing is already in the database, so we don't have to ask an LLM to summarize it again.
2. Return the original chunk content to keep the resulting prompt as small as possible.

Remember, this is not to benefit the LLM. This is to benefit the vector database and help it figure out where to put pieces of information, so the most relevant information can be returned.

This simple addition of context isn’t just useful for decoding movie plots. In real-world scenarios—like legal document retrieval or technical knowledge bases—this strategy drastically reduces irrelevant results and hallucinations, leading to more accurate and trustworthy responses.

## Solving the First Person Confusion

This one is quite simple, and a little bit hacky. I have a directory in my knowledge base with my first name. If the chunk processor is processing any file in this directory or any of its sub-directories, I add an additional prompt instructing the LLM to replace all instances of *I*, *me*, *my* with my actual full name and to make it **"abundantly clear that this chunk is about me"**. And then when my prompt gets sent to the vector database to retrieve relevant chunks, it's also augmented with my name by simply prepending it.

Sure, this isn’t the most elegant fix, but it works. And when it comes to RAG systems, simple and effective often beats complex and fragile. By swapping out first-person pronouns with my full name in personal files, the system can now differentiate between my achievements and that travel blog I saved about climbing Mount Fuji.

# Scratch Pad

To create table in vector db:
```sql
CREATE SEQUENCE documents_id_seq;

CREATE TABLE documents (
    id BIGINT PRIMARY KEY DEFAULT nextval('documents_id_seq'),
    content TEXT,
    metadata JSONB,
    embedding VECTOR(768),
    content_hash TEXT DEFAULT 'placeholder'::text,
    summary TEXT DEFAULT 'placeholder'::text,
    full_context TEXT DEFAULT 'placeholder'::text
);
```
