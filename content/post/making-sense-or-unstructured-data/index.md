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

Curious to see how it all worked out? Stick around, and I'll walk you through it. All the code samples are available in this [GitHub repository](https://github.com/droosma/making-sense-or-unstructured-data).

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

var kernel = Kernel.CreateBuilder()
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

var result = await kernel.InvokePromptAsync($"""
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
Console.WriteLine(string.Join($"{Environment.NewLine}===={Environment.NewLine}", listings));
Console.ReadLine();
```

As you can see, we added a bit more structure to the prompt. First moved the file contents from the string interpolation to a argument. We also added more groundings to the prompt, making sure the output is something we can automatically parse, which we do with a `System.Text.Json.JsonSerializer`. There are of course more ways to structure the output, but I have had a lot of success with this `json` approach in the past. But I encourage you to experiment and find what works best for you.

After running the code, the output should look something like this:

```plaintext
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

Nice, we now have a list of car listings, each in a separate string. You might be wondering why I'm introducing this intermediate step, and the reason is simple. I expect the parsing of the individual car listings to be a bit more complex, and when you limit the context for an LLM like this it reduces the chance of the model taking a wrong path and start hallucinating. But there are a few other benefits. By splitting up the single large task into small single listings we can also parallelize the parsing of the individual car listings, making the process faster. The final benefit is that it's pretty much impossible to hit the token limit.

So lets implement the parsing of a individual car listing. The process is almost exactly the same as splitting the file, but we need to ground our prompt a bit more to make sure we get the information we need.

```csharp
private async Task<Listing> ExtractListing(string contents)
{
    var arguments = new KernelArguments { { "content", contents } };
    var result = await kernel.InvokePromptAsync("""
                                                You are tasked with converting a car listing into a structured JSON format below is the text content from a file:
                                                ```
                                                {{$content}}
                                                ```

                                                ### Tasks:

                                                1. Ensure no information is omitted. Include all text as it appears in the file.
                                                2. Produce the output in valid JSON format. The output must be directly parsable into an Listing object.

                                                ### Sample Output:
                                                {
                                                    "Make":"Toyota",
                                                    "Model":"highlux",
                                                    "Odometer":"100000",
                                                    "ManufacturerDate":"2000-05-29",
                                                    "Price":"16900",
                                                    "Contact":"contact@example.com"
                                                }
                                                """,
                                                arguments);
    var response = result.ToString();
    return JsonSerializer.Deserialize<Listing>(response);
}
```

As we want to call this prompt for each individual car listing, I created a wrapper function that takes the contents of a single car listing and returns a `Listing` object. The `Listing` object is a simple record that represents the structured data we want to extract from the car listing. The `Listing` record should look something like this:

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

The entire execution of the program now looks like this:

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

In the code we first call the `ExtractListings` function to split the file into individual car listings. We then use `AsParallel` to parallelize the extraction of the structured data from each car listing. Finally, we use `Task.WhenAll` to wait for all the tasks to complete and then print the structured data to the console. Running the code should give you an output similar to this:

```plaintext
BMW 3 Series [60,000/2017-03] - ¥2,000,000 | 123-456-7890
Ford Mustang [80000/2018-05] - 15000 |
Honda Civic [150000/2010-12] - 8000 | John@example456.email.com
Mercedes-Benz C-Class [120,000 km/January 2019] - ?20,00,000 | contact@example.com
Toyota Camry [100,000 miles/October 2015] - $12,500 | example123@email.com
```

One last step, as we iliminated the rist of hitting the token limit for the second `ExtractListing` method by only sending a single car listing to the model, we need to make sure we don't hit the token limit for the first `ExtractListings` method. We can do this by splitting the file into chunks and sending each chunk to the model. This is a simple process, but it does require a bit of code. Again there are many ways to do this, but as an example I will show you how I did it.

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

The `GetChunks` function takes the file contents and splits it into chunks of a specified size. The `chunkSize` parameter determines the number of lines in each chunk, while the `overlapLines` parameter determines the number of overlapping lines between chunks. This function returns an `IEnumerable<string>` of the chunks. I'm using overlapping chunks to make sure we don't miss any information when splitting the file as we are just splitting on newlines, which can be a bit risky as we can split a car listing in half. this way we make sure we don't miss any information at the cost of possible duplicate listings.

The final implementation of the `Execute` function now looks like this:

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

First we split the file into chunks using the `GetChunks` function. We then use `AsParallel` to parallelize the extraction of the individual car listings from each chunk. Finally, we use `Task.WhenAll` to wait for all the tasks to complete and then print the structured data to the console. Running this should give you the same output as before, but now we are minimizing the risk of running into the token limit.

And that's it, we have successfully parsed the unstructured data into structured data. We can leave the rest of the processing to more traditional mechanisms. But if I have peaked your interest in to what else you can do with Microsoft Semantic Kernel, let's take a look at a few more advanced features.

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