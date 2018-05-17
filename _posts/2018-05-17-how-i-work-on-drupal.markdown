---
layout: post
title:  "How I work on Drupal"
date:   2018-05-17 08:00:00 -0500
categories: drupal
---
I recently celebrated my [five-year anniversary] on Drupal.org, and wanted to
write about how I work on issues day-to-day and my general contribution "vibe".

My Drupal.org account was created the week I started working at Acquia as a
part of their employee on-boarding, and I only really used it to search issues
and post an occasional comment at first. I know a lot of people in the
community have grand stories about how they found Drupal, but mine is rather
boring, unfortunately.

Fast-forwarding to today, my full time job is to contribute to Drupal core. I
don't have an official community title, but have recently been helping out on
the Media and Layout initiatives. I also maintain or co-maintain
[28 contributed projects], which means I'm jumping in and out of different
issue queues all the time.

When you spend enough time doing the same kind of task, you end up
compartmentalizing what you do in order to make things more efficient or sane.
I do this all the time when working on issues, but never write my process down.
This post is going to be my best attempt to brain-dump what I've learned after
five years of contribution.

## Trust your feelings

You may have heard of [code smell] before, but issues can definitely feel weird
too. When I first open an issue I try to read through the entire issue summary
and every comment, then try to get a feeling for what hasn't been explicitly
said.

Trying to categorize this is tough, but here are some examples:

1. Issues marked as "Needs review" or "Reviewed and tested by the community"
(RTBC) where users are arguing in the comments. It's probably best to change
the status to "Needs work" or wait for the flames to die down before committing
the patch.
1. Issues with huge (100k+) patches. Large issues can often be split into
multiple sub-issues, or are otherwise just introducing too many changes in one
go.
1. Issues where the motivation isn't clear. Sometimes features are proposed to
projects and the true intentions of the user are intentionally hidden. I would
guess that this usually happens when a client requests something and a
developer creates a patch for it, just so they can finish the contract. That's
fine for the developer, but the maintainer has to own that code forever.
1. Issues where everyone involved is from the same company. It's not a bad
thing that a company will open an issue and push it forward by themselves, but
it's reasonable to double-check an issue marked RTBC if everyone involved works
together.

If an issue seems like a minefield, it's harder for me to jump into it and
contribute, but not impossible. Maintainers are much more likely to respond to
issues that seem clear and uncomplicated.

## Replication and confirmation

Once I know why the issue exists, I usually jump into replicating the problem.
This part is tough because replication steps aren't always available, and there
are cases where the issue is only repeatable on a customer's production site.

Regardless of how hard it may be to manually replicate a bug, or manually test
out a new feature, I think this is a really important part of fixing an issue.
If you skip this step, you may miss UX regressions or that the fix wasn't as
complete as the issue led on.

## Actual code review

Among my peers I'm probably the most lax code reviewer. I'm usually not
scouring every line of code looking for nitpicks, but do focus on the same kind
of problems in every patch:

1. Check that test coverage exists and covers _most_ of the patch.
1. Check coding standards using [PHPCodeSniffer].
1. Look for code that could be split into new methods or functions.
1. Check for logical problems - this one seems obvious but people can get so
hung up on best practices that they don't review the actual code.
1. Smell the code - if a patch doesn't feel right to me, even if I can't
explain why right away, I won't sign off on it.

For my own projects, if a patch is good enough, but not perfect, I'll probably
commit it. I think keeping the community engaged and people feeling positive
about contribution is sometimes more valuable than perfect code. For Drupal
core, maintainers have been burned by "good enough" code too often, so the
standards for what gets committed are extremely high. That could mean months of
code review for a patch, which while grueling is necessary to keeping core
maintainable and stable.

## Empathy is a hell of a drug

For projects I work on with a lot of users and open issues, I often see
contributors frustrated at how long it takes for bugs to get fixed. This one is
tough for me, because I take criticism to heart and when someone questions if
something I work on is even maintained, it makes me feel like a bad steward.

From my perspective, I'm a volunteer maintainer and don't have a lot of time
to work in issue queues. When I do have time, I focus on issues that are in
"Needs review" or RTBC. Often times those issues are missing test coverage or
have a logical problem, so I'll move them back to "Needs work" which I'm sure
frustrates contributors to no end.

From some contributor and user perspectives, maintainers have a responsibility
to respond to requests in a timely fashion, and have a duty to their userbase.
If a project doesn't have recent commits and issue activity, these users view
it as unmaintained.

Both perspectives are valid, but I can't spend 100% of my time on Drupal.org,
and while I'm at work I can (should?) only really work on Drupal core.

I spent a lot of time thinking about a middle-ground between burning out and
checking out, and have decided to mark **all my projects** as "Seeking
co-maintainer(s)". The hope here is that contributors can step up and help me
maintain projects that they're interested in, instead of relying solely on me
to fix everything. This is a little scary for me as I tend to work alone, or
as support for other maintainers, but I think it's the right move.

## Automating my job

Working on Drupal.org can feel monotonous - I end up performing the exact same
set of commands when testing and creating patches, but have never thought to
automate this before recently. The barrier to entry for the patch workflow can
can also be quite high for new users, who may be used to Github pull requests
or cowpoke coding on production.

Recently I've made a commitment to implement [DRY] for my Drupal.org workflow,
and have created a repository for Symfony commands that automate my most common
and least-liked tasks. You can view the project at [mortenson/issue].

Here's an end-to-end example of me checking out Drupal core, trying out the
latest patch from the [Media library issue I'm working on], and creating a new
patch and interdiff:

<script src="https://asciinema.org/a/T70iaxAMwz8f1vk229Wjz4g82.js" id="asciicast-T70iaxAMwz8f1vk229Wjz4g82" async></script>

Tools like this will hopefully let me work more efficiently, and help newcomers
as well.

## Parting thoughts

I've done a lot in the last five years, but am still balancing what's best for
the community and what's best for not going completely mad. I think the biggest
lesson I've learned is that the hardest problems in Drupal revolve around
communication, collaboration, respect and empathy. There are real people behind
usernames and their biggest gripes are rarely about code. I want to grow as a
developer, but want to spend more time supporting the community going forward.

Here's to five more years! 🍻☕️

[five-year anniversary]: https://drupalbirthday.fun/samuel.mortenson
[28 contributed projects]: https://www.drupal.org/project/user/samuel.mortenson
[code smell]: https://en.wikipedia.org/wiki/Code_smell
[PHPCodeSniffer]: https://www.drupal.org/node/1419988
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[mortenson/issue]: https://github.com/mortenson/issue
[Media library issue I'm working on]: https://www.drupal.org/project/drupal/issues/2962110
