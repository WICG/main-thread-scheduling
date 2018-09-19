# Main thread Scheduling

## Motivation: Main thread contention
Most user interactions (like taps, clicks) and rendering require main thread work. Script also executes on the main thread. It is easy for long running script to hold up the main thread and cause responsiveness issues, such as:

* High/variable input latency: critical user interaction events (tap, click, scroll, wheel, etc) are queued behind long tasks, which yields janky and unpredictable user experience.
* High/variable event handling latency: similar to input, but for processing event callbacks (e.g. onload events etc), which delay application updates.
* Janky animations and scrolling: some animation and scrolling interactions require coordination between compositor and main threads; if the main thread is blocked due to a long task, it can affect responsiveness of animations and scrolling.

**Key strategies** for addressing main thread contention and improving responsiveness:
1. Break up script work into chunks and execute asynchronously on main thread
2. Move work off the main thread

These are essentially scheduling problems, improved scheduling will result in better guarantees of responsiveness.
This explainer will focus on #1: scheduling on main thread.
TODO(panicker): link to explainer for #2 Off Main Thread Scheduling.

## Main Thread Scheduling
Schedulers have been built in userspace to chunk up main thread work and schedule it at appropriate times, in order to improve responsiveness and maintain high and smooth frame-rate.
The specific schedulers we looked at are: Google Maps Scheduler and React Scheduler. These case studies demonstrate that schedulers can be (and have been) built as JS libraries, and also point to the platform gaps that they suffer from.

### High level goals
We want to explore two avenues:
#### a. JS library
develop a freely available “standardized” (potential LAPI) scheduling library, written in JS.
#### b. Platform primitives
develop primitives to fill platform gaps so that JS schedulers can be successful and so that it is easier to serve the goal of “improved responsiveness guarantees”.

### Platform Gaps
#### The yielding & coordination issue
JS should schedule work while yielding to the browser for rendering and input.
JS schedulers need to be able to schedule chunks of work, and importantly, yield to the browser -- so that the frame is not overrun and so the browser is able to do its rendering work, and other important work like handling input.
Currently JS schedulers have to guess when the browser needs to do pertinent work, when it will schedule posted work, and how much browser-side work is remaining.
While rAF is suited for render related work that needs to happen per frame, there is lot of other work that is lower priority and should get out of the way of input and render. OTOH there is work that is higher priority than rendering at a given time eg. fetching critical components during loading.

Also, the browser doesn’t have insight into JS work and knowledge of priority that could help it to more effectively schedule this, as well as schedule it appropriately relative to browser’s own work and other async app work (such as processing network responses).

#### Requirements for new Platform primitives
The following issues require platform primitives to address, and constitute the requirements for solutions:
##### 1. Able to get out of the way of important work (input, rendering etc).
NOTE: shouldYield proposal targets this issue.
Eg. from shouldYield: during page load, an app needs to initialize a set of components and scripts. These are ordered by priority: for example, first installing event handlers on primary buttons, then a search box, then a messaging widget, and then finally moving on to analytics scripts and ads.
The developer wants to complete this work as fast as possible. For example, the messaging widget should be initialized by the time the user interacts with it. However when the user taps one of the primary buttons, they shouldn’t block until the entire page is ready.

##### 2. Able to schedule work at a higher priority than rendering 
Using rAF doesn’t fit in some cases, where it is not rendering work:
Eg. when user zooms on Google Map, it is more important to quickly fetch Maps tiles than to render.
Eg. during page navigation, it could be more important to fetch and prepare the critical content of the page, than rendering.
##### 3. Able to schedule work reliably at “normal” priority
JS schedulers need to schedule “normal” priority work, that execute at an appropriate time (eg. after paint), to spread out work while yielding to the browser (as opposed to using rIC for “idle time” work or rAF for rendering work). 
Currently they use workarounds which are inefficient and often buggy compared to first class platform support:

* messagechannel workaround (google3 nexttick used by Maps etc): use a private message channel to postMessage empty messages. A bug currently prevents yielding.
* postmessage after each rAF (used by ReactScheduler)
* settimeout 0: doesn’t work well, clamped to ~4ms
* “await yield” pattern in JS: causes a microtask to be queued 

**Why not just use rIC?**
rIC is suited to idle time work, not normal priority work AFTER yielding to browser (for important work). By design, rIC has the risk of starvation and getting postponed indefinitely.

##### 4. Able to prioritize network fetches and timing of responses
Processing of network responses (parsing and execution) happens async and can occur at inopportune times relative to other ongoing work which could be more important.
Certain responses are time sensitive (eg. when needed to respond to user interaction) while others could be lower priority (eg. optimistic prefetching).

##### 5. [MAYBE?] Able to classify priority for input (handlers)
Similar to #4 above, but for input.

* certain input is low priority (relative to current work in the app)
* certain input is urgent and needs to be processed immediately without synchronizing to rendering (waiting until rAF)
TODO: Add examples.

##### 6. Able to target lower or different frame rate
Apps do not have access to easily learn the browser’s current target frame rate, and have to infer this withbook-keeping and guessing.
Furthermore, apps are not easily able to target a different frame rate, or ask the browser to target different frame rate; default targeting 60fps can often result in starving of other necessary work.
Use-cases:
Eg. Maps is building a throttling scheduler (non-trivial effort) for the purpose of targeting a lower frame rate during certain cases like zooming, when a lot of tiles need to be loaded, and rendering work can easily starve the loading work.
Eg. The React scheduler defaults to a target of 30fps with their own book-keeping, and have built detection (by timing successive scheduling of frames) for increasing to a higher target FPS. 

Some of the above could be addressed with JS library except for changing browser's target frame rate, as well as accurately knowing what the current target rate is.


