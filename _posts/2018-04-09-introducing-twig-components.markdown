---
layout: post
title:  "Introducing Twig Components"
date:   2018-04-09 08:00:00 -0500
categories: drupal twig
---
Last week I published the [Twig Components Drupal module] - the latest in a
series of projects aiming to combine Twig, Web Components, and PHP. I wanted to
write about why I'm doing this work, and why developers should care.

## Web Components - a Time and Place

[Web Components] are a series of new W3C standards that allows Javascript to
define custom HTML tags. These custom tags, known as [custom elements], differ
from the tags that web frameworks provide because no runtime is required to use
them - the browser is aware of all custom elements and knows how to use them.

It's important to note here that using web standards and "vanilla" Javascript
has no inherit technical value, it's just really exciting to be a part of
something that's new and framework agnostic. Betting on Web Components is
betting on web standards, not *against* any other frontend tool.

That aside aside, the coolest part of custom elements to me is that they move a
lot *back* to the DOM. The API for custom elements is HTML, you use them the
same way you would use any other tag. Asking "Why custom elements" is like
asking "Why `<input>`?". If a custom element is useful, use it!

In my view, Web Components are a great fit for template-heavy CMSes. Frontend
developers are probably already writing HTML templates/CSS/JS, so if anything
Web Components are just a way to codify what you already do. This is not a
revolutionary technology, but I think it's something that bridges the very real
gap between traditional CMSes and fully-fledged frontend frameworks.

I would also say that, at least out of the box, Web Components are not as
ambitious as frontend frameworks, and do not aim to solve the same problems.
Web Components don't come with a solution for routing, state, or context, but
what they do solve is componentizing HTML/CSS/JS in something that acts like
any other native element. It's worth nothing that people *have* built entire
web applications with Web Components, but I don't think that's the
[80% use case].

## Discovering Universal Twig

If you're a Symfony or Drupal 8 developer, you probably have seen or used Twig,
a PHP template engine designed to be fast and flexible. I like writing Twig,
and from what I can tell most of the Drupal community does as well. Some like
it so much that they've created massive Atomic Design based themes that already
include dozens of re-usable, componentized Twig templates.

I knew going into Web Components that I wanted to use Twig on the backend and
the frontend - both for my own preference and to ease PHP developers into this
uncharted territory.

Luckily for me, some generous folks had already ported Twig to Javascript with
the appropriately named [Twig.js] project. This opened the door for me to write
[Twig Components, an NPM project] that provided a base vanilla Web Component
that knew how to handle Twig. I later created [an example project] and a
[project template] that included polyfills, test coverage, template
pre-compiling and ES5 transpiler.

All the Javascript was coming into place, but I realized somewhere down the
line that I could feasibly pre-compile these Web Components server-side with
PHP. Normally for something like this you would use [V8JS], which allows you to
execute Javascript in PHP, but most hosting providers don't have this extension
installed and asking users to compile an extension to use Twig Components
seemed like a bad developer experience. In practice I've also noticed that
because V8JS isn't exactly like the Node environment, it can be unruly to get
things running.

So I set out to server-side render Web Components in PHP, and it turned out
pretty well. My Composer project, [mortenson/twig-components-ssr], now supports
nested components, `<style>` tags in templates, and the `<slot>` element. At
this point, I felt like I could push forward on a Drupal integration.

## The end of the funnel

Making Twig Components work in Drupal was always the end goal, and I knew that
for the integration to work I had to make a module that provided more value
than just getting the right Javascript on the page.

The first thing I added to the module is a new plugin annotation for
components, which could be used in a class or a YML file. Using the plugin
system was a little tough, but worth it since it gave me discoverability,
cacheability, and extensibility. I based a ton of the code and tests on the
Layout component and Layout Discovery module from Drupal core, so it may look
eerily familiar.

Once plugins were in place, I started working on an event subscriber that
responds to every HTML response in Drupal, pre-renders all components, and
ensures that only the minimally required libraries are present on the page. You
can [read the documentation for the renderer] for more details.

With a way for modules and themes to provide Twig Components, and assurance
that server-side rendering and dynamic library addition works, I released the
module on Drupal.org.

At this point, I just need users to create *real* things with it before I know
what direction to take with the module. If anyone is interested in diving in
and trying this out, you can [jump into the documentation on Drupal.org here].

Enjoy!

[Twig Components Drupal module]: https://www.drupal.org/project/twig_components
[Web Components]: https://github.com/w3c/webcomponents
[custom elements]: https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements
[80% use case]: https://en.wikipedia.org/wiki/Pareto_principle
[Twig Components, an NPM project]: https://www.npmjs.com/package/twig-components
[an example project]: https://github.com/mortenson/twig-components-example
[project template]: https://github.com/mortenson/generator-twig-components-webpack
[mortenson/twig-components-ssr]: https://github.com/mortenson/twig-components-ssr
[read the documentation for the renderer]: https://www.drupal.org/docs/8/modules/twig-components/the-twig-component-render-pipeline
[jump into the documentation on Drupal.org here]: https://www.drupal.org/docs/8/modules/twig-components/building-your-first-twig-component
[Twig.js]: https://github.com/twigjs/twig.js
[V8JS]: https://github.com/phpv8/v8js
