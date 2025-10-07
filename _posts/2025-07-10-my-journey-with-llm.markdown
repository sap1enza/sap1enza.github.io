---
layout: post
title: "My Six-Month Journey to Build an AI Feature That Delivers"
tags: [python, machine-learning]
image: '/images/posts/robot-construction.webp'
---

Nearly six months ago, I've kicked off an ambitious and honestly, pretty challenging project on my present company: build something our customers would actually value at a time when “AI-powered” products were everywhere, but very few actually delivered on their promises.

From the start, our team was careful not to get caught in the hype. We’d all seen the wave of apps that promised magic and left people disappointed. We wanted to do the opposite as to create something solid, reliable, and genuinely useful.

The idea was to connect our existing template library to a process that could dramatically shorten campaign creation. Instead of starting from scratch, a customer could upload or select a layout, and our system would analyze it and automatically generate a ready-to-edit campaign that felt right from the start.

Once we defined our metrics and the product direction, I partnered with ConvertFlow’s CTO, Jonathan, to dig deep into the research. At first, we considered simply wrapping an LLM API, but after running tests, I realized it wouldn’t give us the consistency, cost control, or predictability we needed. That moment became a turning point for me since I’d always been a late adopter of AI tools, but here I fully embraced them, not only to solve this challenge but also to learn as much as I could about the field.

I started by using AI every day to reason through architectural decisions, compare trade-offs, and explore examples of different implementations. I experimented with LLM APIs to understand how embeddings worked, what tokens were, and how the models processed information. Then I moved on to visual similarity techniques, trying out perceptual hashing with Phash, measuring perceptual differences with the structural similarity index, and even training simple supervised learning models with public datasets to see how well they could classify layouts. Along the way, I also made time to study the fundamentals of machine learning, its origins, the different model types, and the implications of various algorithms not in extreme depth, but enough to have a solid foundation to make informed choices.

![Architecture Flow.](/images/posts/ml_flow.png)


After many iterations, the solution that proved most effective was a process that first identifies and segments layout elements within an image, and then processes them with ResNet50. In our case, it turned out to be a highly robust and efficient approach for computer vision and image classification.

The result is a feature that makes our users lives easier, speeds up their workflow, and genuinely delivers on its promise. For me, it’s also been one of the most rewarding projects I’ve worked on. Not just because of what we built, but because of the
skills, knowledge, and hands-on ML experience I gained along the way.

Can’t wait to see what we’ll build next!

![Remix UI](/images/posts/remix_example.png)

Feel free to try it out and let me know your feedbacks!


Available on <a href="https://www.convertflow.com" target="_blank">www.convertflow.com</a>
