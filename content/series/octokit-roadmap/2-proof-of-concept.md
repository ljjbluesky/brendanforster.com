---
layout: post
title: "Proof of Concept"
slug: "proof-of-concept"
description: "I had an idea, and I wanted to see how quickly I could prove or disprove the idea was worth exploring further."
weight: 2
---

I won't share the code I used to prove this out because it was disorganized and
a bit of a mess, but I was able to generate a slice of the API as C# code that
almost compiled except that I'd omitted requests and response types:

```
Generating file MarketplaceListingAccountsClient.cs
--------
using System.Collections.Generic;
using System.Net.Http;
using System.Threading.Tasks;

namespace Octokit
{
    public interface IMarketplaceListingAccountsClient
    {
        Task<MarketplaceListingAccountsResponse> Get(string accept, int account_id);
    }

    public class MarketplaceListingAccountsClient : ApiClient, IMarketplaceListingAccountsClient
    {
        public MarketplaceListingAccountsClient(HttpClient httpClient)
        {
        }

        public Task<MarketplaceListingAccountsResponse> Get(string accept, int account_id)
        {
            // TODO: setup the request and url
            // TODO: invoke GET /marketplace_listing/accounts/{account_id} on HTTP client
            // TODO: parse response and return result to client
            throw new NotImplementedException();
        }
    }
}
--------

Generating file MarketplaceListingClient.cs
--------
using System.Collections.Generic;
using System.Net.Http;
using System.Threading.Tasks;

namespace Octokit
{
    public interface IMarketplaceListingClient
    {
        IMarketplaceListingAccountsClient Accounts
        {
            get;
        }

        IMarketplaceListingPlansClient Plans
        {
            get;
        }

        IMarketplaceListingStubbedClient Stubbed
        {
            get;
        }
    }

    public class MarketplaceListingClient : ApiClient, IMarketplaceListingClient
    {
        public MarketplaceListingClient(HttpClient httpClient)
        {
            Accounts = new MarketplaceListingAccountsClient(httpClient);
            Plans = new MarketplaceListingPlansClient(httpClient);
            Stubbed = new MarketplaceListingStubbedClient(httpClient);
        }

        public IMarketplaceListingAccountsClient Accounts
        {
            get;
            private set;
        }

        public IMarketplaceListingPlansClient Plans
        {
            get;
            private set;
        }

        public IMarketplaceListingStubbedClient Stubbed
        {
            get;
            private set;
        }
    }
}
--------

...
```

If you're not familiar with the internals of Octokit.net, these two files help
to illustrate the basics of what we need to generate:

 - clients contain methods that correspond to an action the API supports
 - clients can also contain child clients (as properties) to group related
   parts of the API

This was a great little exercise to refresh my memory on how things are organized
in the codebase, and I ended up doing things a bit differently for this proof of
concept.

## Organizing the client API

This is the first place where I diverged from how Octokit is organized.
Currently we organize clients based on how they are listed on the
[developer documentation](https://developer.github.com/v3/) which worked up to
a point, but this sample used the paths as a source of truth.

To use the example above:

 - For the API route `/marketplace_listing/accounts/{account_id}`...
 - The corresponding client is named `IMarketplaceListingAccountsClient` in the
   codebase...
 - Which the caller can access this at `client.MarketplaceListing.Acccounts.Get(account_id)`

I'm going to keep following this convention for new routes, and I suspect this
will introduce some breaking changes down the track when I need to start to
generate code for existing routes. For example, `/users/*` and `/user/*` are
routes that are both normalized to `client.User.*` in Octokit.net currently.

## Organizing the source

Another important thing is the interface and implementation being in the same
file. This is different to how Octokit.net is currently structured, with
separate code files for the interface, implementation and request/response
models. Some might consider this an anti-pattern for typical .NET development
(I can already hear the mob chanting "one class per file!") but I believe
keeping related things within the same file will make reviews for API changes
much easier, rather than having to know which files relate to eachother. I also
plan to include the request and response models in the same file, but I didn't
get that far with the proof of concept.

## Client internals

You may notice that the constructor accepts a `HttpClient` but doesn't use it
currently. This is also different from the current Octokit.net behaviour (where
`ApiConnection` is the constructor parameter). I'll switch back to that for the
actual implementation, but I'd love to get to the point where we can remove
our internal abstractions with components that exist for some parts, in favour
of using built-in features:

 - making HTTP requests based on the required verb and path
 - parsing JSON response into C# objects and vice versa

`HttpClient` is already used inside Octokit.net, but I'd love to make it's use
more explicit rather than burying it in a bunch of abstractions. We also need
to be mindful of GitHub Enterprise environments, as these routes require a 
prefix to the path (`/api/v3`) when being invoked. 

The `System.Text.Json.JsonSerializer` namespace (only available since .NET Core
3.0) could be a replacement for our `SimpleJson`-based parsing, but I think we
can survive with that currently because it's being built from source for our use
case and has a lot of conventions and opinions that would also need to be
ported.

Maybe this will be worth discussing when we have a case to make for dropping
support for .NET Framework completely.
