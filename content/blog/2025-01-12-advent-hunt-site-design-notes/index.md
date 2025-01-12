---
title: Advent Hunt Site Design Notes
date: 2025-01-12
slug: advent-hunt-site-design-notes
summary: Discussion about the tech stack and implementation of the Advent Puzzle Hunt's website, a Django web app created from scratch for the hunt.
params:
    ShowToc: true
---

I was the tech lead for the [Advent Puzzle Hunt](https://2024.adventhunt.com/), a [puzzle hunt](https://2024.adventhunt.com/about/) in December 2024 organized by Peppermint Herrings 🎏. I designed and implemented our hunt website infrastructure nearly all on my own.[^1]

[^1]: Shout-out and thanks to Zach Zagorski for one small contribution.

Our hunt website was a Django application developed from scratch. I first threw together a prototype over a weekend in April that was mainly just a puzzle database model and a page with an embedded PDF viewer and a working answer checker. The bulk of the development later happened between mid-July and when we launched the website with the teaser puzzle at the beginning of September alongside the Kickstarter for the physical box.

In this post, I will discuss the tech stack and implementation choices that I made. This post will probably only be interesting to you if you're a puzzle hunt tech person or maybe at least a Django developer. It's rather long and in the weeds, so please make use of the table of contents. All of our hunt website code is open source under an MIT license [on GitHub](https://github.com/puzzle-herrings/advent-hunt-site). For general post-hunt reflections, see our hunt [wrap-up](https://2024.adventhunt.com/wrapup/).

## Django from scratch vs. gph-site

Our hunt website was implemented using [Django](https://www.djangoproject.com/), the widely used "batteries-included" Python web framework. The batteries-included nature of Django—with common functionality like user account management, database integration, and an admin panel available out of the box—makes it a good choice for web applications where you don't want to spend time dealing with the basics. Notably, the open source [gph-site](https://github.com/galacticpuzzlehunt/gph-site) from ✈✈✈ Galactic Trendsetters ✈✈✈, commonly used as the foundation for many online hunts, is also a Django application.

If you know anything about hunt tech, you may know that many puzzle hunts use gph-site. However, I decided to create a Django application from scratch instead for this hunt. I did look to gph-site for some inspiration (such as how the answer checker normalized guesses), but for the most part, everything was written by me.

The primary motivation for doing this is that gph-site is a sizable existing web application with a lot of functionality implemented for typical online hunts. For the Advent Hunt, I knew we planned to have some major differences from typical online hunts:

* Our unlock structure was simple, specific, and nonstandard—we were going to release 24 puzzles at fixed times. This meant most of the progress and unlock mechanics and round mechanics present in typical hunts were not going to be relevant.
* Teams were expected to be small and solving one puzzle at a time. This meant functionality like WebSocket-based notifications would be overkill and likely a liability from a complexity and reliability perspective instead.
* All of our puzzles would be distributed as PDF files. This is because we planned to make the physical puzzle box through the Kickstarter, and we knew the puzzles were going to be printed out. This meant we didn't need any of the functionality to postprod puzzles within the website itself.

There was also some personal context: I happen to do a fair amount of Django web development as a secondary responsibility in my day job. (I am a data scientist at a small company of almost all data scientists, but we have some Django web apps that we develop and run.) I consider myself an intermediate Django dev but not an expert. I decided that I would feel more comfortable working with code that I wrote myself, instead of trying to adapt a sort-of-mature-but-also-not-formally-maintained codebase with a nearly 5-year history. It was also a good opportunity for me to practice Django development and try out some specific modern technologies (like htmx) without being as beholden to technical choices that were made in the past with gph-site.

I hope that having another open source puzzle hunt website example that isn't based on gph-site provides some value to the tech ecosystem for the puzzle hunt community.

## Frontend framework for reactivity: htmx

Our website is primarily server-side-rendered Django views with a handful of places with simple AJAX functionality—just a few forms. Most notably, the answer submission form is on each puzzle's page and allows a user to submit guesses and get feedback without navigating away. This is implemented using htmx.

[htmx](https://htmx.org/) is a simple JavaScript library for getting AJAX functionality. It has become popular in the Django community as an alternative to frontend frameworks like React and Vue.js. When using htmx, the client-side frontend can make AJAX calls to the server, which are Django views that just return HTML fragments. htmx then can swap those fragments into the page DOM. This functionality is configured with HTML tag attributes and doesn't require writing any JavaScript. It's much simpler for people like me, as a primarily Python developer and not a JavaScript developer.

There are a handful of places with dynamic UI behavior, like the expanding or collapsing card elements. I ended up just writing some basic JavaScript from scratch to do that. I found out about [Alpine.js](https://alpinejs.dev/) too late to use it, but I would probably try that next time.

## CSS Framework: Bulma

For styling the site, I used the [Bulma](https://bulma.io/) CSS framework. It provides UI components, layout classes, and styling helpers and is relatively easy to use.

I picked Bulma because it is one of the more widely used ones and has support for overriding CSS variables for customization. I originally looked at using Bootstrap 5, which is more popular, but unfortunately it doesn't support overriding CSS variables. I did not want to deal with building JavaScript or CSS assets and instead just load frontend dependencies from CDNs. With Bulma, I was able to [redefine the CSS variables](https://github.com/puzzle-herrings/advent-hunt-site/blob/32efaabeb7681b2406103f21a373329c1bc92670/static/css/base.css#L123) and customize colors and spacing. This didn't work perfectly smoothly—sometimes redefining the variables for the `:root` pseudo-class did not trickle down to how the variables were used for specific components, so there was a fair amount of browser developer tools debugging. Overall though, I was able to successfully customize things without needing a frontend build pipeline. I loaded Bulma from a CDN.

I also used [django-crispy-forms](https://github.com/django-crispy-forms/django-crispy-forms) for creating forms, and there is helpfully a [crispy-bulma](https://github.com/ckrybus/crispy-bulma) template pack for Bulma support.

## Static assets

I ended up with two different kinds of static assets:

- The normal static files like CSS, JavaScript, and images that are checked into version control. Changes require redeploying the application.
- Static files that are stored in database fields and hotlinked directly. Changes are just admin panel updates. The puzzle PDF files are all configured in this way.

This felt like an appropriate dichotomy because the puzzles feel more like data than code in the context of the application.

The former were all served using WhiteNoise, which is a Django package that lets you serve static files directly using the Django web server. This is nice and simple because deployments don't need to integrate with any external services. WhiteNoise does have performance limitations and recommends you use a CDN to address them. I set up our domain with [CloudFlare](https://www.cloudflare.com/plans/free/), which includes a free CDN that seemed to work well enough.

For the latter, I manually uploaded them to an object storage service. I used XXH32 hashes in the filename for versioning and made sure to set the [`Cache-Control`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) headers to instruct browsers to cache them.

As mentioned in the ["CSS Framework"](#css-framework-bulma) section, I wanted to avoid dealing with any frontend asset build pipelines. For all external JavaScript libraries and frameworks, I used public CDNs.

## Performance considerations

I wanted the site to perform well while being relatively inexpensive to run, so I paid some attention to performance and optimization with an eye towards low-hanging fruit. Given that I was developing this by myself in a short amount of time, I also tried to keep things relatively simple.

I did some experimentation using [Locust](https://locust.io/) for load testing locally (`locustfile.py` [here](https://github.com/puzzle-herrings/advent-hunt-site/blob/main/scripts/locustfile.py)). Load testing the production deployment didn't seem to work correctly, with my best guess for a culprit being anti-DDOS functionality from Cloudflare.

### Query optimization

I made all database queries just using the Django ORM. I generally tried to do as few queries as possible with the views, which meant I made a lot of use of [`select_related`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#django.db.models.query.QuerySet.select_related) and [`prefetch_related`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#prefetch-related) to join across models. I especially paid attention to not having any [N+1 queries](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping). The queries involved in the website were generally pretty simple, so it was nice practice to try to write them idiomatically with the Django ORM.

### Caching

The production website was deployed with a Redis cache. This was mainly used for [cached sessions](https://docs.djangoproject.com/en/5.1/topics/http/sessions/#using-cached-sessions) using the `cached_db` database backend with write-through caching.

I didn't end up implementing any other caching for the hunt. Response times seemed decent and dealing with caching seemed like too much complexity. In future situations, [django-cachealot](https://django-cachalot.readthedocs.io/en/latest/introduction.html) seems like a potentially interesting tool for database query caching. In particular, models that are not directly related to users don't change often and may be good candidates for caching.

### WSGI vs. ASGI

[WSGI and ASGI](https://asgi.readthedocs.io/en/latest/introduction.html) are two Python web server standards, with the former running applications synchronously and the latter running applications asynchronously. ASGI is more modern and things in general seem to be moving in that direction.

I ended up deciding to stick with WSGI with [Gunicorn](https://gunicorn.org/) workers. Here are some of the considerations that went into this:

- I don't actually feel like I understand either asynchronous programming or ASGI all that well.
- WhiteNoise [doesn't support ASGI](https://github.com/evansd/whitenoise/issues/251) and emits a warning when running with as ASGI. It wasn't clear to me if this would be a big deal, so I learned towards acting conservatively. I also didn't want to trust a not-as-widely-used fork like [ServeStatic](https://github.com/Archmonger/ServeStatic/) that supports ASGI.
- Making views asynchronous involves having to touch a lot of code, like [making views `async def`](https://docs.djangoproject.com/en/5.1/topics/async/#async-views) and changing ORM methods to use [async versions](https://docs.djangoproject.com/en/5.1/topics/async/#queries-the-orm). This all seemed like a pain and not friendly for quickly experimenting with.
- Trying to reason through it, it didn't seem obviously like asynchronous views would be a big performance benefit. The main source of being I/O-bound in our website would be database queries, but the Django ORM [isn't actually asynchronous yet](https://forum.djangoproject.com/t/are-concurrent-database-queries-in-asgi-a-thing/24136).

I did run the production server with 2 Gunicorn workers, based on the [recommended formula of `2 * cpus + 1`](https://docs.gunicorn.org/en/latest/design.html#how-many-workers) on our 0.5 CPU node.

I think in a future case that needs network interactions or background tasks, it might make sense to consider ASGI more seriously. For example, I ended up deciding not to try to integrate notifications in the organizing team's Discord server when puzzles were solved, because I felt like adding in a Discord webhook call would introduce performance risk in a synchronous app.

## Uptime

In the 4+ months that the hunt website was up, we had only three incidents of approximately 3 minutes of downtime each, according to my uptime monitoring service—none of which happened in December while the hunt was active. The first was in September—it seemed like maybe some kind of resource leak, where the webserver actually crashed and restarted. People seem to [solve this](https://old.reddit.com/r/django/comments/1az7z6a/django_gunicorn_do_you_guys_use_maxrequests_and/) by setting `max_requests` and having workers restart, so I configured it at that point. The second two incidents happened in November. I never really figured out what happened with these (didn't see anything in the logs), and it could have just as well been a problem with the uptime monitor.

## Other packages and libraries used

Here's some discussion on additional Django packages that I used:

- [**django-allauth**](https://docs.allauth.org/en/latest/) — Used for user account functionality. I primarily used it for email management and password resets. It also supports a lot of other things that I did not use, like authentication using social media accounts. My experience with this was mixed. I still stand by not wanting to muck with account functionality like password resets, but customizing templates and dealing with a custom user model (like having a field for team names) was a big pain.
- [**django-crispy-forms**](https://django-crispy-forms.readthedocs.io/en/latest/) and [**crispy-bulma**](https://github.com/ckrybus/crispy-bulma) — Lets you configure the rendering behavior of forms in Python with an object-oriented API and integrates directly with Bulma CSS classes.
- [**django-anymail**](https://anymail.dev/en/stable/) — Integrations to allow sending emails with transaction email service providers.
- [**django-admin-action-forms**](https://github.com/michalpokusa/django-admin-action-forms) and [**django-no-queryset-admin-actions**](https://pypi.org/project/django-no-queryset-admin-actions/) — I used this to create admin actions to send emails to registered users.
- [**django-solo**](https://github.com/lazybird/django-solo) — I started using this fairly late, when implementing the wrap-up page. It seems to work pretty well for setting up singleton models.
- [**django-robots**](https://github.com/jazzband/django-robots) — Used for managing the `robots.txt` file for search engine crawlers.
- [**pytest-django**](https://pytest-django.readthedocs.io/en/latest/) — test runner. I'm used to using [pytest](https://docs.pytest.org/en/stable/) in all of my normal Python projects, but I've only ever used the [Django's builtin unittest-based framework](https://docs.djangoproject.com/en/5.1/topics/testing/overview/) on other projects. I took this opportunity to try out pytest-django and it was quite nice! I think I like it better then the Django tests.

And here are some of the additional frontend JavaScript and CSS libraries that I used:

* [**zkreations Tooltips.css**](https://github.com/zkreations/tooltips) — a lightweight CSS-only implementation of mouseover tooltips.
* [**Moment.js**](https://momentjs.com/) — used to determine a user's timezone from the browser and render timestamps in that timezone.

## Hosting and other services

### Web host: Render

The hunt site was hosted on [Render](https://render.com/), a platform-as-a-service provider. Render is a Heroku competitor and more or less equivalent. We've been using Render at my work, and that was what I had used to deploy my last Django app for work.

I deployed the app with the following instance types:

* **Web service**: single instance at Starter tier (0.5 CPU, 512 MB RAM) at $7/mo
* **PostgreSQL database**: Starter tier (256 MB RAM, 100m CPU, 1 GB Storage) at $7/mo
* **Redis**: Free tier (25 MB RAM, 50 connection limit)

We were able to get away with fairly lightweight and inexpensive hardware while having pretty solid and consistent performance through the hunt. Since we designed our hunt structure to be relaxed and not a race with only one puzzle each day, we didn't really see major spikes in load that brought down the website. I suspect this experience may not be representative of a more typical hunt, when you have a major spike of expected load concentrated in the early part of the hunt.

### Object storage service: Cloudflare R2

As discussed in the ["Static assets"](#static-assets) section, I served certain static files like puzzle PDFs from an object storage service. I used [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) for this. Object storage is relatively inexpensive to begin with, but R2 has zero egress fees which is a nice thing to not have to worry about.

### Email services: MailerSend + Gmail

We had three main kinds of emails to deal with:

1. Sending transactional emails, like verification and password reset emails
2. Sending email blasts for announcements to all registered users
3. Receiving and responding to support emails (questions, errata reporting, teaser hint requests)

For the first two, we used [MailerSend](https://www.mailersend.com/). Choosing a service for this was actually pretty challenging. Many services seem to have eliminated functional free tiers and inexpensive pay-as-you-go tiers. Paying more for the email service than web hosting ($~14/mo) felt wrong on principle. I initially set up [Mailgun](https://www.mailgun.com/), but it turned out that the free tier was a trial and not fully functional, and their old PAYG "Flex" plan is [no longer available](https://www.reddit.com/r/webdev/comments/1362g9d/mailgun_transactional_email_does_payg_still_exist/). Mailgun's lowest tier is $15/mo for 10,000 emails, while MailerSend has a free tier with up to 3,000 emails included and $1.00/1,000 overage.

For the third, I ended up setting up email forwarding through our DNS provider (Cloudflare) to forward emails from contact@adventhunt.com to a regular free Gmail account that I set up for the hunt. The organizing team shared login credentials, and we just collectively kept an eye on the inbox. I did have a concern about our emails to solvers being spam-filtered, but it seems like at least this wasn't an issue for most cases since people generally responded to our emails successfully.

### Application monitoring: GlitchTip on PikaPods

Render's monitoring at the Starter instance type tier is fairly barebones and not that useful.

At my work, we use [Sentry](https://sentry.io/welcome/) for monitoring. It seems like maybe Sentry has a [free tier](https://sentry.io/pricing/) up to 5,000 errors. When I was looking, I had gotten confused by their free trial, which made me think there was no free tier. Their next lowest tier is $29/mo which is quite expensive. It might be worth looking at Sentry more closely in the future, since I didn't get anywhere close to 5,000 errors.

I also tried [GlitchTip's hosted service](https://glitchtip.com/pricing). GlitchTip is an open source Sentry alternative. However, their free tier is 1,000 "events" per month, and it turns out that "events" includes their uptime monitoring pings, so I ran out immediately.

I happened to come across [PikaPods](https://www.pikapods.com/) as an inexpensive and easy way to spin up self-deployed open source web apps, including GlitchTip. It was only ~$2.65/mo to run GlitchTip in a pod and worked like a charm. Running this was a low fixed cost and not subject to any overage, which was a nice thing to just not worry about.

## Django style conventions

Since I was writing an app from scratch by myself, I got a chance to try out adopting some stylistic conventions. In particular, I was influenced by the following:

- [Django Views — The Right Way](https://spookylukey.github.io/django-views-the-right-way/)—advocates for using function-based views. As someone with some but not extensive Django experience, I generally find class-based views to be confusing and obfuscating. Function-based views feel a lot easier to write and read to me. This app also didn't really have the type of CRUD views that class-based views would've saved any boilerplate for.
- [HackSoft's Django Styleguide](https://github.com/HackSoftware/Django-Styleguide)—in particular, separating domain logic into a "service" layer.
	- Although, at the end of the day, this probably didn't matter all that much since the amount of code that went into service layers—guess submission for puzzles, plus some administrative user management things—was pretty small.
	- I did not find the "selectors" concept as useful. This probably also was a result of the simplicity of this app—I didn't really need to repeat access patterns across more than one view.

I did use custom methods on model managers and querysets quite a bit. They felt like a natural way to chain together reusable logic given how Django's ORM works, and felt pretty readable and concise enough.

## Developer experience and tools

Some developer experience notes:

* **sqlite** for local dev and testing — it's commonly recommended that you should test using the same database engine that you run in production. I bucked this by using PostgreSQL in production but using sqlite for local development and testing. I find it kind of a pain to have to run and manage Postgres locally, and spinning up Postgres in GitHub Actions CI would have cost money. This was a simple app and I felt that it was low risk to just use sqlite. It worked out fine.
* [**Just**](https://github.com/casey/just) — task runner. I'm used to using Make as a task runner and tried out Just for the first time. Since it's purpose-built to be a task runner, I found it to be a much nicer user experience than Make. I definitely recommend it as a good task runner choice.
* **`uv pip compile`** for dependency locking — I stuck with `requirements.txt` files for dependencies but used `uv pip compile` and `uv pip sync` rather than pip-tools for locking. uv is very fast! In the time since I started development though, uv has created a [project management interface](https://docs.astral.sh/uv/concepts/projects/init/#applications) that I'm looking forward trying out more.

## Fun CSS animations

### Snowflakes on homepage

The homepage has animated falling snowflakes across the hero banner. The snowflakes are just the ["tight trifoliate snowflake" Unicode character ❅](https://graphemica.com/%E2%9D%85), and the animation is done with CSS. It was adapted from [this demo by Pavel Ševčík](https://pajasevi.github.io/CSSnowflakes/).

{{< figure src="images/snowflakes.gif" caption="Clip of animated snowflakes on website home page." align="center" width="600" height="330">}}

### Interactive Christmas card

There is an interactive Christmas card that was on the homepage before December and in the [first story entry](https://2024.adventhunt.com/story/). This was also done with CSS. It was inspired by [this tutorial by CodeWizardsHQ](https://www.codewizardshq.com/html-css-tutorial-holiday-card/).

{{< figure src="images/card.gif" caption="Clip of the interactive Christmas card." align="center" width="600" height="329">}}

## Archival version

The archived version of the hunt website at https://2024.adventhunt.com is a static website version of the hunt website. First, I disabled anything that would require server-side interaction in post-hunt website state. Notably, the server-side answer checker is replaced with a client-side JavaScript version. Then, I used [HTTrack](https://www.httrack.com/), an open source website crawling tool, to scrape the website HTML pages. This works generally pretty well out of the box, though some post-processing is still necessary to fix some things. Then, the static website is served for free using GitHub Pages. You can see the code for the archiving pipeline [here](https://github.com/puzzle-herrings/advent-hunt-2024-archive).
