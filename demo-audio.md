# SpecHub Demo text

Hi, and welcome. In this demo, I'll walk you through the five-step loop behind the SpecHub AI workflow.

As I explained in the article, the AI agent already knows these five steps and how to follow them. It also knows where to find the context for the app and for all of its services.

CLAUDE.md has all the instructions that are going to be automatically loaded into a new Claude session. This could also be replaced by Copilot instructions if you're using Copilot, or a similar thing if you're using any other AI. 

This instructions file contains links to the files that have the context to the application and all its services. For instance, architecture overview file contains the overview for the whole application while the Main API file contains the context to one of the microservices. 

These files do not contain code. These files are the compressed contexts for each of the services in the application. 

In this demo I'm going to be using a personal project to showcase the SpecHub workflow. The project is a news website CMS, and I will create a new feature that will modify multiple microservices and frontends. 

[open Public Frotend]

This feature is about adding a block that will display the top five top news stories for the website. For this, I need to create a new analytics microservice. There will be a script on this Public Frontend that will trigger request to the Analytics service, and it'll capture the sections and the story views. 

The top news component will be getting its data from a static JSON file from the CDN. 

The Analytics microservice will be catching the data, saving it into the database, and generating the static JSON file to the CDN. 

[open Webapp Frotend]

I will also create a new section on the Webapp Frontend to display the analytics data. These will require changes on our main API. 

[open Control Center]
On the Control Center webapp, I will add an analytics URL as well as a time zone for the tenant. This time zone will be used to calculate start of the day for each tenant. These will also require changes to the tenant API. 


So let's get to it and start with the workflow. 

Step 1: Design a new feature. Features are designed with help from the AI. The way I do it is simple. I start by talking through the feature into the agent's chat, explaining what I want and how it should work. 
