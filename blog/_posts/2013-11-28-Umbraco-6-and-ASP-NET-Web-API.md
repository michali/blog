---
layout: post
title:  "Umbraco 6 and ASP.NET Web API"
date:   2013-11-28 09:00:13
categories:
- clean code
- rest
---

Umbraco 6.1.0 has introduced support for the ASP .NET Web API. We'll see what kind of support is currently offered, its shortcomings and how we can improve it.

## How Umbraco does it
Controllers that expose an API need to inherit from Umbraco.Web.WebApi.UmbracoApiController. This is a base class that exposes several properties you can use to to work with Umbraco.

By deriving from the base controller you get the following properties:

```csharp
ApplicationContext ApplicationContext { get; }
ServiceContext Services { get; }
DatabaseContext DatabaseContext { get; }
UmbracoHelper Umbraco { get; }
UmbracoContext UmbracoContext { get; }
```
If you had an Event document type and wanted to get an event by ID, you could write a controller like so:

```csharp
public class EventsApiController : UmbracoApiController
{
    public Event Get(int id)
    {
        dynamic eventPublishedContent = Umbraco.Content(id);
        // map the eventPublishedContent to an object of type Event here
        return @event;
    }
}
```

```UmbracoHelper.Content(int id)``` brings back a dynamic object on which one can call properties of the document with that id. For example, speaking of Events, if you had a document type in your Umbraco site called Event and this had a text property called Venue, you would get its value by

```csharp
eventPublishedContent.Venue
```

Or you can cast the dynamic to ```IPublishedContent```.

In either case is then up to us to map this type that describes Umbraco published documents to a custom type so we can return an object specific to the domain served by this RESTful service.

There is a naming convention and that is the API controllers must end with ApiController. For example, a controller that would operate on Conference domain objects would be called ConferencesApiController.

To get an event by routing a request to the above EventsApiController we hit this URL:
```~/Umbraco/Api/EventsApi/Get/1```

If you had a method to get all events:

```public IEnumerable GetAll()```

you would call it with this URL:
~/Umbraco/Api/EventsApi/GetAll

In the [Richardson Maturity Model](http://martinfowler.com/articles/richardsonMaturityModel.html), this would be... level 0 and 2 without being level 1. We're getting a resource or resources using a URL and HTTP verbs but the action method of the API controllers has also got to be in the URL. Action methods for POST, PUT and DELETE. In addition, the name of the resource is obfuscated by the Api suffix. We want the URLs that would access a level 2 REST API to be as follows:

```~/Umbraco/Api/Events/1``` to get the Event document with ID 1
```~/Umbraco/Api/Events``` to get all Event documents

## Creating a (Level 2) REST API with Umbraco support
What's happening here is that because Umbraco uses its own Global.asax, all default MVC routes are overriden with its own. We need to put the Web API default route back into the application so the above URIs get us the resource we want.

You can do this in a Global.asax that inherits from the default Umbraco one. You may already have such a file in your application with all the global logic in it. In this case it's a matter of reintroducing the route registration of the ```WebApiConfig``` static class found in all Web API projects on Global's ```Application_Start()``` and you're good to go.

The other way is to create an Umbraco application event handler and register any custom routes there. For the Web API default routes this could be like the below:

```csharp
public class WebApiRouteRegistrarHandler : IApplicationEventHandler
{
    public void OnApplicationInitialized(UmbracoApplicationBase umbracoApplication, ApplicationContext applicationContext)
    {
    }

    public void OnApplicationStarting(UmbracoApplicationBase umbracoApplication, ApplicationContext applicationContext)
    {
        WebApiConfig.Register(GlobalConfiguration.Configuration);
    }

    public void OnApplicationStarted(UmbracoApplicationBase umbracoApplication, ApplicationContext applicationContext)
    {
    }
}
```

Umbraco picks those handlers automatically so you don't need to register it anywhere.

Now you can write a Web API Controller for your Event documents

```csharp
public class EventsController : ApiController
{
    private readonly IEventsService _eventsService;

    public EventsController(IEventsService eventsService)
    {
        _eventsService = eventsService;
    }

    public Event Get(int id)
    {
        return _eventsService.GetById(id);
    }

    public IEnumerable<Event> GetAll()
    {
        return _eventsService.GetAllEvents();
    }
}
```
You can also change or register a new route if you want the API URIs to begin with ```Umbraco```.

Also instead of inheriting from ```UmbracoApiController``` we're inheriting from Microsoft's ```ApiController```. This results in a few more lines of code but you gain testability as the controller does not expose properties that you (correctly in my opinion) can't set, therefore can't test with. By only exposing what you need you have a controller that doesn't do too much and doesn't depend on properties it doesn't use.