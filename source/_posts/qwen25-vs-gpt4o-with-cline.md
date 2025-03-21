---
title: Qwen2.5 vs. GPT-4o - Unlocking Coding Potential with Cline
date: 2025-01-17 22:22:30
updated: 2025-01-18 01:35:00
description: If you haven't heard Cline before, just try it!
categories:
- ML/DL/LLM
- AI Agent
tags:
- large language model
- llm
- gpt
- agent
- coding assistant
- open source
- gpt-4o
- qwen2.5
- cline
- windsurf
- cursor
- azure ai
---

### Key Takeaways:

- The locally deployed open-source model Qwen2.5-72B outperforms GPT-4o with Cline.
- A more general instruction-tuned model is preferable over coding/math-specific versions when using Cline.
- Improvements in model performance could enhance other tools, such as OpenHands, which I plan to try next.

### Exploring the Coding Agent Landscape

Recently, one of the hottest topics has been coding agents. Our team explored the Cursor and Windsurf editors (both forks of VS Code) and found them to be amazing! Their capabilities, including code quality, response speed, automation, and less human feedback, are superior to the GitHub Copilot extension on VS Code. It's not surprising that Cursor and Windsurf can easily outperform GitHub Copilot (and Cline + GPT-4o which I'll introduce soon), as these products have comprehensive optimizations, including post-trained models, customized prompts, RAG, etc. However, they also come with privacy and security risks.

To adopt a coding assistant while meeting compliance requirements, the Cline extension with the GPT-4o service provided by Azure AI is an excellent option. We are grateful for the leadership team's encouragement and support in utilizing this tool.

### My Short Journey with Cline + AOAI GPT-4o

I started using Cline + AOAI GPT-4o to replace Cursor right after the callout email. However, like my teammates, I found its performance below expectations. The setup process was complicated with unclear configurations and TPM rate limits, and its capabilities were far inferior to Cursor/Windsurf. It also suffered from stability issues, including extended hangups (over 5 minutes) and code 500 request errors. The downgraded experience was disappointing for an aspiring engineer.

GPT-4o should be the best model we can use so far, we can only choose AOAI models from the base model list. Although o1-preview is also on the list, applications are required, and there is a lengthy wait for approval. While creating the GPT-4o deployment on Azure AI Foundry, I noticed that fine-tuned models were "available". This made me wonder if other models on Azure AI might be both compliant and superior to GPT-4o. Public leaderboards could answer the performance question, though verification would take effort. Azure AI only offers fine-tuning tool, which isn't ideal for our case. Azure AI Foundry's Model catalog presents another opportunity with its many popular models, but I lacked both quota and budget to experiment. Faced with either using a mediocre tool or leaving an idea unverified, I decided to locally deploy the open-source Qwen2.5 model.

### Deployment and Testing

I deployed the official Qwen2.5-72B-Instruct-AWQ model using vLLM, and the process went smoothly. Even configuring Cline proved easier than with AOAI. For smaller tasks, the model performs on par with GPT-4o. For project-level challenges like "write a Tetris game for the web using TypeScript", its planning and reasoning capabilities surpass GPT-4o. I tested both models a few times for this task - Qwen2.5 consistently provided similar, rough, and workable versions with few rounds of human feedback, while GPT-4o only succeeded once. Both outputs were buggy and required multiple rounds of instructions to refine.

The response speed for both models is about the same, which is acceptable to me. The token usage for Qwen2.5 always shows as 0, which might be a bug, but the log sizes for both models are comparable. While scaling up usage might reveal new challenges, the service and quality of Qwen2.5 currently meet my needs.

Initially, I tried Qwen2.5-Coder-32B-Instruct because of its high download ranking and coding purpose, but it occasionally struggled with Cline prompts. There is [similar issue](https://www.reddit.com/r/LocalLLaMA/comments/1gpqhgu/qwen_25_32b_coder_doesnt_handle_the_cline_prompt/) discussed on Reddit. I also noticed that Cline supports DeepSeek, which has received glowing feedback from colleagues about its DeepSeek-V3 commercial version. However, its MoE architecture with 685B parameters makes it impractical for typical personal use.

### Scaling Up for Team Deployment

I'm happy with my local server now, but how can I generalize it to the whole team? I guess the solution still lies with Azure AI model service. Interestingly, while Phi-4 is a Microsoft model, we’re unable to use it directly on Azure AI. Perhaps Microsoft is too deeply tied to OpenAI. If it's possible to reallocate a portion of the AOAI compute quotas to deploy some of these wonderful open-source models, things would be different.
