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

Hey there! If you've been following my blog, you know how fascinated I am with Generative AI and its growing applications. Recently, I stumbled upon a tricky problem: parsing unstructured data automatically. Despite my best efforts to get cleaner data from the source, I had to find another solution. That’s when Generative AI came to mind.

In this post, I’ll take you through my journey of creating a proof of concept to tackle this challenge. I've dabbled with various Generative AI frameworks in my spare time. Although the OpenAI SDK impressed me, I wanted something more robust for production. Enter Microsoft Semantic Kernel—a tool that seems a lot more ready for the big leagues. So, I decided to give it a shot for this project.

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

By executing this code, you can achieve the same detailed output as before while effectively managing larger data volumes. We have successfully transformed unstructured data into structured data, ready for further processing or analysis using more traditional methods. If you are curious about the additional capabilities of Microsoft's Semantic Kernel, let's explore some advanced features next.

## Creating and calling functions

As indicated earlier, there is still some work to be done to make the structured data more usable. For example, we might want to convert the price of each car listing to a common currency. Or we want to parse the dates to `Date` objects and instead of having the odometer be in different units we might only want a integer in kilometers. We can do this by creating a function that takes the price and currency as input and returns the price in the desired currency. Or take the odometer part and convert it to kilometer integer. We than configure the LLM model to call these models when it assess they are required.

As far as I have been able to find, there is no way to allow the model to call a function through the same way we have been calling the model until now. So we are going to have to switch to using the `IChatCompletionService` that comes out of the box with Semantic Kernel. This service is designed to abstract the intricacies of creating a chat application, but we can use it a little bit differently.

```csharp
using Microsoft.SemanticKernel;

var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                        endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey:"API_KEY");
var kernel = kernelBuilder.Build();

var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();
```

Same as before we create the kernel builder and add the Azure OpenAI chat completion service to it. But this time we also get the `IChatCompletionService` from the kernel.
Next we implement the `ExtractListing` in the new way

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

As we are now using the ChatCompletionService, we need to create a `ChatHistory` object that contains conversation history. As I said earlier, the `IChatCompletionService` is an abstraction designed for use with a chat bot like [`ChatGPT`](https://chat.openai.com/). In the history you can specify who said what and by chosing between `AuthorRole.User` The user, or `AuthorRole.Assistant` the bot. You usually start off the converstation, as I have done here with a `AuthorRole.System` which tells the bot how to behave. So I changed the original prompt to be a bit to expect the user to input the car listing. The rest of the code is the same as before.

Ok, so now we have what we had before, but a bit more complex. Let's get to why we changed it, as Semantic kernel calls them `Plugins`. Let's start with the currency conversion.

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

The `CurrencyPlugin` class contains a single method, `ConvertToDollar`, that takes a currency code and an amount and returns a random amount in USD. The method is decorated with the `KernelFunction` attribute, which tells the Semantic Kernel that this method is a plugin function. The `Description` attribute is used to provide a description of the method and its parameters. The method returns a `decimal` value, which is the dollar equivalent of the provided currency amount. For the sake of this example, I'm just returning a random value, but you can replace this with a real currency conversion service.

Next we need to register the plugin with the kernel. We do this by adding the `CurrencyPlugin` class to the kernel builder.

```csharp
var kernelBuilder = Kernel.CreateBuilder()
                          .AddAzureOpenAIChatCompletion(deploymentName:"AZURE_OPENAI_MODEL_NAME",
                                                        endpoint:"https://DEPLOYMENT_NAME.openai.azure.com/",
                                                        apiKey:"API_KEY");
kernelBuilder.Plugins.AddFromType<CurrencyPlugin>();
var kernel = kernelBuilder.Build();
```

And we change the `IChatCompletionService` call a bit to allow the model to call the plugin.

```csharp
var settings = new OpenAIPromptExecutionSettings
               {
                   ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
               };
var result = await _completionService.GetChatMessageContentAsync(history, settings, kernel);
```

The `OpenAIPromptExecutionSettings` object is used to configure the behavior of the tool when calling kernel functions. In this case, we set the `ToolCallBehavior` property to `AutoInvokeKernelFunctions`, which tells the tool to automatically call kernel functions when needed. Running the code now should give you the same output as before, but now the price of each car listing is a random value in USD.

from the original listing test:

```plaintext
I'm selling my beloved Toyota Camry. It's a fantastic car with only 100,000 miles on it. Manufactured back in October 2015. I'm looking to get € 12,500 for it. Let me know if you're interested!

Contact me at: example123@email.com
```

We now get the following output:

```plaintext
{
    "Make":"Toyota",
    "Model":"Camry",
    "Odometer":"100000",
    "ManufacturerDate":"2015-10-01",
    "Price": 6423.30,
    "Contact":"example123@email.com"
}
```

Something I have noticed is that the model is not always able to correctly do what the System prompts is asking it to do. So you might need to add some more grounding to the prompt to make sure the model is able to do what you want it to do. An alternative is to switch models, as I have noticed that the `GPT-4` model is a lot better at following the instructions given to it. But as I mentioned earlier, the `GPT-4` model is a lot more expensive, so you will need to weigh the cost against the benefit.

As you can see using `Plugins` is a powerful way to extend the functionality of the model. The price conversion is a simple example, but you can use plugins to do more complex operations like date parsing, unit conversion, or even calling external services. The possibilities are endless, and it's a great way to make your models more powerful and flexible.

## Using multiple models

If, as indicated above you want to use different models for different parts of the process, you will have to configure the kernel to allow for this.

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

By adding multiple `AddAzureOpenAIChatCompletion` calls with a different `modelId` and `serviceId` you can configure the kernel to register different models which you can than use for different parts of the process.

For the `IChatCompletionService` you can pass the `serviceId` to the `GetRequiredService` get the service like this:

```csharp
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>("gpt-35");
```

And for the `InvokePromptAsync` method you can adjust the `arguments` with `PromptExecutionSettings` where you can specify the `ModelId` like this:

```csharp
var arguments = new KernelArguments(new PromptExecutionSettings{ModelId = "gpt-35"}) {{"content", contents}};
```

## Conclusion

I'm still a little split about using something as powerful and unpredictable as a Generative AI model to parse some data. But for the life of me, I really can't think of another way to achieve this. If you do please let me know! As for my choice of using Microsoft Semantic Kernel, I'm really happy with it. As I hope I have shown in this article, it is a powerful and flexible framework that makes it easy to build AI solutions with baked in amenities that make it feel a lot more production ready than the OpenAI SDK.