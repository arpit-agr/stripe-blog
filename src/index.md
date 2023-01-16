---
title: "How Stripe builds interactive docs with Markdoc"
date: 2022-09-13
author:
  name: "Ryan Paul"
  role: "Docs Product"
  img: "https://images.ctfassets.net/fzn2n1nzq965/3d1Af1q4jrKmklAZ1YUrws/8c00045f0af912268fc4b089a3a30075/rpaul.jpeg.png?w=96&amp;h=96"

layout: "layouts/home.html"
---

At Stripe, [our product docs](https://stripe.com/docs) are designed to feel like an application rather than a traditional user manual. For example, we incorporate a user's own API test key into code samples, making it possible to copy and paste code that seamlessly works with the user's own account. We have client-side interactivity, like checklists and collapsible sections. We tailor the content to the individual user, conditionally displaying content based on their location or the Stripe features they use. These features result in a high-quality user experience that reduces friction and contributes to the success of developers.

For these capabilities to have the desired impact we have to make it easy for writers to use them in their content. Delivering a good user experience without compromising the authoring experience required us to develop an authoring format that enables writers to express interactivity and simple page logic without mixing code and content.

Over several years, we learned how to balance interactivity, customization, and authoring productivity while undertaking a major overhaul of our documentation platform.

## Past is prologue

To understand how we got here it's important to understand where we started. The legacy documentation platform that we replaced was a monolithic Ruby application built with [ERB templates](https://github.com/ruby/erb/blob/master/README.md) and Sinatra routing. The content freely mixed HTML, Markdown, Ruby, and ERB helper functions.

Mixing code and content provided a natural way to programmatically tailor the docs to the developer, but it posed serious challenges to quality and maintainability when the body of content grew to hundreds of pages. Alongside the technical burden of maintaining code within the content, the behavior of the code can make the content itself harder to understand and manipulate safely, particularly when used by many different teams with different objectives, timetables, and areas of expertise. Content authoring effectively became software development, and with that became subject to the same technical complexity and overhead.

We wanted to introduce new content formats with significantly more interactivity and more sophisticated frontend logic, but we knew that the limitations of a code-first approach would prevent us from using these features widely. For example, our [integration builder](https://stripe.com/docs/payments/quickstart) format, which was originally created as a React application with content authored in JSON, became much easier for technical writers to reproduce and maintain when it was migrated to use Markdoc for the authoring experience.

## Designing Markdoc

When we began building our current documentation platform, we wanted to simplify our authoring experience by adopting an intuitive format like Markdown. Although Markdown is significantly easier to read and reason about than ERB templates, its simplicity also imposes limitations that make it challenging to use for rich content like our product docs.

Markdown is a relatively flat format that isn't designed to express complex structure or hierarchy. It offers a small number of formatting features and provides limited control over presentation. It does not have exotic templating features like support for custom page logic, variables, conditionals, or content reuse. Markdown's enduring success and relative ubiquity are largely due to its intentionally narrow scope and the restraint exercised in its design. It is easy and enjoyable to use because it prioritizes readability and leans heavily on intuitive plain-text authoring conventions.

Our custom authoring format, called Markdoc, was designed to decouple code and content while enforcing proper discipline at the boundaries. Instead of allowing each page to be treated like an open-ended application, it imposes constraints on styling and programming, providing prescriptive rails for content extensibility. It extends Markdown with custom syntax that meets the needs of our documentation platform without sacrificing Markdown's simplicity, familiarity, or ease of use for writing prose. Following the ethos and design sensibility of Markdown, Markdoc keeps the overall surface area of new features small by adding a few highly-composable primitives that can be used together to express all the functionality we need.

Markdoc provides an extensible system for defining custom tags that can be used seamlessly in Markdown content. Using the custom tag syntax, we're able to support features like conditional content, content inclusion, and variable interpolation.

{% raw %}

```
# This is a heading {% #section %}

{% callout type="info" %}
This is a paragraph with _formatted_ text inside the callout
{% /callout %}

{% if $condition %}
This content shows if `$condition` is true.
{% /if %}
```

{% endraw %}

The features we decided to leave out of Markdoc in order to protect content maintainability are a critical aspect of its design. For example, when deciding what built-in flow control to include in Markdoc, we deliberately chose not to include looping. We wanted to discourage writers from performing procedural content generation from inside of a document, forcing them to move it outside of the system for better encapsulation. We also decided to leave out variable assignment in order to ensure that the content is fully stateless, thus eliminating an entire class of potential bugs.

> I like Markdoc because it lets us still do anything we want with code in the docs without bogging down the content authoring experience. If we need some new component, designers and engineers can whip that up. So as a writer, I can work in the docs content and stay focused.
>
> <cite>**Lucie Lozinski**, Technical Writer</cite>

## React integration

Markdoc has a modular rendering system that supports multiple output formats. Using Markdoc’s React renderer, a Markdoc document can be rendered to a React virtual DOM. Custom Markdoc tags can be configured to output React components, passing through tag attributes as React props. Markdoc also supports assigning custom React components to standard Markdown document nodes such as headings and paragraphs.

Defining custom Markdoc tags that output React components makes it possible to include interactive features, like tab switchers and collapsible content sections, inside of documents. Using custom tags to express these features helps create a writer-friendly interface for the functionality.

The React ecosystem also has a wealth of useful and interesting libraries that we can incorporate into our documentation to enrich presentation. For example, we're using the React Flow library to create interactive diagrams in our documentation. We defined a set of Markdoc tags for expressing the contents of a diagram, making it easy for writers to build beautiful and consistent visual representations of APIs and technical concepts from a set of composable elements.

![Tux, the Linux mascot](https://images.ctfassets.net/fzn2n1nzq965/3nSXx3uQbRq8fwq7PxDuw2/3eaa05604f335eb61c846848b61c6135/markdoc-graph-remini-enhanced.png?w=1620&q=80&fm=webp)

{% raw %}

```
{% diagram type="sequence" description="Usage-based billing" %}
  {% node #customer icon="customer" %}
  Customer
  {% /node %}
  {% node #typographic icon="platform" %}
  Typographic
  {% /node %}
  {% node #stripe icon="stripe" %}
  Stripe
  {% /node %}
  {% edge from="customer" to="typographic" %}
  Select plan
  {% /edge %}
  {% edge from="typographic" to="typographic" %}
  Create [usage records](/docs/api/usage_records)
  {% /edge %}
```

{% endraw %}

Unlike static images, the diagrams that are built with React Flow can easily incorporate interactivity and clickable links. They are also easier to localize and can be restyled universally.

Moving our documentation frontend to React was an important goal of our platform overhaul. Stripe already used React across many parts of the user experience, including our API reference docs and user dashboard. Enabling integration and cross-pollination between those surfaces and the product docs opens up a lot of exciting opportunities for future innovation, like showing API reference overlays when the reader hovers their cursor over a function or parameter in a code example. Sharing a common set of components from Stripe's internal design pattern library helps improve cohesion.

React also offers some compelling technical advantages. The implementation of interactive frontend components in our legacy stack was split between markup ERB templates and logic written in JavaScript which made it difficult to properly encapsulate, extend, and reuse functionality—a set of problems that modern component-based frontend frameworks address in a more satisfying way.

Markdoc comes with two distinct React renderers: a renderer that dynamically builds the React virtual DOM tree on the client side, and a static renderer that transpiles the document to JavaScript code. We use the dynamic renderer in our documentation platform at Stripe, but the static renderer is useful in cases where you want to treat a piece of Markdoc content as though it is a React component or JavaScript module. For example, the static renderer makes it possible to implement a Markdoc loader for Webpack in only five lines of code.
