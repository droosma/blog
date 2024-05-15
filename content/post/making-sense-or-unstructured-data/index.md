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

As I have some familiarity with how the GPT LLM models work, there are a few considerations I will take into account when implementing the solution. The first is that the file can be large. So I will need to make sure I don't hit the token limit. The second is that I need to make sure to use the right model for the right job. Given that at time of writing the best and largest model is `GPT-4-Turbo` with a `128.000` token context at € 0.010/1000 tokens. Meaning a request can cost as much as € 1.28, yikes. So do I really need all the power of `GPT-4-Turbo` for this task? I think not. I will start with a cheaper model like `GPT-3.5-Turbo-0125` at € 0.0005/1000 tokens, and see how well that performs on these relatively simple tasks. This choice does have the implication that I will need to spend more time on the implementation to make sure I don't hit the token limit. as the limit for this model is 16.000 tokens per request.

First things first, we are going to need access to the GPT models, as our cloud provider of choice is Azure, Azure OpenAI it is. If you want to follow along, given that this service is still not available to all, you will need to request access for you Azure subscription, try [here](https://azure.microsoft.com/en-us/products/ai-services/openai-service/) and click on `Apply for access` it usually takes about 24h for someone on the azure side to approve your application, so please account for that.
Once you have access, you will need to create a new Azure OpenAI resource. I would love to give you a step by step guide on how to do this, but the process is in flux and I don't want to give you outdated information. So I will just say, go to the Azure portal and create a new Azure OpenAI resource. Once you have created the resource, you will be able to get the API key and the endpoint. You will also need to deploy the model you want to use, being careful to remember the name you gave your model deployment as you will be needing this later.

Right, let's get started. First we need to create a new project and add the Microsoft.SemanticKernel package to it. You can do this by running the following command in the terminal:

```powershell
dotnet new console -n ParsingUnstructuredData && cd ParsingUnstructuredData && dotnet add package Microsoft.SemanticKernel
```

Now that we have our project setup, we can start implementing the solution. First we need to setup the kernel builder and add the Azure OpenAI chat completion service to it. After which your `Program.cs` file should look something like this:

```csharp
using Microsoft.SemanticKernel;

var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                        endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey:"API_KEY")
                          .Build();

Console.WriteLine("Hello, World!");
```

As you can see, this is the spot where you will need to add your Azure OpenAI model deployment name, endpoint, and API key.

So as mentioned earlier, I want to use `GPT-3.5-Turbo-0125` to do the heavy lifting. So let's start with something simple, let's see if is able to split the file into individual car listings.
To keep things simple I'm just going to read the from my file system and read it into a string, this is not the recommended way to do this in production, but it is enough for the scope of this article.

Adding the following code to the `Program.cs` file should do the trick:

```csharp
var fileContents = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, "importFile.txt"));

var result = await kernelBuilder.InvokePromptAsync($"""
                                                    You are given a text which contains a number of car listings.
                                                    Your task is to split the text into individual car listings, being careful to not omit any information.
                                                    -----
                                                    {fileContents}
                                                    """);

Console.WriteLine(result);
Console.ReadLine();
```

In the code above, we read the file `importFile.txt` I added to the project with a `CopyToOutputDirectory` property set to always. Now that we have the file contents, we invoke the GPT model with a simple prompt asking it to split the file. After running the code, the output should look something like this:

```plaintext
Car 1:
Make: Toyota
Model: Camry
Mileage: 100,000 miles
Manufacture Date: October 2015
Price: $12,500
Contact: example123@email.com

Car 2 (Special Offer):
Make: Ford
Model: Mustang
Mileage: 80,000 kilometers
Manufacture Date: May 2018
Price: ?15,000
Contact: No contact provided

Car 3:
Make: Honda
Model: Civic
Mileage: 150,000 km
Manufacture Date: December 2010
Price: £8,000
Contact: John at example456@email.com

Car 4:
Make: BMW
Model: 3 Series
Mileage: 60,000 miles
Manufacture Date: March 2017
Price: ¥2,000,000
Contact: 123-456-7890

Car 5 (Deal Alert):
Make: Mercedes-Benz
Model: C-Class
Mileage: 120,000 km
Manufacture Date: January 2019
Price: ?20,00,000
Contact: No contact provided
```

Color me impressed, it was able to split the file into individual car listings, even formatting the output into a uniform structure. This is a great start, but we are not done yet. While the file is technically no longer unstructured, we still need to parse it to something our system can deal with. Given the tendency of the GPT models to be non-deterministic, we will need to add some structure to the prompt to make sure we always get the same output.

```csharp
var arguments = new KernelArguments {{"content", fileContents}};
var result = await kernelBuilder.InvokePromptAsync("""
                                                   You are tasked with splitting a large text into individual blocks, each describing a single car listings. Below is the text content from a file:
                                                   ```
                                                   {{$content}}
                                                   ```

                                                   ### Tasks:

                                                   1. Ensure no information is omitted. Include all text as it appears in the file.
                                                   2. Produce the output in valid JSON format. The output must be directly parsable into an Array of Strings, each string representing a single listings description.

                                                   ### Sample Output:
                                                   [
                                                       "Description text for the first listing",
                                                       "Description text for the second listing"
                                                   ]
                                                   """,
                                                   arguments);
var listings = JsonSerializer.Deserialize<IEnumerable<string>>(result.ToString());
Console.WriteLine(string.Join($"{Environment.NewLine}===={Environment.NewLine}", listings));
Console.ReadLine();
```

As you can see, we added a bit more structure to the prompt. First moved the file contents from the string interpolation to a argument. We also added more groundings to the prompt, making sure the output is something we can automatically parse, which we do with a `System.Text.Json`. There are of course more ways to structure the output, but I have had a lot of success with this `json` approach in the past.

After running the code, the output should look something like this:

```text
I'm selling my beloved Toyota Camry. It's a fantastic car with only 100,000 miles on it. Manufactured back in October 2015. I'm looking to get $12,500 for it. Let me know if you're interested!

Contact me at: example123@email.com
====
SPECIAL OFFER!

Check out this Ford Mustang! It's got 80,000 kilometers on the clock and was built in May 2018. I'm selling it for ?15,000. Don't miss out on this deal!
====
Honda Civic for sale. 150,000 km, manufactured in December 2010. Price: £8,000.

Contact John at example456@email.com for more details.
====
Looking to sell my BMW 3 Series. It's in great condition with only 60,000 miles on it. Manufactured in March 2017. Price is ¥2,000,000.

Contact me at: 123-456-7890
====
DEAL ALERT!

Selling my Mercedes-Benz C-Class. It's got 120,000 km on it and was built in January 2019. Price: ?20,00,000.

Contact me for more information!
```

Nice, we now have a list of car listings, each in a separate string. You might be wondering why I'm introducing this intermediate step, and the reason is simple. I expect the parsing of the individual car listings to be a bit more complex, with this intermediate step I can now also parallelize the parsing of the individual car listings, making the process faster. An added bonus is that it's highly unlikely that I will hit the token limit for subsequent requests. And lastly, given that the context for a subsequent request is now much smaller, this reduces the chance of the model missing important information or taking a wrong turn and getting confused.

//Expand the demo, introduce the gpt-4 model, parallel the subsequent requests into that other model.

//Do the initial file split with overlapping chunks to make sure we don't miss any information.

//Add the currency conversion part.

//https://github.com/microsoft/semantic-kernel/blob/main/dotnet/samples/Concepts/Filtering/RetryWithFilters.cs


1. Split the file into individual car listings.
2. Extract structured data from each car listing.
3. Split original file into chunks to make sure we don't hit the token limit.

## Perform Currency Conversion

## Conclusion
