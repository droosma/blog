---
title: "Making Sense of Unstructured Data using Semantic Kernel"
description: ""
slug: parsing-unstructured-data-with-semantic-kernel
date: 2024-05-05 00:00:00+0000
original: true
image: cover.webp
categories:
    - Development
tags:
    - Generative AI
    - Semantic Kernel
    - dotnet
---

As you might have read on the blog in the past, I have been closely following the development of Generative AI and its applications in various fields. So when I recently came across a problem that required parsing unstructured data in a automated way, after having tried to push the problem back to the source and failed, my mind now went to Generative AI and how it might be able to help me solve this problem.

In this article, I will take you along on my Proof of concept solution to the problem. In my spare time, I have been playing around with a few Generative AI frameworks, and while I have been impressed with the OpenAI SDK, I have been looking for an framework that feels more production ready, and I think I have found it in Microsoft Semantic Kernel. So I will be using it to attempt to solve this problem.

I gave myself a few objectives to achieve with this Proof of Concept, and I will consider the solution a success if I can achieve the following:

1. Take hard to automatically parse text and extract structured data from it.
2. Optimize the process for speed, accuracy, and cost.
3. Bonus: Optimize import by using functions for real-time currency conversions.

## Problem: Parsing a Difficult File Format

We have all been there, you are given a assignment to integrate with a third party system, and when you start to investigate what exactly integrating with the system entails, you find out that the data you need to extract is in a format that is difficult to parse. While not the exact file format, here is a approximation of the file format I was dealing with:

```plaintext
Hey there!

I'm selling my beloved Toyota Camry. It's a fantastic car with only 100,000 miles on it. Manufactured back in October 2015. I'm looking to get $12,500 for it. Let me know if you're interested!

Contact me at: example123@email.com

🚗🚗🚗 SPECIAL OFFER! 🚗🚗🚗

Check out this Ford Mustang! It's got 80,000 kilometers on the clock and was built in May 2018. I'm selling it for €15,000. Don't miss out on this deal!

Honda Civic for sale. 150,000 km, manufactured in December 2010. Price: £8,000.

Contact John at example456@email.com for more details.

Looking to sell my BMW 3 Series. It's in great condition with only 60,000 miles on it. Manufactured in March 2017. Price is ¥2,000,000. 

Contact me at: 123-456-7890

🌟🌟🌟 DEAL ALERT! 🌟🌟🌟

Selling my Mercedes-Benz C-Class. It's got 120,000 km on it and was built in January 2019. Price: ₹20,00,000.

Contact me for more information!
```

Well, 💩 I had no idea how I was going to solve this one. When seeing a file like this my first response is usually, ok can I contact a developer on the other end. We need to sit down and figure out how we can make this more ingestible.
So I did just that, I reached out to the third party and asked if they could provide the data in a more structured format. The response I got was not what I was hoping for. They are, as you can guess from the file format, a small company and they are manually creating these files, cobbling them together from there various internal systems. They had no plans to change the format of the file, and they were not interested in helping me parse it. So there you are, stuck with a for all intents and purposes, an unparsable file that I needed to parse and extract structured data from.

Just to be clear, A human can easily parse this file, but I needed to automate the process. I needed to extract the make, model, mileage, manufacture date, and price of each car. I also needed to extract the contact information for the seller. I needed to do this for each car in the file. And while I could have done this manually, I needed to do this on a regular basis, and the file was growing in size. So I needed to automate the process.

## Why Choose Microsoft Semantic Kernel

I love keeping things simple, but I also know the value of using the right tool for the job. The OpenAI SDK is great, but for a production environment, I need a bit more. That’s where Microsoft Semantic Kernel comes in.

Microsoft Semantic Kernel is a step up from the Azure OpenAI client library, offering better abstraction, built-in orchestration, solid error handling, and a great developer experience. It handles the messy parts of API interactions with high-level commands, cutting down on boilerplate code and making your codebase cleaner and easier to manage. The built-in orchestration helps you seamlessly coordinate multiple AI services, perfect for complex applications without a ton of integration work. Plus, its robust error handling and retry mechanisms keep things running smoothly in production. With extensive documentation and an intuitive API, you can get up to speed quickly, making it a top choice for building reliable AI solutions.

### Key Advantages of Microsoft Semantic Kernel

- Advanced Capabilities and Extensibility
  - Extensibility: Semantic Kernel is super flexible, letting you easily integrate custom models, data sources, and workflows. This makes it a great fit for a variety of production needs.
  - Support for Advanced Scenarios: It comes with built-in support for advanced features like agents, caching, memory, and functions, making them easy to use in your projects.

- Support and Documentation
  - Comprehensive Support: Microsoft offers a ton of support, including detailed documentation, tutorials, and dedicated support teams for enterprise customers. This is key for troubleshooting and optimizing your production systems.
  - Community and Ecosystem: As part of the Microsoft ecosystem, Semantic Kernel benefits from a large community of developers and a rich set of tools and libraries to help you out.

With all these features and strong support, Microsoft Semantic Kernel is a great choice for building and deploying AI solutions in production.

## Implementing the Solution

Taking a step back, I realized that the file was not entirely unparseable. The file was structured in a way that each car listing was at least separated by a newline. So I could split the file into individual car listings and then extract the structured data from each car listing. The way I see it, splitting the large file into separate car listings is something a cheap and fast model like `GPT-3.5-Turbo-0125` would be able to do. So let's see how we implement this first step.

First we configure Semantic Kernel to use the Azure OpenAI. Go to the Azure portal and create a new Azure OpenAI resource. Once you have created the resource, you will be able to get the API key and the endpoint.
Next you will need to deploy the model you want to use. Go to Model deployments and than Manage deployments. Click on the Create new deployment button. Select your model and version, and give the deployment a name. You should now have all the information you need to configure Semantic Kernel to use the Azure OpenAI.

```csharp
var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName: "AZURE_OPENAI_MODEL_NAME",
                                                        endpoint: "https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey: "API_KEY",
                                                        modelId:"gpt-35")
                          .Build();
```

Next we setup a simple prompt to split the file into individual car listings.

```csharp
var executionSettings = new OpenAIPromptExecutionSettings { ModelId = "gpt-35" };
var result = await kernel.InvokePromptAsync($"""
    You are given a text which contains a number of car listings.
    Your task is to split the text into individual car listings, being careful to not omit any information.
    -----
    {fileContents}""", new(executionSettings));
```



//https://github.com/microsoft/semantic-kernel/blob/main/dotnet/samples/Concepts/Filtering/RetryWithFilters.cs


1. Split the file into individual car listings.
2. Extract structured data from each car listing.
3. Split original file into chunks to make sure we don't hit the token limit.

## Perform Currency Conversion

## Conclusion
