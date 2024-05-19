---
title: "Making Sense of Unstructured Data using Semantic Kernel"
description: "Explore the challenges and solutions of parsing unstructured data using Generative AI in this detailed blog post. Discover how Microsoft Semantic Kernel enhances the process with robust features like multiple model integration and custom function plugins. Follow the journey from concept to implementation, including code samples and insights into making AI work in production environments."
slug: parsing-unstructured-data-with-semantic-kernel
date: 2024-05-19 00:00:00+0000
original: true
image: cover.webp
categories:
    - Development
tags:
    - Generative AI
    - Semantic Kernel
    - dotnet
---

Hey there! If you've been following my blog, you know how fascinated I am with Generative AI and its growing applications. Recently, I stumbled upon a tricky problem: parsing unstructured data automatically. Despite my best efforts to get cleaner data from the source, I had to find another solution. That’s when Generative AI came to mind.

In this post, I’ll take you through my journey of creating a proof of concept to tackle this challenge. I'll run though my choice of Semantic Kernel, I've dabbled with various Generative AI frameworks in my spare time. Although the OpenAI SDK impressed me, I wanted something more robust for production. I'll show you the steps I took and my thought process behind them. And I'll touch on some advanced features like using multiple models and creating and calling functions.

Here are the goals I set for my proof of concept:

1. Extract structured data from tough-to-parse text.
2. Make the process fast, accurate, and cost-effective.
3. Bonus: Use real-time currency conversion functions to streamline imports.

Curious to see how it all worked out? Stick around, and I'll walk you through it. All the code samples are available in this [GitHub repository](https://github.com/droosma/parsing-unstructured-data-with-semantic-kernel).

## The Problem: Parsing a Difficult File Format

We've all been there—assigned to integrate with a third-party system, only to discover that the data we need is in a format that's a nightmare to parse. Let me show you an example similar to what I was dealing with:

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

Well, 💩, I had no idea how to handle this. My first instinct was to contact a developer on the other end and ask if we could make the data more manageable.

So, I reached out to the third party, hoping they could provide the data in a structured format. Unfortunately, they’re a small company, manually creating these files from various internal systems. They had no plans to change the format or help me parse it.

Here I was, stuck with an almost unparseable file that I needed to extract structured data from. While a human could easily understand this file, I needed to automate the process. I had to extract the make, model, mileage, manufacture date, and price of each car, along with the seller’s contact information. Doing this manually wasn't an option since the file was growing in size and I needed to do this regularly. Automation was the only way forward.

So, I turned to Generative AI to see if it could help me out. I had some experience with the OpenAI SDK, but I needed something more robust for production. That's where Microsoft Semantic Kernel came in.

## Why Choose Microsoft Semantic Kernel

I love keeping things simple, but I also know the value of using the right tool for the job. The OpenAI SDK is great, but for a production environment, I need a bit more. That’s where Microsoft Semantic Kernel comes in.

Microsoft Semantic Kernel improves upon the Azure OpenAI client library by offering better abstraction and a simpler developer experience. It handles complex API interactions efficiently, minimizing repetitive code and keeping your codebase tidy. This system is particularly useful for applications that need to integrate multiple AI services smoothly. If you’re interested in how it manages different models, take a look at the "Using Multiple Models" section. The Semantic Kernel also provides reliable error handling and robust retry mechanisms to ensure steady performance in production settings. With its comprehensive documentation and intuitive API, you can quickly learn how to use it effectively, making it a practical choice for building dependable AI solutions.

### Key Advantages of Microsoft Semantic Kernel

- Advanced Capabilities and Extensibility
  - The Semantic Kernel is highly adaptable, allowing for the easy integration of custom models, data sources, and workflows. This flexibility makes it suitable for a wide range of production environments.
  - It includes built-in support for complex features such as agents, caching, memory, and functions, simplifying their incorporation into your projects.

- Support and Documentation
  - Microsoft provides extensive support, with detailed documentation, tutorials, and access to dedicated support teams for enterprise customers. This comprehensive assistance is crucial for troubleshooting and enhancing your systems.
  - Being part of the Microsoft ecosystem, the Semantic Kernel benefits from a robust community of developers and a wide array of tools and libraries, which can provide additional help and resources.

