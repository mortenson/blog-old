---
layout: post
title:  "Simplifying Drupal frontend with Single File Components"
date:   2019-10-06 08:00:00 -0800
category: drupal
---
I’ve been thinking about ways to make Drupal frontend easier recently, and have
been working on an experimental module called Single File Components (SFC),
which lets you put your CSS, JS, Twig, and PHP in one file. If you want to skip
the blog (😭) you can just check out the project at
[https://www.drupal.org/project/sfc].

The main problems with Drupal frontend SFC aims to help with are:

1. Organizing Drupal frontend code is hard.
1. Splitting your CSS/JS into separate files and writing `*.libraries.yml` entries
is tedious, especially with small files. Many themes have a single CSS/JS
bundle for this reason.
1. Writing JS in the "correct" Drupal way is surprisingly hard, and rare to
find even in popular contributed themes and modules.
1. When Twig isn’t enough and you need PHP for something, it’s difficult to
figure out where that goes and it’s often unclear that something like a
preprocess is required for a Twig template to work.
1. Testing frontend Twig and PHP is hard.
1. Quickly providing custom Blocks, Layouts, and Field Formatters that use Twig
templates is hard.

It’s important to note that there are many other component-y solutions out
there for Drupal, and most help with the same problems. Projects like Pattern
Lab and Component Libraries are widely used and enjoyed by many, so please use
them over SFC if they feel better to you! Open source isn’t a competition but
with frontend it can often feel that way. Phew, glad that’s covered.

Let’s get into it and start building a component. There are two ways to build
components with SFC, but we’ll start with the simplest, which is to make a
`.sfc` file in the "components" directory of any enabled module or theme. The
component we’re building says "Hello" to users, so we’ll need some Twig that
takes the name of the user, and some CSS to uniquely style it.

Here’s `say_hello.sfc`:

```html
<style>
  .say-hello {
    font-size: 20px;
  }
</style>

<template>
  <p class="say-hello">Hello {{ name }}!</p>
</template>
```

After a cache rebuild, you can now use this component anywhere by including it
in a Twig template:

```twig
{% raw %}{% include "sfc--say-hello.html.twig" with {"name": "Sam"} %}{% endraw %}
```

Pretty cool right? There’s obviously a lot happening here that I’ve abstracted
away from users, so here’s my quick rundown:

- `*.sfc` files are used by a service that derives SingleFileComponent plugins.
- A library definition is automatically generated because the component includes
CSS.
- A custom Twig loader will create instances of SingleFileComponent plugins
when they’re rendered and use their templates.
- An `{% raw %}{{ attach_library(...) }}{% endraw %}` call is automatically prepended to the
template.
- When assets for the library are collected, the component’s CSS is output to a
separate file.

The goal here is to make the end users, frontend developers, unaware of all the
magic that makes this so easy. That’s what a module should do, right?

Let’s try a more complicated component, one that counts how many times it has
been clicked.

Here’s `click_counter.sfc`:

```html
<script data-type="attach">
  var count = 0;
  $(this).on('click.sfc_click_counter', function () {
    $(this).text('Clicked ' + ++count + ' times');
  });
</script>

<script data-type="detach">
  $(this).off('click.sfc_click_counter');
</script>

<template>
  <button class="click-counter">Clicked 0 times</button>
</template>

<?php
$selector = '.click-counter';
```

This has a few new elements we should talk about. We’re writing JavaScript now,
so there are `<script>` tags, but that `data-type` attribute is something
unique to SFC. To make things easier for frontend developers, you can write
attach/detach code inside these special script tags, and just reference `this`,
which is the element selected by your `$selector`.

The code in these blocks is parsed and wrapped with a lot of other stuff - the
actual output JS from this `.sfc` file is this:

```js
(function ($, Drupal, drupalSettings) {
  Drupal.behaviors.sfc_click_counter = {
    attach: function attach(context, settings) {
      $(".click-counter", context).addBack(".click-counter").once('sfcAttach').each(function () {
        var count = 0;
        $(this).on('click.sfc_click_counter', function () {
          $(this).text('Clicked ' + ++count + ' times');
        });
      });
    },
    detach: function detach(context, settings, trigger) {
      $(".click-counter", context).addBack(".click-counter").once('sfcDetach').each(function () {
        $(this).off('click.sfc_click_counter');
      });
      var element = $(".click-counter", context).addBack(".click-counter");element.removeOnce('sfcAttach');element.removeOnce('sfcDetach');
    },
  }
})(jQuery, Drupal, drupalSettings);
```

To people familiar with Drupal, this code might make sense to you. But for
modern frontend development, this is a ridiculous amount of scaffolding. This
is how "correct" JS is meant to be written in Drupal: It should be inside a
behavior, it should use attach/detach, and it should use jQuery.once. Why
should frontend developers have to write this code every time they’re trying
to write a simple Drupal behavior? I figured that if the point of SFC is to
simplify the frontend, it should try to make JS easier to write too.

All that said, if you define a normal `<script>` tag without a `data-type`,
it’ll be left as is. This is nice if you don’t like my magic scaffolding or
want to define any global JS.

So that’s the basics of defining a `.sfc` file, but as previously mentioned
you can also define SingleFileComponent plugins as PHP classes.

Here’s the same `say_hello` component as a class:

```php
<?php

namespace Drupal\example\Plugin\SingleFileComponent;

use Drupal\Core\Form\FormStateInterface;
use Drupal\sfc\ComponentBase;

/**
 * Contains an example single file component.
 *
 * @SingleFileComponent(
 *   id = "say_hello",
 *   block = {
 *     "admin_label" = "Say hello",
 *   }
 * )
 */
class SayHello extends ComponentBase {

  const TEMPLATE = <<<TWIG
<p class="say-hello">Hello {{ name }}!</p>
TWIG;

  const CSS = <<<CSS
.say-hello {
  font-size: 20px;
}
CSS;

  public function buildContextForm(array $form, FormStateInterface $form_state, array $default_values = []) {
    $form['name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Name'),
      '#default_value' => isset($default_values['name']) ? $default_values['name'] : '',
    ];
    return $form;
  }

  public function prepareContext(array &$context) {
    if (!isset($context['name'])) {
      $context['name'] = \Drupal::currentUser()->getDisplayName();
    }
  }

}
```

Class components that extend ComponentBase define their CSS/JS/Twig in
constants, and are used just like `*.sfc` components. This example does
have some functional differences though - the annotation for the plugin
includes a `block` key, which as you may guess allows class components to be
used as blocks in the Block UI or Layout Builder. This saves developers from
defining a separate block plugin themselves. Components can also have layout
and field formatter plugins derived with similar annotations. I think component
templates should be agnostic to how they’re used, but I’ve found that for
components that act as layouts and field formatters that’s very hard to pull
off, so don't sweat it.

The `buildContextForm` method in this example can be used by anyone consuming
your component - if you’re deriving a block, layout, or field formatter the
form will be used there, but it could be used in other user interfaces as well.
The values from your form will be passed directly to your template, so make
sure to sanitize them yourself.

Finally, the `prepareContext` method allows you to write PHP that processes
template context before it’s passed to Twig. This is great for grabbing default
values, or doing anything that Twig can’t normally do. It’s nice to have the
PHP code so close to the template, I’ve found that tracking down preprocesses
and alters in themes to be quite difficult.

Class components also support dependency injection, which means their PHP
methods can be fully unit tested!

For integration-style testing (any rendering of Twig is probably going to be an
integration test), SFC provides two test traits. The first is meant for Kernel
tests, and provides methods for render a component by its ID:

```php
$session = new UserSession([
  'name' => 'Default',
]);
$proxy = new AccountProxy();
$proxy->setAccount($session);
\Drupal::currentUser()->setAccount($proxy);
$this->assertEquals('<p class="say-hello">Hello Default!</p>', $this->renderComponent('say_hello', []));
```

and for rendering component objects - which is nice when using mocks:

```php
$file_system = $this->createMock(FileSystemInterface::class);
$current_user = $this->createMock(AccountProxyInterface::class);
$current_user->method('getDisplayName')->willReturn('Default');
$component = new SayHello([], 'say_hello', [], FALSE, 'vfs:/', $file_system, $current_user);
$this->assertEquals('<p class="say-hello">Hello Default!</p>', $this->renderComponentObject($component, []));
```

The other trait is for use with full-blown functional testing:

```php
$this->visitComponent(say_hello, []);
$assert_session->pageTextContains("Hello Default!");
```

You can also use this to test an individual component’s JS behavior!

To manually test components you can enable the `sfc_dev` sub-module, which
provides Drush commands for re-building component assets, and an interactive
library that lets you play with all your single file components. Here’s a demo
of using the live reloading feature of the library to change an existing
component:

<iframe width="560" height="315" src="https://www.youtube.com/embed/0PF4tyO1gWo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Test coverage is important to me, especially with complex frontend components,
and SFC itself has 100% test coverage (in terms of executed lines), which is
a first for one of my projects. Not sure if I'd do it again but it was pretty
fun to test my skills out.

That’s about all I have to say about what the module does - if you want to know
more check out the project page at [https://www.drupal.org/project/sfc],
and make sure read the [README.md] file which goes into more detail about all
these features.

SFC is still in the alpha phase of development, so be wary that things might
change a bit before its stable release. However, no major rewrites are
wanted or planned.

Thanks for reading and please try the module out!

[https://www.drupal.org/project/sfc]: https://www.drupal.org/project/sfc
[README.md]: https://git.drupalcode.org/project/sfc/blob/8.x-1.x/README.md
