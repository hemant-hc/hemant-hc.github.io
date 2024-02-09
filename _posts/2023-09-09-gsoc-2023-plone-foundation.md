---
layout: post
title: "Building a RestAPI Client with TypeScript and TanStack Query for Plone 6 [GSoC'23]"
---

Recently, I embarked on a project that aimed to create a RESTful API client written in TypeScript, leveraging the power of TanStack Query, to interact with a Plone 6 backend. This project not only addressed the need for modernizing data fetching but also made it versatile for use in various front-end frameworks like React, Vue, Solid, and Svelte. In this blog post, I'll share my journey and the key highlights of this project.

Community Benefits

One of the primary motivations behind this project was to discontinue the use of the deprecated AsyncConnect library and transition to TanStack Query for server state handling. By doing so, we aimed to improve data fetching capabilities and streamline the development process. This change not only benefits our project but also contributes to the wider community by promoting best practices in web development.

Example Usage

Here's a look at how @plone/client library can be utilized in Volto:

```typescript
import ploneClient from "@plone/client";

const client = ploneClient.initialize({
  apiPath: "http://localhost:8080/Plone",
});

const { useGetContent } = client;

const { data, isLoading } = useGetContent({ path: pathname });
```

This code snippet showcases the use of the `useGetContent` action hook in Volto, a react based frontend for plone.

The framework-agnostic approach to ensures that the client is versatile and compatible with a wide range of front-end technologies. While it naturally supports React and Next.js, other frameworks can utilize the query creator function with the appropriate version of TanStack Query.

Challenges Faced and Resolutions

Throughout this project, I encountered a series of challenges. For instance, transitioning from legacy Volto code to axios for API requests ensured compatibility with various frameworks. Testing complications were addressed by introducing a new test configuration, ensuring reliable results. And when it came to custom hooks, a utility function made the process significantly more efficient.

Accessing Project Contributions

If you're interested in exploring the project contributions or diving into the technical details, you can access them [here](link).

Thanks to Plone Foundation and Google for providing me with this wonderful opportunity.