With its versatile features and strong backing, Microsoft Semantic Kernel offers a solid foundation for building and deploying AI solutions in production environments. So having seen the potential of the Semantic Kernel, I decided to give it a try for my unstructured data parsing problem.

## Implementing the Solution

With some familiarity with GPT LLM models, there are a few considerations to keep in mind when implementing this solution. Firstly, the file may be large, so avoiding the token limit is crucial. Secondly, choosing the right model is important. As of now, the best and largest model is GPT-4-Turbo with a 128,000 token context at €0.010 per 1,000 tokens, potentially costing up to €1.28 per request. That’s quite expensive. However, for this task, the full power of GPT-4-Turbo may not be necessary. I’ll start with a more cost-effective model, GPT-3.5-Turbo-0125, at €0.0005 per 1,000 tokens, but I need to be mindful of its 16,000 token limit.

Firstly, to access the GPT models through our cloud provider, Azure, you'll need to request access for your Azure subscription. You can apply for access on [Azure's OpenAI service page](https://azure.microsoft.com/en-us/products/ai-services/openai-service/), usually approved within 24 hours.

Once you have access, create a new Azure OpenAI resource in the Azure portal. After setup, you’ll receive an API key and an endpoint URL. Don’t forget to deploy the model and note down the deployment name.

Now, let’s start by creating a new project and adding the Microsoft.SemanticKernel package:

```powershell
dotnet new console -n ParsingUnstructuredData && cd ParsingUnstructuredData && dotnet add package Microsoft.SemanticKernel
```

Next, set up the kernel in your Program.cs and configure it to use the Azure OpenAI chat completion service:

```csharp
using Microsoft.SemanticKernel;

var kernel = Kernel.CreateBuilder()
                   .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                 endpoint:"END_POINT",
                                                 apiKey:"API_KEY")
                   .Build();

Console.WriteLine("Hello, World!");
```

Replace placeholders with your actual deployment name, endpoint, and API key.

As a start, let's try splitting the file into individual car listings. We’ll read the file contents into a string for simplicity (note, this is not recommended for production):

```csharp
var fileContents = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, "importFile.txt"));

var result = await kernel.InvokePromptAsync($"""
                                            You are given a text which contains a number of car listings.
                                            Your task is to split the text into individual car listings, being careful to not omit any information.
                                            -----
                                            {fileContents}
                                            """);

Console.WriteLine(result);
Console.ReadLine();
```

In this setup, `importFile.txt` should be included in the project with the `CopyToOutputDirectory` property set to `Always`, ensuring it’s always available at runtime.

Upon running, the output will display each car listing structured uniformly:

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

... And so on
```

Great! The file was successfully split into individual car listings and formatted uniformly. However, to achieve consistent automated parsing, we need to revise the prompt. This adjustment not only caters to the non-deterministic nature of GPT models but also provides more grounding, helping to stabilize the outputs.

```csharp
var arguments = new KernelArguments {{"content", fileContents}};
var result = await kernel.InvokePromptAsync("""
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

//For demonstration purposes we output the listings to the console
Console.WriteLine(string.Join($"{Environment.NewLine}===={Environment.NewLine}", listings));
Console.ReadLine();
```

As you can see, we've structured the prompt a bit more by moving the file contents from string interpolation to an argument, enhancing clarity and order in the code. Additionally, by refining the prompt we ensure the output can be easily parsed automatically using `System.Text.Json.JsonSerializer`. Using JSON as the output format from Large Language Models (LLMs) like Microsoft's Semantic Kernel is highly effective because it provides a structured, predictable format that facilitates clear task definitions, which is crucial for obtaining accurate and useful responses from the model. However, I encourage you to experiment and find what works best for your specific needs.

The output now, visually segmented for clarity:

```plaintext
I'm selling my beloved Toyota Camry. It's a fantastic car with only 100,000 miles on it. Manufactured back in October 2015. I'm looking to get $12,500 for it. Let me know if you're interested!

Contact me at: example123@email.com
====
SPECIAL OFFER!

Check out this Ford Mustang! It's got 80,000 kilometers on the clock and was built in May 2018. I'm selling it for ?15,000. Don't miss out on this deal!
====
Honda Civic for sale. 150,000 km, manufactured in December 2010. Price: £8,000.

Contact John at example456@email.com for more details.

... And so on
```

Nice, we now have a list of car listings, each presented as a separate string. You might wonder why I introduced this intermediate step. The reason is straightforward: parsing individual car listings is expected to be more complex, and limiting the context for an LLM like this reduces the risk of the model deviating and producing inaccurate content, known as "hallucinating." Additionally, breaking down a large task into smaller segments allows for parallel processing of each listing, which speeds up the overall process. Another significant advantage is that this method nearly eliminates the risk of exceeding the model's token limit.

Now, let’s dive into parsing an individual car listing. The process mirrors the initial file splitting, but we'll refine our prompt to ensure we capture all necessary details:

```csharp
private async Task<Listing> ExtractListing(string contents)
{
    var arguments = new KernelArguments { { "content", contents } };
    var result = await kernel.InvokePromptAsync("""
                                                Task: Convert a car listing into a structured JSON format. Below is the text content from a file:
                                                ```
                                                {{$content}}
                                                ```

                                                Requirements:
                                                1. Ensure no information is omitted. Include all text as it appears in the file.
                                                2. Produce the output in a valid JSON format that can be directly parsed into a Listing object.

                                                Sample Output:
                                                {
                                                    "Make": "Toyota",
                                                    "Model": "Highlux",
                                                    "Odometer": "100000",
                                                    "ManufacturerDate": "2000-05-29",
                                                    "Price": "16900",
                                                    "Contact": "contact@example.com"
                                                }
                                                """,
                                                arguments);
    var response = result.ToString();
    return JsonSerializer.Deserialize<Listing>(response);
}
```

This structured approach ensures that each listing is meticulously parsed, maintaining the integrity of the data while streamlining the extraction process.

As we want to call this prompt for each individual car listing, I created a wrapper function that takes the contents of a single car listing and returns a `Listing` object. The `Listing` object is a simple record that represents the structured data we want to extract from the car listing. The `Listing` record is defined as follows:

```csharp
public record Listing(
    string Make,
    string Model,
    string Odometer,
    string ManufacturerDate,
    string Price,
    string Contact)
{
    public override string ToString() => $"{Make} {Model} [{Odometer}/{ManufacturerDate}] - {Price} | {Contact}";
}
```

This ensures a clean and organized structure, making each listing easy to handle and further process or display.

I also encapsulated the entire execution of the in a `Execute` method, which orchestrates the parsing and processing of car listings from unstructured text data:

```csharp
public async Task Execute(string contents)
{
    var listingTexts = await ExtractListings(contents);
    var listingTasks = listingTexts.AsParallel()
                                   .Select(ExtractListing);
    var listings = await Task.WhenAll(listingTasks);

    Console.WriteLine(string.Join<Listing>(Environment.NewLine, listings));
}
```

In this code, we start by calling the `ExtractListings` function to split the text file into individual car listings. We then enhance performance by initiating parallel processing of each listing to convert them into structured data using the `AsParallel` method. This approach utilizes multiple threads, speeding up the process significantly. Once all parallel tasks are completed, synchronized with `Task.WhenAll`, the structured data is then printed to the console.

When you run this code, it produces a neatly formatted output, displaying detailed information about each car:

```plaintext
BMW 3 Series [60,000/2017-03] - ¥2,000,000 | 123-456-7890
Ford Mustang [80000/2018-05] - 15000 |
Honda Civic [150000/2010-12] - 8000 | John@example456.email.com
Mercedes-Benz C-Class [120,000 km/January 2019] - ?20,00,000 | contact@example.com
Toyota Camry [100,000 miles/October 2015] - $12,500 | example123@email.com
```

The final step in our data processing pipeline involves ensuring we do not hit the token limit for the `ExtractListings` method, having already managed this risk for the `ExtractListing` method by limiting each invocation to a single listing. To achieve this, we split the initial large file into manageable chunks before sending each to the model. This is a simple yet crucial process and requires a bit of code to implement correctly. Let me show you how I tackled this:

```csharp
public static IEnumerable<string> GetChunks(string content, 
                                            int chunkSize = 100, 
                                            int overlapLines = 10)
{
    var lines = content.Split(Environment.NewLine);
    return Enumerable.Range(0, (lines.Length + chunkSize - 1) / chunkSize)
                     .Select(i => string.Join(Environment.NewLine,
                                              lines.Skip(i * chunkSize)
                                                   .Take(chunkSize + overlapLines)));
}
```

The `GetChunks` function breaks down the file contents into chunks based on a specified size. The `chunkSize` parameter sets the number of lines in each chunk, and `overlapLines` adds a buffer by overlapping lines between chunks. This overlapping ensures that no car listing is inadvertently split across two chunks, thus maintaining the integrity of the data despite the risk of potential duplicate entries.

The updated implementation of the `Execute` function is outlined below:

```csharp
public async Task Execute(string contents)
{
    var chunks = GetChunks(contents);
    var listingTextTasks = chunks.AsParallel()
                                 .Select(ExtractListings)
                                 .ToList();
    var listingTexts = await Task.WhenAll(listingTextTasks);

    var listingTasks = listingTexts.SelectMany(listingText => listingText)
                                   .AsParallel()
                                   .Select(ExtractListing)
                                   .ToList();
    var listings = await Task.WhenAll(listingTasks);

    Console.WriteLine(string.Join<Listing>(Environment.NewLine, listings));
}
```

In this approach, we first split the file into chunks using the `GetChunks` function. These chunks are then processed in parallel with `ExtractListings` to extract individual car listings. Finally, `Task.WhenAll` synchronizes the parallel tasks, ensuring that all data is processed before printing the structured listings to the console. This method not only minimizes the risk of exceeding token limits but also maintains a high processing speed.

By executing this code, you can achieve the same detailed output as before while effectively managing larger data volumes. We have successfully transformed unstructured data into structured data, ready for further processing or analysis using more traditional methods. So now we have a system that can handle large volumes of unstructured data, and parse it into structured data. This system is not only efficient but also cost-effective, as it uses the right model for the right task, ensuring optimal performance without unnecessary expense. There are a few more things we can do to enhance this system further. If you are curious about the additional capabilities of Microsoft's Semantic Kernel, let's explore some advanced features next.

## Creating and calling functions

As previously discussed, to make the structured data from car listings more usable, we need further processing. For instance, standardizing the price to a common currency, parsing dates into `DateTime` objects, or converting the odometer readings from various units to a uniform integer in kilometers, are some enhancements that could greatly improve usability. To facilitate these transformations, we can create functions dedicated to handling each specific task.

However, integrating these functions directly into the flow controlled by the LLM poses a challenge. Currently, there is no direct method to have the model invoke external functions as part of its processing pipeline using the basic model invocation methods we've used so far. To address this, we need to utilize the `IChatCompletionService` provided with the Microsoft Semantic Kernel. This service is typically designed to simplify the creation of chat applications but can be adapted for our purposes to orchestrate complex workflows involving external function calls.

Here’s how we can set up the `IChatCompletionService`:

```csharp
using Microsoft.SemanticKernel;

var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                        endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey:"API_KEY");
var kernel = kernelBuilder.Build();

var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();
```

In this setup, we begin by constructing the kernel builder, just as we did previously, and add the Azure OpenAI chat completion service. However, this time we also integrate the `IChatCompletionService` provided by the Semantic Kernel. This service is a higher-level abstraction designed specifically for creating chatbot-like interactions, similar to those managed by [`ChatGPT`](https://chat.openai.com/). Unlike traditional methods that handle single prompts, `IChatCompletionService` manages a continuous conversation history, allowing the model to maintain context over the course of an interaction.

Here’s how you can implement the ExtractListing using this approach:

```csharp
public async Task<Listing> ExtractListing(string listingText)
{
    var history = new ChatHistory
                  {
                      new(AuthorRole.System,
                          """
                          tasked with converting a single car listing into a sting structured JSON format

                          ### Tasks:

                          1. Ensure no information is omitted. Include all text as it appears in the file.
                          2. Produce a single valid JSON representation of the object. 
                          3. The output must be directly parsable into an Listing object.

                          ### Sample Response:
                          {
                              "Make":"Toyota",
                              "Model":"highlux",
                              "Odometer":"100000",
                              "ManufacturerDate":"2000-05-29",
                              "Price": extracted price as decimal converted to dollar,
                              "Contact":"contact@example.com"
                          }
                          """),
                      new(AuthorRole.User, listingText)
                  };
    var result = await chatCompletionService.GetChatMessageContentAsync(history);

    var response = result.Content;
    return JsonSerializer.Deserialize<Listing>(response);
}
```

In this setup, we use the ChatHistory object to simulate a conversation with the model. The conversation starts with a system-generated message that outlines the task for the model, explaining exactly how the car listing should be processed. This is followed by the user (our application) providing the actual car listing text.

- AuthorRole.System: This role is used to provide instructions or context to the model, guiding its response pattern.
- AuthorRole.User: This role represents the input from the user, which in this case is the car listing that needs processing.

The `GetChatMessageContentAsync` method is then called with the structured conversation history. This method sends the entire conversation to the model and retrieves the structured JSON output, which we then deserialize into a Listing object.

With the transition to using `IChatCompletionServic`e and further structuring our code, we've introduced a more sophisticated system that allows for expandable functionality through what the Semantic Kernel refers to as [Plugins](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/?tabs=Csharp). Plugins enable us to extend the capabilities of our application seamlessly by integrating custom functions directly into the processing workflow of the Semantic Kernel.

To illustrate how plugins can be utilized, let's consider a practical example of a currency conversion plugin. This plugin will allow our system to convert various currency amounts into US dollars (USD) based on a simplified random conversion rate. Below is the implementation of such a plugin:

```csharp
public sealed class CurrencyPlugin
{
    private static readonly Random _random = new();

    [KernelFunction,
     Description("Currency amount and returns the equivalent amount in USD")]
    public static decimal ConvertToDollar([Description("The ISO 4217 currency code")] string currencyCode,
                                          [Description("The amount of money to convert")] decimal amount)
        => amount * (decimal)(_random.NextDouble() * 2);
}
```

In this example, we're using a random multiplier to simulate the conversion process. This is purely for demonstration; in a practical setting, you’d replace this with a call to a genuine currency conversion service that provides real-time exchange rates. The `KernelFunction` attribute marks ConvertToDollar as a plugin function, signaling to the Semantic Kernel that it can be called as part of its operational workflow. The `Description` attributes ensure that the purpose of the function and its parameters are clear for the LLM.

To integrate our newly created `CurrencyPlugin` into the Semantic Kernel's workflow, we need to register the plugin with the kernel. This is done by adding the plugin to the kernel builder. Here’s how you can do it:

```csharp
var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                        endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey:"API_KEY");
kernelBuilder.Plugins.AddFromType<CurrencyPlugin>();
var kernel = kernelBuilder.Build();
```

With the `CurrencyPlugin` now part of our kernel configuration, it's ready to be invoked as needed. The next step involves adjusting how we call the `IChatCompletionService` to leverage this new capability.

```csharp
var settings = new OpenAIPromptExecutionSettings
               {
                   ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
               };
var result = await _completionService.GetChatMessageContentAsync(history, settings, kernel);
```

By modifying the `OpenAIPromptExecutionSettings`, specifically the `ToolCallBehavior` property to `AutoInvokeKernelFunctions`, we instruct the system to automatically call our plugin functions when certain conditions within the chat content are met.

With these changes implemented, when you run the code, the interaction will not only parse the car listings but also dynamically convert the price of each listing into USD using the CurrencyPlugin. This random conversion demonstrates the plugin's functionality, although in a live environment, you would likely use a more deterministic method tied to actual currency exchange rates. This enhancement makes the output not only more uniform but also more adaptable to varying international inputs, illustrating a significant leap in the system's capability to handle diverse data types and requirements.

Using the `CurrencyPlugin`, we can now observe the transformation in how our data processing. Given the original car listing:

```plaintext
I'm selling my beloved Toyota Camry. It's a fantastic car with only 100,000 miles on it. Manufactured back in October 2015. I'm looking to get € 12,500 for it. Let me know if you're interested!

Contact me at: example123@email.com
```

the listing is transformed into the following JSON:

```json
{
    "Make":"Toyota",
    "Model":"Camry",
    "Odometer":"100000",
    "ManufacturerDate":"2015-10-01",
    "Price": 6423.30,
    "Contact":"example123@email.com"
}
```

Here, we see a clear transformation: the car’s price is now shown in USD thanks to the CurrencyPlugin. This example demonstrates how plugins can neatly organize data and convert values. Yet, implementing these plugins can sometimes bring up challenges.

Even with the advanced technology behind the Semantic Kernel and its plugins, there are times when the model might not react to the prompts as we expect. This issue might require us to make some adjustments to the prompts to help the model perform better. Another option is to consider using more advanced models like `GPT-4`, known for better understanding and responding to detailed instructions. However, switching to `GPT-4` could be more expensive, so it's important to think about whether the additional cost is justified by the need for more precise outcomes.

The use of plugins, as shown in the currency conversion, significantly boosts the functionality of our models. This currency plugin is just one example. You can create plugins for various tasks, such as parsing dates, converting measurement units, or even calling external services, which greatly broadens what your applications can do. The ability to add these plugins opens up many possibilities, allowing for the creation of more powerful and adaptive applications. Whether you decide to tweak the prompts for better accuracy or upgrade to a more robust model, these tools offer valuable ways to improve how your system works and how accurately it performs.

## Using multiple models

If, like me you run into scenarios where you want to balance the use of the expensive `GPT-4` to certain functions, you'll need to configure the kernel to include the models you want to use.

First, set up the kernel to handle multiple models:

```csharp
var kernel = Kernel.CreateBuilder()
                   .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                 endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                 apiKey:"API_KEY",
                                                 serviceId:"gpt-35",
                                                 modelId:"gpt-35")
                    .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                 endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                 apiKey:"API_KEY",
                                                 serviceId:"gpt-4",
                                                 modelId:"gpt-4")
                   .Build();
```

In this setup, we add multiple `AddAzureOpenAIChatCompletion` calls with different `modelId` and `serviceId`. This way, the kernel registers different models that you can use for various parts of your application.

To use a specific model with `IChatCompletionService`, pass the serviceId when retrieving the service:

```csharp
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>("gpt-35");
```

When you want to call `InvokePromptAsync`, you can specify which model to use by adjusting the `arguments` with `PromptExecutionSettings`:

```csharp
var arguments = new KernelArguments(new PromptExecutionSettings{ModelId = "gpt-35"}) {{"content", contents}};
```

By setting up your kernel like this, you can easily switch between models as needed, optimizing both performance and cost for different parts of your process. This flexibility allows you to tailor the application's AI capabilities to specific tasks, ensuring you get the best results without unnecessary expense.

## Conclusion

I find myself somewhat torn over using something as powerful and unpredictable as a Generative AI model to parse data. Honestly, I can't think of another method that achieves this level of complexity and capability. If you know of any alternatives, I’d love to hear about them! I have been testing this approach with a variety of unstructured data, and the results have been promising. It just takes this seemingly impossible task and makes it possible. With the parallel processing and plugins, the system was able to handle large volumes of data and perform complex transformations with ease. However, I've encountered some challenges, particularly with hallucinations and inaccuracies when calling functions or plugins. Sometimes it works flawlessly; other times, it doesn't respond as expected, even with identical inputs. This inconsistency leads me to conclude that while this tool is indeed powerful, it isn’t quite ready for autonomous use in production settings without oversight.

To reliably utilize such a system in production, a mechanism to verify the accuracy of the data—like a human-in-the-loop system—would be necessary. Despite these challenges, I'm quite pleased with choosing Microsoft Semantic Kernel. As demonstrated throughout this article, it provides a robust and flexible framework that simplifies the development of AI solutions, equipped with features that make it feel more production-ready compared to the OpenAI SDK.
