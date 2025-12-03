Title: “The local-first rebellion”: How Home Assistant became the most important project in your house

URL Source: https://github.blog/open-source/maintainers/the-local-first-rebellion-how-home-assistant-became-the-most-important-project-in-your-house/

Published Time: 2025-12-02T09:19:32-08:00

Markdown Content:
Franck Nijhof—better known as Frenck—is one of those maintainers who ended up at the center of a massive open source project not because he chased the spotlight, but because he helped hold together one of the most active, culturally important, and technically demanding open source ecosystems on the planet. As a **lead of**[**Home Assistant**](https://github.com/home-assistant) and a **GitHub Star**, Frenck guides the project that didn’t just grow. It exploded.

[This year’s Octoverse report confirms it](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/?utm_source=octoverse-homepage&utm_medium=blog&utm_campaign=universe25): Home Assistant was one of the **fastest-growing open source projects by contributors**, ranking alongside AI infrastructure giants like vLLM, Ollama, and Transformers. It also appeared in the **top projects attracting first-time contributors**, sitting beside massive developer platforms such as VS Code. In a year dominated by AI tooling, agentic workflows, and typed language growth, Home Assistant stood out as something else entirely: an open source system for the physical world that grew at an AI-era pace.

The scale is wild. Home Assistant is now running in **more than 2 million households**, orchestrating everything from thermostats and door locks to motion sensors and lighting. All on users’ own hardware, not the cloud. The contributor base behind that growth is just as remarkable: **21,000 contributors in a single year**, feeding into one of GitHub’s most lively ecosystems at a time when a new developer joins GitHub every second.

In our podcast interview, Frenck explains it almost casually.

> Home Assistant is a free and open source home automation platform. It allows you to connect all your devices together, regardless of the brands they’re from… And it runs locally.
> 
> Franck Nijhof, lead of Home Assistant

He smiles when he describes just how accessible it is. “Flash Home Assistant to an SD card, put it in, and it will start scanning your home,” he says.

This is the paradox that makes Home Assistant compelling to developers: it’s simple to use, but technically enormous. A local-first, globally maintained automation engine for the home. And Frenck is one of the people keeping it all running.

The architecture built to tame thousands of device ecosystems
-------------------------------------------------------------

At its core, Home Assistant’s problem is combinatorial explosion. The platform supports “hundreds, thousands of devices… over 3,000 brands,” as Frenck notes. Each one behaves differently, and the only way to normalize them is to build a general-purpose abstraction layer that can survive vendor churn, bad APIs, and inconsistent firmware.

Instead of treating devices as isolated objects behind cloud accounts, everything is represented locally as entities with states and events. A garage door is not just a vendor-specific API; it’s a structured device that exposes capabilities to the automation engine. A thermostat is not a cloud endpoint; it’s a sensor/actuator pair with metadata that can be reasoned about.

That consistency is why people can build wildly advanced automations.

Frenck describes one particularly inventive example: “Some people install weight sensors into their couches so they actually know if you’re sitting down or standing up again. You’re watching a movie, you stand up, and it will pause and then turn on the lights a bit brighter so you can actually see when you get your drink. You get back, sit down, the lights dim, and the movie continues.”

A system that can orchestrate these interactions is fundamentally a distributed event-driven runtime for physical spaces. Home Assistant may look like a dashboard, but under the hood it behaves more like a real-time OS for the home.

Running everything locally is not a feature. It’s a hard constraint.
--------------------------------------------------------------------

Almost every mainstream device manufacturer has pivoted to cloud-centric models. Frenck points out the absurdity:

> It’s crazy that we need the internet nowadays to change your thermostat.

The local-first architecture means Home Assistant can run on hardware as small as a Raspberry Pi but must handle workloads that commercial systems offload to the cloud: device discovery, event dispatch, state persistence, automation scheduling, voice pipeline inference (if local), real-time sensor reading, integration updates, and security constraints.

This architecture forces optimizations few consumer systems attempt. If any of this were offloaded to a vendor cloud, the system would be easier to build. But Home Assistant’s philosophy reverses the paradigm: the home is the data center.

Everything from SSD wear leveling on the Pi to MQTT throughput to Zigbee network topologies becomes a software challenge. And because the system must keep working offline, there’s no fallback.

This is engineering with no safety net.

The open home foundation: governance as a technical requirement
---------------------------------------------------------------

When you build a system that runs in millions of homes, the biggest long-term risk isn’t bugs. It’s ownership.

“It can never be bought, it can never be sold,” Frenck says of Home Assistant’s move to the [Open Home Foundation](https://www.openhomefoundation.org/). “We want to protect Home Assistant from the big guys in the end.”

This governance model isn’t philosophical; it is an architectural necessity. If Home Assistant ever became a commercial acquisition, cloud lock-in would follow. APIs would break. Integrations would be deprecated. Automations built over years would collapse.

![Image 1: A list of the fastest-growing open source projects by contributors. home-assistant/core is number 10.](https://github.blog/wp-content/uploads/2025/10/octoverse-2025-fastest-growing-open-source-projects-by-contributors.png?resize=1024%2C576)
The Foundation encodes three engineering constraints that ripple through every design decision:

*   **Privacy:**“Local control and privacy first.” All processing must occur on-device.
*   **Choice:**“You should be able to choose your own devices” and expect them to interoperate.
*   **Sustainability:**If a vendor kills its cloud service, the device must still work.

Frenck calls out Nest as an example: “If some manufacturer turns off the cloud service… that turns into e-waste.”

This is more than governance; it is technical infrastructure. It dictates API longevity, integration strategy, reverse engineering priorities, and local inference choices. It’s also a blueprint that forces the project to outlive any individual device manufacturer.

> We don’t build Home Assistant, the community does.

“We cannot build hundreds, thousands of device integrations. I don’t have tens of thousands of devices in my home,” Frenck says.

This is where the project becomes truly unique.

Developers write integrations for devices _they personally own_. Reviewers test contributions against devices _in their own homes_. Break something, and you break your own house. Improve something, and you improve your daily life.

“That’s where the quality comes from,” Frenck says. “People run this in their own homes… and they take care that it needs to be good.”

This is the unheard-of secret behind Home Assistant’s engineering velocity. Every contributor has access to production hardware. Every reviewer has a high-stakes environment to protect. No staging environment could replicate millions of real homes, each with its own weird edge cases.

Assist: A local voice assistant built before the AI hype wave
-------------------------------------------------------------

Assist is Home Assistant’s built-in voice assistant, a modular system that lets you control your home using speech without sending audio or transcripts to any cloud provider. As Frenck puts it:

> We were building a voice assistant before the AI hype… we want to build something privacy-aware and local.

Rather than copying commercial assistants like Alexa or Google Assistant, Assist takes a two-layer approach that prioritizes determinism, speed, and user choice.

### Stage 1: Deterministic, no-AI commands

Assist began with a structured intent engine powered by hand-authored phrases contributed by the community. Commands like “Turn on the kitchen light” or “Turn off the living room fan” are matched directly to known actions without using machine learning at all. This makes them extremely fast, reliable, and fully local. No network calls. No cloud. No model hallucinations. Just direct mapping from phrase to automation.

### Stage 2: Optional AI when you want natural language

One of the more unusual parts of Assist is that AI is never mandatory. Frenck emphasizes that developers and users get to choose their inference path: “You can even say you want to connect your own OpenAI account. Or your own Google Gemini account. Or get a Llama running locally in your own home.”

Assist evaluates each command and decides whether it needs AI. If a command is known, it bypasses the model entirely.

“Home Assistant would be like, well, I don’t have to ask AI,” Frenck says. “I know what this is. Let me turn off the lights.”

The system only uses AI when a command requires flexible interpretation, making AI a fallback instead of the foundation.

### Open hardware to support the system

To bootstrap development and give contributors a reference device, the team built a fully open source smart speaker—the **Voice Assistant Preview Edition**.

“We created a small speaker with a microphone array,” Frenck says. “It’s fully open source. The hardware is open source; the software running on it is ESPHome.”

This gives developers a predictable hardware target for building and testing voice features, instead of guessing how different microphones, DSP pipelines, or wake word configurations behave across vendors.

Hardware as a software accelerator
----------------------------------

Most open source projects avoid hardware. Home Assistant embraced it out of practical necessity.

“In order to get the software people building the software for hardware, you need to build hardware,” Frenck says.

Home Assistant Green, its prebuilt plug-and-play hub, exists because onboarding requires reliable hardware. The Voice Assistant Preview Edition exists because the voice pipeline needs a known microphone and speaker configuration.

This is a rare pattern: hardware serves as scaffolding for software evolution. It’s akin to building a compiler and then designing a reference CPU so contributors can optimize code paths predictably.

The result is a more stable, more testable, more developer-friendly software ecosystem.

A glimpse into the future: local agents and programmable homes
--------------------------------------------------------------

The trajectory is clear. With local AI models, deterministic automations, and a stateful view of the entire home, the next logical step is agentic behavior that runs entirely offline.

If a couch can trigger a movie automation, and a brewery can run a fermentation pipeline, the home itself becomes programmable. Every sensor is an input. Every device is an actuator. Every automation is a function. The entire house becomes a runtime.

And unlike cloud-bound competitors, Home Assistant’s runtime belongs to the homeowner, not the service provider.

Frenck sums up the ethos: “We give that control to our community.”

* * *

Tags:
-----

*   [generative AI](https://github.blog/tag/generative-ai/)
*   [Home Assistant](https://github.blog/tag/home-assistant/)
*   [IoT](https://github.blog/tag/iot/)
*   [Octoverse](https://github.blog/tag/octoverse/)
*   [open source](https://github.blog/tag/open-source/)
*   [python](https://github.blog/tag/python/)

Written by
----------

![Image 2: Andrea Griffiths](https://github.blog/wp-content/uploads/2025/08/Andrea-Griffiths_avatar_1755783168-200x200.jpeg)

Andrea is a Senior Developer Advocate at GitHub with over a decade of experience in developer tools. She combines technical depth with a mission to make advanced technologies more accessible. After transitioning from Army service and construction management to software development, she brings a unique perspective to bridging complex engineering concepts with practical implementation. She lives in Florida with her Welsh partner, two sons, and two dogs, where she continues to drive innovation and support open source through GitHub's global initiatives. Find her online @acolombiadev.