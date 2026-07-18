# SpecHub Demo Script

Hi, and welcome. In this demo, I'll walk you through the five-step loop behind the SpecHub AI workflow.

Before we start, what you're seeing in Visual Studio Code is the SpecHub repository. It contains the specs and context documents for the entire application and all of its services.

As I explained in the article, the AI agent already knows these five steps and how to follow them. It also knows where to find context for the app and all of its services.

CLAUDE.md includes instructions that are automatically loaded into each new Claude session. If you're using Copilot, you can use Copilot instructions instead, or an equivalent setup for another AI tool.

This instruction file links to documents that contain context for the application and each service. For example, the architecture overview file covers the whole application, while the Main API file contains context for one microservice.

These files do not contain code. They are compressed context documents for each service in the application.

In this demo, I'm using a personal project to showcase the SpecHub workflow. The project is a news website CMS, and I'll create a new feature that modifies multiple microservices and frontends.

[open Public Frontend]

This feature adds a block that displays the top five news stories on the website. To support this, I need to create a new analytics microservice. A script on the Public Frontend will trigger requests to the Analytics service and capture section and story views.

The Top News component will read its data from a static JSON file on the CDN.

The Analytics microservice will receive the data, save it to the database, and generate the static JSON file on the CDN.

[open Webapp Frontend]

I will also create a new section on the Webapp Frontend to display analytics data. This will require changes to the Main API.

[open Control Center]
In the Control Center web app, I will add an analytics URL and a tenant time zone. This time zone will be used to calculate the start of the day for each tenant. This will also require changes to the Tenant API.

Now let's begin the workflow.

Step 1: DESIGN a new feature. I design features with AI support. My approach is simple: I start by describing the feature in the agent chat, explaining what I want and how it should work.

The agent takes my input and uses the application specs to generate a new feature file. It also knows where to create it, how to structure it, and what to include. It will ask follow-up questions to fill in any missing gaps.

Once the feature file is created, I review it with the agent and keep refining it until I am 100% satisfied. One thing worth mentioning is that I never write directly in the feature file. I work through chat, and most of the time I just dictate.

You can see the finished feature with all its details. Even before prompts are generated, the workflow already determines how many prompts are needed and recommends the best model for each one. This is a large feature, and I could have split it into three or four smaller features. But to better showcase the SpecHub workflow, I kept it together.

From here, the remaining steps are mostly execution. There are not many decisions left to make; it is mainly about following the workflow.

Step 2. CASCADE, Write the decisions into the affected service spec(s).

ON THE AGENT DICTATE: cascade this feacture into the affected services. 

Step two is done. You can see how it affected the spec files for multiple services. Plus, it also created a new one for the Analytics service. 

Step 3. PROMPT, Generate Prompts. Always use a fresh session for this.

ON THE AGENT DICTATE: create all the prompts for this feature. 

Step three is done. These are all the prompts that got generated. All prompts follow a template. It varies a bit whether it's a backend or frontend prompt. As you can see, it includes certain things. For instance, study these three files and generate these two files. This makes the prompt more efficient because the agent will not need to file all the hundreds of files on the service. 

Step 4. IMPLEMENT, Apply each prompt inside its service repo. 

This is mainly just copying the prompt and pasting it on the agent of each service.

Step 5. CLOSE, Verify specs against the code, reconcile, log, archive.

ON THE AGENT DICTATE: all prompts for this feature have been implemented and verified. Close the feature.



