---
layout: post
title: "Training a ML model on browser"
tags: [tensorflowjs, javascript, browser, machine-learning]
image: '/images/posts/tensorflow.png'
---

# Building a Product Recommendation System That Runs Entirely in the Browser

*Part of my post-graduation in Applied AI Engineering at UniPD*

I've been going through a post-graduation program in Applied AI Engineering at UniPD. One of the modules covered recommendation systems at a conceptual level, which was enough to get me curious but not enough to feel like I actually understood what was happening. So I decided to extend it into a proper hands-on implementation and see how far I could take it. I ended up building something that felt slightly ridiculous on paper: a neural recommendation model that trains **directly in the browser**, with no server, no Python, no cloud GPU. Just a webpage.

---

## The dataset

I started with a real e-commerce dataset from Kaggle: [Customer Shopping Behaviour Analysis](https://www.kaggle.com/datasets/wardabilal/customer-shopping-behaviour-analysis), which has around 3,900 rows of customer data. Each row is one customer with a mix of demographic info (age, gender, location), behavioural signals (purchase frequency, payment method, whether they used a discount), and what they actually bought (item, category, color, season).

The goal I set for myself: given a customer profile, recommend which products they're most likely to want. Classic recommendation problem, deliberately constrained to run entirely client-side.

---

## Starting simple: doing it by hand

Before reaching for any ML library, I went to understand what the data actually required. Recommendation models are fundamentally about turning mixed data types (numbers, categories, binary flags) into something a neural network can process. That means normalisation and encoding.

My first pass did this manually. Numeric fields like age got min-max normalised by scanning the dataset and computing the range. Categorical fields like gender or season got label-encoded into integers by building a simple lookup dictionary. I wrote the forward pass myself too: matrix multiplications, a sigmoid activation, the whole thing, just to feel how the pieces fit together.

It worked, in the sense that it ran. But it was brittle, slow to iterate on, and I was spending most of my time debugging tensor shapes rather than thinking about the actual model. There's real value in writing things from scratch once. After that, you've earned the right to use a library.

---

## Switching to TensorFlow.js

TensorFlow.js turned out to be a surprisingly good fit for this project. It runs in the browser via WebGL acceleration, has the same model-building API as Keras, and, crucially for this project, models can be saved to IndexedDB and loaded back without any server involved.

The shift from manual to TF.js wasn't just a tooling swap. It changed what kind of model I could realistically build. With the plumbing handled, I could focus on architecture decisions that actually matter.

---

## The model: neural collaborative filtering

The approach I landed on is called **neural collaborative filtering**. The core idea is to score every possible (user, product) pair (how likely is *this* user to buy *this* product?) and train the network to output 1 for pairs that actually happened in the dataset and 0 for everything else.

The interesting part is how you represent "this user" and "this product" as inputs. Raw category strings are useless to a neural network, and one-hot encoding 50 US states would be wasteful. The standard solution is **embedding layers**: each category gets mapped to a small dense vector that the network learns during training. By the end of training, similar categories end up close together in embedding space. The model figures out that "Weekly" and "Bi-Weekly" purchase frequencies have something in common without you telling it.

The network ends up with ten input branches, one per feature, that get concatenated and passed through two dense layers before producing a single sigmoid score:

```
Product: item · category · color · season     → embedding layers
User:    age · discount                        → numeric
         gender · location · frequency        → embedding layers
         taste (purchase history)             → shared item embedding → avg pool
                  ↓
           Concatenate all features
                  ↓
            Dense(64, relu) → Dropout(0.2) → Dense(32, relu)
                  ↓
            Dense(1, sigmoid)  →  score ∈ [0, 1]
```

One detail I'm particularly happy with: the **item embedding layer is shared** between the product side and the user's taste side. When the model embeds "what item is this product?" and "what item did this user buy?", it uses the exact same learned vectors. That means the model naturally learns to score items as more relevant when they're similar to what the user has bought before. Not because I told it to, but because the shared embedding space makes that the easiest thing to learn.

---

## Running training in the browser: the Web Worker problem


**Web Worker** is a separate thread that runs JavaScript in the background without touching the DOM. The architecture ended up being:

- Main thread handles the UI, reads the form, dispatches events
- Worker receives the dataset and training config, does all the TF.js work, and streams log messages back to the main thread after each epoch
- The main thread updates the training log display as messages arrive

Communication between them happens through `postMessage`, which means everything that crosses the thread boundary has to be serialisable. The model itself never leaves the worker. Only lightweight log objects (epoch, loss, accuracy) get sent across.

---

## Persisting the model so you don't retrain every time

Training on 3,900 customers × 26 products gives around 100,000 training pairs. That takes a few minutes even with WebGL. Refreshing the page and starting over every session would be unbearable.

TF.js has a clean solution for this: `model.save('indexeddb://recommendation-model')`. IndexedDB is a proper browser database that persists across page loads and, unlike localStorage, is accessible from Web Workers. The model topology and weights go there.

The vocabulary context (all the index maps, the product catalog, the normalisation stats) is a small JSON object that goes to `localStorage` via the main thread.

On the next page load, the app checks localStorage for a saved context. If it finds one, it sends it to the worker which loads the model from IndexedDB and signals back that it's ready. The whole restore path takes a couple of seconds instead of minutes.

---

## The UI: a study tool, not a product

The interface was deliberately designed as an interactive study tool using Claude Code. There are two panels on the left side: a customer profile form (age, gender, location, season, purchase frequency, discount status, number of previous purchases, payment method) and a training configuration panel (epochs, batch size, learning rate, validation split, optimizer choice, shuffle toggle). On the right is the recommendations output.

The idea is that you change a profile field, say switch from "Weekly" to "Annually" purchase frequency, run the recommendation again, and watch the ranked products shift. You can also re-train with different hyperparameters and compare how aggressively the model converges.

The tech stack underneath is intentionally minimal: plain HTML, Tailwind CSS v4, and vanilla ES modules. No frontend framework. The ML stack is TF.js core + TF.js Vis for the training charts.

---

## What I took away from this

A few things stuck with me after building this:

**The browser is a surprisingly capable ML runtime.** WebGL-backed tensor operations are fast enough for this kind of model. The constraints (no Python, no server) forced cleaner thinking about data flow and thread boundaries. ( It's also easier to setup and play with )

**Embedding layers are one of the most useful ideas in deep learning.** They turn the vague concept of "similar categories" into something a network can actually learn. Once you've seen them work you start wanting to apply them everywhere.

**Building a thing you understand beats running a notebook you don't.** I've run plenty of Jupyter notebooks that produced a model I couldn't fully explain. This project forced me to understand every step: what shape are the tensors, why does training a cross-product of pairs work, what does the sigmoid output actually mean. That friction was the point.

---

## Links

- **GitHub repo:** *(coming soon)*
- **Dataset:** [Customer Shopping Behaviour Analysis on Kaggle](https://www.kaggle.com/datasets/wardabilal/customer-shopping-behaviour-analysis)
- **TF.js Vis:** [https://js.tensorflow.org/api_vis/latest/](https://js.tensorflow.org/api_vis/latest/)
- [Deep Learning Model with TensorFlow.js — IBM Developer](https://developer.ibm.com/tutorials/coding-a-deep-learning-model-using-tensorflow-javascript/)
- [Finding Intent to Buy with TensorFlow.js — George Perry](https://medium.com/@GeorgePerry/finding-intent-to-buy-from-instagram-comments-with-tensorflow-js-3f764c132be7)
- [Reinforcement Learning in the Browser — Pierre Rouhard](https://medium.com/@pierrerouhard/reinforcement-learning-in-the-browser-an-introduction-to-tensorflow-js-9a02b143c099)
- [tfjs-mountaincar — GitHub](https://github.com/prouhard/tfjs-mountaincar)
- [Reinforcement Learning in JavaScript — Scribbler](https://scribbler.live/2024/06/28/Reinforcement-Learning-in-JavaScript.html)
- [Dino Run with TF.js — Paperspace](https://blog.paperspace.com/dino-run/)
- [Neural Networks Explained — AWS MLU](https://mlu-explain.github.io/neural-networks/)
- [AWS MLU Explain — GitHub](https://github.com/aws-samples/aws-mlu-explain)
- [Awesome ML Visualizations — Analytics Vidhya](https://medium.com/analytics-vidhya/awesome-machine-learning-visualizations-5208f1617ec5)
- [Vector Storage for the Browser — Nitai Aharoni](https://nitaiaharoni1.medium.com/introducing-vector-storage-a-lightweight-vector-database-for-the-browser-bc10775fd9dd)
