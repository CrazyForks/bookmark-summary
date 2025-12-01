Title: Migrating Dillo from GitHub

URL Source: https://dillo-browser.org/news/migration-from-github/

Published Time: Sun, 30 Nov 2025 12:40:09 GMT

Markdown Content:
_Written on 2025-11-30 by Rodrigo Arias Mallo_
I would like to migrate the Dillo project away from [GitHub](https://github.com/dillo-browser/) into a new home which is more friendly to be used with Dillo and solves some of its problems. This page summarizes the current situation with GitHub and why I decided to move away from it into a self-hosted server with multiple mirrors in other forges.

Background
----------

Before we dive into the details, I would like to briefly mention what happened with the old site. The original Dillo website was at dillo.org, which also had the source code of Dillo in a mercurial repository at hg.dillo.org. But it also included the mail server used to reach the developers, a bug tracker and archives for the mailing list. However, in 2022 [the domain was lost](https://dillo-browser.org/dillo.org.html) and someone else decided to buy it to put a similar site but plaged with AI generated ads. The original developers are no longer active, but luckily I had a copy of the mercurial repository and with some help I was able to recover a lot of material from the original server (some parts are still missing to this day).

I want to avoid this situation as much as possible, so we cannot rely on a single site that can go down and the whole project become lost. Initially, I uploaded the Dillo source and website to git repositories on GitHub, but I no longer think this is a good idea.

The situation with GitHub
-------------------------

GitHub has been useful to store all repositories of the Dillo project, as well as to run the CI workflows for platforms in which I don't have a machine available (like Windows, Mac OS or some BSDs).

However, it has several problems that make it less suitable to develop Dillo anymore. The most annoying problem is that the frontend barely works without JavaScript, so we cannot open issues, pull requests, source code or CI logs in Dillo itself, despite them being mostly plain HTML, which I don't think is acceptable. In the past, it used to gracefully degrade without enforcing JavaScript, but now it doesn't. Additionally, the page is very resource hungry, which I don't think is needed to render mostly static text.

Another big problem is that it is a single point of failure. I don't mean that GitHub is stored in a single machine, but it is controlled by a single entity which can unilateraly ban our repository or account and we would lose the ability to notify in that URL what happened. This can cause data loss if we don't have a local copy of all the data.

On the usability side, the platform has become more and more slow over time, which is affecting the development process. It also requires you to have a fast Internet connection at all times, which is not the case for me sometimes. Additionally, GitHub seems to encourage a "push model" in which you are notified when a new event occurs in your project(s), but I don't want to work with that model. Instead, I prefer it to work as a "pull model", so I only get updates when I specifically look for them. This model would also allow me to easily work offline. Unfortunately, I see that the same push model has been copied to alternative forges.

On the social side, I feel that it doesn't have the right tools to moderate users, specially for projects where the ratio of non-technical users to developers is high. This is specially problematic when active issues with developer notes begin to be filled with comments from users that have never contributed to the project and usually do more harm than good. This situation ends up causing burnout in developers.

Lastly, GitHub seem to follow the current trend of over-focusing on LLMs and generative AI, which are destroying the open web (or what remains of it) among [other problems](https://en.wikipedia.org/wiki/Generative_artificial_intelligence#Concerns). It has a direct impact on us because sites protect themseves with a JavaScript wall (or worse, browser fingerprinting) to prevent aggresive LLM crawler bots from overloading the site, but they also leave Dillo users out. So I would prefer not to encourage this trend. Despite my intentions, moving Dillo away won't change much their capability to train their model with our code, but at least I won't be actively helping.

Self-hosting Dillo
------------------

After researching the available options, it seems that none of the current forges would allow us to have a redundant system that can prevent the forge from becoming a single point of failure and solve the rest of the problems with GitHub. Therefore, I decided to self-host Dillo myself, move all important data to git repositories and keep them synchronized in multiple git mirrors.

I decided to buy the dillo-browser.org domain name and setup a very small VPS. Initially, I was very skeptical that it would be able to survive on today's web, but it seems to be doing an acceptable job at handling it (mostly AI bot traffic masquerading as users). The Dillo website is available here:

[https://dillo-browser.org/](https://dillo-browser.org/)

I researched which git frontends may suit our needs, and I discovered that most options are very complicated to self-host and require a lot of server resources and JavaScript on the frontend. I ended up testing cgit, which is written in C and it seems to be very lightweight both on RAM and CPU usage. Furthermore, the web frontend doesn't require JS, so I can use it from Dillo (I modified cgit CSS slightly to work well on Dillo). It is available on this URL:

[https://git.dillo-browser.org/](https://git.dillo-browser.org/)

Regarding the bug tracker, I also took a look at the available options. They are all too complicated for what I would like to have and they seem to centralize the data into a database that can get lost. This is precisely the case that happened with the old dillo bug tracker and we are still unable to recover the original bug entries.

To avoid this problem, I created my own bug tracker software, [buggy](https://git.dillo-browser.org/buggy/), which is a very simple C tool that parses plain Markdown files and creates a single HTML page for each bug. All bugs are stored in a [git repository](https://git.dillo-browser.org/bugtracker/) and a git hook regenerates the bug pages and the index on each new commit. As it is simply plain text, I can edit the bugs locally and only push them to the remote when I have Internet back, so it works nice offline. Also, as the output is just an static HTML site, I don't need to worry about having any vulnerabilities in my code, as it will only run at build time. You can see it live here, with the exported issues from GitHub:

[https://bug.dillo-browser.org/](https://bug.dillo-browser.org/)

The mailing list archives are stored by three independent external services, but I might include a copy with our own archives in the future.

Setting up mirrors
------------------

As all the important data is now stored in git repositories, we can mirror them in any forge, without having to rely on their custom storage format for the issues or other data. If a forge goes down (or goes rogue) we can simply switch to another site with low switching cost. To this end, I have created git mirrors in Codeberg and Sourcehut that are synced with our git server:

*   **Codeberg**: [https://codeberg.org/dillo/](https://codeberg.org/dillo/)
*   **Sourcehut**: [https://git.sr.ht/~dillo/](https://git.sr.ht/~dillo/)

However, we still have a single point of failure: the DNS entry of the dillo-browser.org domain. If we lose the DNS entry (like with dillo.org) it would cause a problem as all services will be unreachable. We could recover from such situation by relying on alternative ways to reach users, by the mailing list, fediverse or IRC, as well as updating the mirrors to reflect the current situation. It is not ideal, but I don't think it would cause a catastrophic data loss (like it happened before) as all the data is now stored in git and replicated across independent locations.

OpenPGP signature
-----------------

In order for this page to have some authority, the HTML file is signed with my [GPG key](https://dillo-browser.org/news/migration-from-github/rodarima.asc) (32E65EC501A1B6FDF8190D293EE6BA977EB2A253), which is the same that I use to sign the [last releases](https://dillo-browser.org/release) of Dillo and is also listed in [my GitHub user](https://github.com/rodarima.gpg). The signature [is available here](https://dillo-browser.org/news/migration-from-github/index.html.asc) and is linked to the page with the `<link>` tag using the `rel=signature` relation. You can find more information and how to verify the signature in the [Dillo RFC-006](https://dillo-browser.org/rfc/006-signatures-html/).

Using OpenPGP signatures is robust against losing the DNS entry, as the authority is not given by the TLS certificate chain but by the trust in the OpenPGP signature, so we could move the site elsewhere and still claim that is owned by us. Additionally, as we can store the signatures inside all git mirrors, they are also resilient against data loss.

Closing remarks
---------------

Keep in mind that the migration process requires several moving parts and it will take a while for it to stabilize (switching costs). **The GitHub repositories won't be removed at any point in time and they will continue to be updated until we finish the migration**. When the migration process is completed, I will mark the Dillo repositories as archived and properly comunicate it in our site. It is important that we don't remove any commit or tarball release to avoid breaking downstream builds that still rely on the GitHub URL.

Lastly, I'm glad that we can have our own fully independent and self-hosted site with relatively low expenses and very little energy cost (which is good for the environment, but probably not even noticeable at large scale). With the current DNS and server costs and our current donations I consider that it is likely that we can continue covering the expenses for at least the next 3 years in the worst case scenario. If you are interested in keeping us afloat, you can [help via Liberapay](https://liberapay.com/dillo/).