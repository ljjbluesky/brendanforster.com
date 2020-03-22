---
layout: post
title: "Uncovering a design"
slug: "uncovering-a-design"
description: "The proof of concept uncovered some promising features, but it's
often best to throw away that first system and start fresh. So I did that, but
also took time to write up my thoughts before diving back into the code."
weight: 4
---

### The Components

This whole experiment made me realize there were discrete components that could
be worked on independent of eachother, once the fundamentals were in place, that
could represent what the tooling required:

 - `PathProcessor` - responsible for scanning the schema document and extracting
   the parts relevant to what a client needs
 - `PathFilter` - exclude paths that we don't want to generate code for - very
   helpful initially to simplify development, with a goal that we'd be
   eventually be generating the entire schema
 - `ApiBuilder` - take the schema information and translate it into the
   resulting API metadata, based on conventions that the codebase expects
 - `RoslynGenerator` - take the API metadata, generate the relevant C# code,
   write the code to disk

The input - in this case our schema definition - would flow through each
component until we got source code as the output of the last component. Those
would be written to disk, and then handled like our existing code - with unit
tests, convention tests, and even integration tests.

The conventions themselves are likely the biggest part of this work, because
they need to cover all the ways we translate between the GitHub API and
idiomatic .NET code:

 - naming things - the interfaces, classes, methods, properties and types
   associated with each route
 - conversion - handling deserializing and serializing between JSON and C# objects
 - handling output - status codes and payloads from an endpoint represent
   something - can we ensure callers get this information in a format that makes
   sense?

Some of these features is already present in the library - codifying it and
moving it to something that's usable as part of code generation is what we would
aim for here.
