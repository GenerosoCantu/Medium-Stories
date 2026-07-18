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

Step 1: Design a new feature. I design features with AI support. My approach is simple: I start by describing the feature in the agent chat, explaining what I want and how it should work.

The agent takes my input and uses the apllications specs to generate the a new feature file, it also knows where to create, how to structure it and what to include. It'll also ask me questions to fill in the missing gaps.

Once a feature file is created, I review it and work with the agent to polish it, add more details or features, until I am 100% satisfied. 
