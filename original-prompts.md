Some of the md files in this repo were made by chat prompts, others were made by claude code analyzing certain repos. Those that can be replicated I'll try to note down here for updating their respective md files in the future

**UI Stragety doc**
I need to make a strategy doc for my UI component creator. I want to gather best practices for making react, and react nextjs components. This will require some research on ui component creation. Research best practices, not just what people say but what is actually being used and works. I want a strategy document that outlines how to break down a ui from ui design (eg. figma file, or from just a example website, or just a style guide and wireframe guide ) to a library of components and pages, be it in react, svelte, or vue. Include research on common pain points, such as search bars and progressive loading (libs that work), side menus and accessibility, animations and compatibility, nesting components when and why, passing props and stores and when to use each. I'd like the research to come from the most legitimate sources in both experience and reputation. The goal is to be able to reliably develop robust ui's that are easy to maintain, and facilitate good user experiences. Research first, we'll make the doc in the next step.

 note a unique point for this doc, we'll be making a framework agnostic strategy doc. The implementation won't be as necessary as much as the design patterns. Things common to the main js frameworks. We'll implement strategy docs for each framework later. If any additional research is necessary, please do so now.

**PWA Strategy doc**
Research for me the up to date way to make a pwa that is local first, uses cached items and re validates with up to date info / images if possible. Uses svelte or svelte kit, queries a cms (such as sanity), and uses tailwind. Also, optionally a kind of pwa update functionality that will check if there's a new version of the site and offers to update/refresh. The goal is to make a strategy document for a dev team to follow in addition to their requirements document.
Also, let's make a second version that is framework agnostic (no tailwind/svelte, but vite can still be assumed). So if you could generate these 2 md files


