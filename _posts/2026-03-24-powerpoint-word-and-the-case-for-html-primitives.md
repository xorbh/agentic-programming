---
layout: post
title: "PowerPoint, Word, and the Case for HTML Primitives"
date: 2026-03-24
tags: [communication, html, abstractions, ai-agents, tooling]
---

<div class="tldr">
<strong>TL;DR:</strong> PowerPoint and Word are translation layers. They take structured content and wrap it in proprietary formats that ultimately get rendered as HTML and CSS in a browser anyway. They exist to give humans a visual canvas for arranging information, but that canvas comes with enormous costs: vendor lock-in, format incompatibility, bloated files, and an entire class of work (formatting, layout, template wrangling) that has nothing to do with the actual message. The primitives underneath, HTML and CSS, are more expressive, more portable, and more composable. What we need isn't another office suite. It's a communication layer built on web primitives that preserves what actually matters and throws away what doesn't.
</div>

## The Problem With Documents That Aren't Documents

Open any corporate slide deck or Word document in a browser. What happens? The application converts it to HTML and CSS and renders it in a web view. Google Docs does this. Microsoft 365 does this. Every modern document viewer does this. The proprietary format is a detour. The content starts as ideas, gets encoded into `.pptx` or `.docx`, gets stored as XML inside a ZIP archive, and then gets decoded back into HTML and CSS for you to actually read it.

This round-trip made sense in 1990 when documents lived on local disks and the web didn't exist. It makes no sense now. The final rendering target is a browser. We're adding two translation steps to end up where we could have started.

## What These Tools Actually Do

PowerPoint and Word solve a specific problem: they give non-technical people a visual interface for arranging text, images, and shapes on a page. That's it. The core value proposition is the canvas, not the format.

But the canvas comes with baggage.

**The format problem.** A `.pptx` file is a ZIP archive containing XML, embedded media, and style definitions spread across dozens of internal files. It's not readable without specialized software. You can't `grep` it. You can't `diff` it meaningfully. You can't pipe it through a script. Version control systems treat it as a binary blob. Two people editing the same deck will produce merge conflicts that no tool can resolve automatically.

To make this concrete, here's what a single slide with a title and three bullet points looks like inside a `.pptx` file. This is the actual XML that PowerPoint generates:

```xml
<!-- slide1.xml inside the .pptx ZIP archive -->
<p:sld xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main"
       xmlns:p="http://schemas.openxmlformats.org/presentationml/2006/main"
       xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">
  <p:cSld>
    <p:spTree>
      <p:nvGrpSpPr>
        <p:cNvPr id="1" name=""/>
        <p:cNvGrpSpPr/>
        <p:nvPr/>
      </p:nvGrpSpPr>
      <p:grpSpPr>
        <a:xfrm>
          <a:off x="0" y="0"/>
          <a:ext cx="0" cy="0"/>
          <a:chOff x="0" y="0"/>
          <a:chExt cx="0" cy="0"/>
        </a:xfrm>
      </p:grpSpPr>
      <!-- Title -->
      <p:sp>
        <p:nvSpPr>
          <p:cNvPr id="2" name="Title 1"/>
          <p:cNvSpPr><a:spLocks noGrp="1"/></p:cNvSpPr>
          <p:nvPr><p:ph type="title"/></p:nvPr>
        </p:nvSpPr>
        <p:spPr>
          <a:xfrm>
            <a:off x="838200" y="365125"/>
            <a:ext cx="10515600" cy="1325563"/>
          </a:xfrm>
        </p:spPr>
        <p:txBody>
          <a:bodyPr/>
          <a:lstStyle/>
          <a:p>
            <a:r>
              <a:rPr lang="en-US" dirty="0"/>
              <a:t>Why We Chose HTML</a:t>
            </a:r>
          </a:p>
        </p:txBody>
      </p:sp>
      <!-- Bullet points -->
      <p:sp>
        <p:nvSpPr>
          <p:cNvPr id="3" name="Content Placeholder 2"/>
          <p:cNvSpPr><a:spLocks noGrp="1"/></p:cNvSpPr>
          <p:nvPr><p:ph idx="1"/></p:nvPr>
        </p:nvSpPr>
        <p:spPr>
          <a:xfrm>
            <a:off x="838200" y="1825625"/>
            <a:ext cx="10515600" cy="4351338"/>
          </a:xfrm>
        </p:spPr>
        <p:txBody>
          <a:bodyPr/>
          <a:lstStyle/>
          <a:p>
            <a:r>
              <a:rPr lang="en-US" dirty="0"/>
              <a:t>Portable across every device with a browser</a:t>
            </a:r>
          </a:p>
          <a:p>
            <a:r>
              <a:rPr lang="en-US" dirty="0"/>
              <a:t>Version-controllable with git</a:t>
            </a:r>
          </a:p>
          <a:p>
            <a:r>
              <a:rPr lang="en-US" dirty="0"/>
              <a:t>No vendor lock-in</a:t>
            </a:r>
          </a:p>
        </p:txBody>
      </p:sp>
    </p:spTree>
  </p:cSld>
</p:sld>
```

That's 2,326 characters and 82 lines of XML for 109 characters of actual content. A 21x overhead. And this is just the slide content. The `.pptx` archive also contains separate XML files for slide layouts, slide masters, theme definitions, content type mappings, and relationship files. A single-slide deck with no images has over a dozen internal files.

Here's the same content in HTML:

```html
<section>
  <h2>Why We Chose HTML</h2>
  <ul>
    <li>Portable across every device with a browser</li>
    <li>Version-controllable with git</li>
    <li>No vendor lock-in</li>
  </ul>
</section>
```

And in Markdown:

```markdown
## Why We Chose HTML

- Portable across every device with a browser
- Version-controllable with git
- No vendor lock-in
```

Three lines. 120 characters. The content is the format. There's nothing to decode, no coordinates to calculate, no relationship IDs to resolve. A human can read it. An AI agent can write it. Git can diff it. A browser can render it.

In token terms: the OOXML version burns roughly 580 tokens. The Markdown version is about 30. Scale that to a 30-slide deck and you're looking at ~17,000 tokens of XML overhead just to represent content that fits in ~900 tokens of Markdown. At current API pricing, that's the difference between a few cents and a meaningful line item when agents are reading and processing documents at scale.

**The formatting tax.** Ask anyone who's built a 40-slide deck how they spent their time. A significant portion went to alignment, font consistency, slide transitions, template compliance, and wrestling with bullet point indentation. This is work that contributes nothing to the quality of the message. It's overhead imposed by the tool.

**The lock-in problem.** Move a `.pptx` to Google Slides and layouts break. Move a `.docx` to Pages and formatting shifts. Export to PDF and you lose editability. These formats are designed to keep you inside their ecosystem. Interoperability is technically possible and practically unreliable.

**The collaboration problem.** Real-time collaboration in Google Docs and Office 365 works, but it's built on top of formats that were never designed for it. The complexity of keeping concurrent edits to a rich document format in sync is enormous, and it shows in the occasional formatting glitch, the cursor that jumps, the comment that attaches to the wrong paragraph.

## Why They Persist

Despite all of this, PowerPoint and Word dominate business communication. There are real reasons for that, and any replacement needs to address them honestly.

**Visual layout matters.** Slides are spatial. The arrangement of elements on a page communicates hierarchy, relationships, and emphasis in ways that linear text doesn't. A well-designed slide can convey a complex idea faster than a paragraph. This is genuine value, not just decoration.

**Familiarity is a feature.** Everyone knows how to use Word. Everyone knows what a slide deck looks like. The cognitive overhead of learning the tool is essentially zero. That's a massive advantage that any replacement has to overcome.

**The presentation ritual.** Slide decks serve a social function in organizations. They structure meetings, create a shared artifact for discussion, and provide a record of what was proposed and decided. Even if the format is inefficient, the practice has institutional momentum.

**Templates as guardrails.** Corporate templates enforce brand consistency. This is a legitimate requirement. A marketing team needs to know that every external deck follows the brand guidelines, and templates are the mechanism for that today.

These are real constraints. A replacement that ignores them will fail, regardless of how technically superior it is.

## The Format Is the Lock-In

There's another reason these tools persist, and it has nothing to do with user needs. The office suites have a vested interest in propagating proprietary formats because the formats are the lock-in.

Strip away the packaging and look at what these tools actually produce. A document is Markdown. A presentation is a website. A spreadsheet is a database. Google Suite and Office 365 wrap these primitives in proprietary formats, host them behind a login wall, and charge a subscription for access to your own content. Your writing, your presentations, your data, all sitting on someone else's infrastructure, retrievable through their APIs, exportable on their terms.

This is a data enclosure. The format creates the dependency, and the platform monetizes it. You can technically export your Google Docs, but the export is lossy, the formatting shifts, and anything that relied on platform-specific features breaks. The friction of leaving is the product.

The AI play makes this even more visible. Google integrating Gemini into Google Suite sounds like progress, but look at what it actually means: an AI agent that can only operate on your content while it stays inside Google's platform. The integration reinforces the enclosure. Your documents become more useful, but only within the walls. The AI is the concierge, not the exit.

A document stored as Markdown in a git repository has no platform dependency. Any AI agent can read it, modify it, and produce output from it without an API key, a subscription, or a connector. The content is yours because the format is open. No login wall, no export step, no permission dialog. The file is the interface.

## The Primitives Are Already There

Here's the thing that's easy to miss: the primitives for a better system already exist and have been battle-tested for decades.

**HTML** is a structured document format. It has headings, paragraphs, lists, tables, images, links, semantic sections, and metadata. It was literally designed to represent documents.

**CSS** handles layout and visual presentation. Flexbox and Grid can reproduce any slide layout. Media queries adapt content to screens, projectors, or print. Custom properties and classes can enforce brand consistency more reliably than any PowerPoint template.

**Markdown** is a lightweight syntax for authoring HTML. It's readable as plain text, trivially convertible to HTML, and already the default writing format for developers, technical writers, and increasingly for non-technical knowledge workers through tools like Notion, Obsidian, and HackMD.

**JavaScript** can handle interactivity, animations, and dynamic content when needed. But the important point is that for most communication, you don't need it. HTML and CSS are enough.

These aren't obscure technologies. They're the foundation of the web. Every browser on every device can render them. They're plain text, so they work with version control, search, and automation. They're an open standard, so there's no vendor lock-in.

<div class="callout">
<strong>The irony:</strong> We use HTML and CSS to build the applications (Google Docs, Office 365) that we then use to create documents that get converted back to HTML and CSS for viewing. The intermediate format is pure overhead.
</div>

## What a New System Would Look Like

Not everything about PowerPoint and Word should be thrown away. The goal isn't to make everyone write raw HTML. It's to build communication tools on the right primitives. Here's what that system would preserve and what it would discard.

### What to keep

**Spatial layout for presentations.** Slides are fundamentally a spatial medium. A presentation tool built on HTML and CSS would still give you a canvas for arranging content on a fixed-size surface. The difference is that the canvas produces clean HTML and CSS rather than proprietary XML. Tools like [reveal.js](https://revealjs.com/) and [Marp](https://marp.app/) already prove this works. You write content in Markdown, and the tool handles slide layout, transitions, and speaker notes, all rendered as HTML and CSS.

**WYSIWYG editing for non-technical users.** Not everyone should have to write markup. A visual editor that produces clean HTML (the way [Notion](https://www.notion.so/) or [Tiptap](https://tiptap.dev/) does) preserves the authoring experience while storing content in an open format. The key difference from Word is that the output is the primitive, not a proprietary encoding of it.

**Templates and brand consistency.** CSS is a better template system than anything PowerPoint offers. A CSS file can enforce fonts, colors, spacing, and layout constraints across every document in an organization. Unlike PowerPoint templates that users can override by accident, CSS constraints can be enforced structurally. You can make it so the brand font is the only font, not just the default font.

**Export to PDF.** Browser print engines already handle this. `@media print` CSS rules let you control exactly how a document renders for print. Chrome's headless mode can generate PDFs from HTML programmatically.

### What to throw away

**Proprietary file formats.** The `.pptx` and `.docx` formats add complexity without value. Content should be stored as HTML, Markdown, or structured data (JSON, YAML). These are plain text, diffable, searchable, and readable by any tool.

**Per-element formatting.** The ability to select a word and change its font, size, and color independently is the source of most formatting inconsistency in documents. A CSS-based system would use semantic classes instead. You mark something as a heading, a callout, or emphasized text, and the stylesheet controls how those render. This is how the web works, and it produces more consistent output with less effort.

**Embedded binary assets at arbitrary sizes.** PowerPoint lets you drop a 50MB image onto a slide and stretch it to whatever dimensions you want. A modern system would reference assets by URL, handle responsive sizing through CSS, and keep the document lightweight.

**Application-specific interactivity.** Animations, transitions, and embedded macros in PowerPoint exist because the format is a self-contained runtime. When the rendering target is a browser, you get the browser's animation capabilities for free, and they're better. CSS transitions, scroll-driven animations, and the Web Animations API are more powerful and more accessible than anything in PowerPoint's animation pane.

## The Token Tax

There's a cost to these formats that's easy to overlook until you start working with AI agents: they're enormously wasteful in terms of tokens.

A simple slide with a heading, three bullet points, and an image might be 50 words of actual content. The underlying OOXML representation of that same slide is hundreds of lines of XML specifying font metrics, position coordinates in EMUs (English Metric Units, a measurement system nobody has heard of outside the OOXML spec), color theme references, placeholder indices, and relationship IDs to embedded media. When an agent needs to read or modify that document, all of that structural noise gets loaded into context. You're burning tokens on formatting metadata that has nothing to do with the content.

The same 50 words in Markdown is roughly 50 words. In HTML, maybe 80 with tags. The signal-to-noise ratio of proprietary formats is terrible, and when you're paying per token for AI processing, that noise has a direct cost. Every document an agent reads in `.docx` or `.pptx` format is consuming context window space with information that serves the rendering engine, not the task at hand.

This isn't a theoretical concern. Try asking an AI agent to summarize a 30-page Word document versus the same content in Markdown. The Markdown version fits comfortably in context. The `.docx` version, once extracted to its XML representation, might blow past the window entirely, and most of what it's carrying is layout instructions the agent doesn't need.

## The MCP PowerPoint Trap

The AI tooling community's response to this problem has been revealing. Instead of questioning whether PowerPoint is the right output format, developers are building [MCP servers](https://github.com/GongRzhe/Office-PowerPoint-MCP-Server) that give AI agents the ability to create and edit `.pptx` files programmatically. There are at least half a dozen of these projects now, from open-source tools wrapping `python-pptx` to commercial products like [SlideSpeak](https://slidespeak.co/) and [Plus AI](https://plusai.com/features/mcp).

Think about what's happening here. An AI agent that can fluently produce HTML, CSS, and Markdown is being given a 32-tool MCP server so it can construct XML inside ZIP archives, position elements by absolute coordinates in EMUs, and manage slide master relationships. We're building elaborate bridges so that agents can produce a format that will immediately be converted back to HTML for viewing in a browser.

This is the tooling equivalent of teaching someone to communicate by writing letters, putting them in envelopes, mailing them to themselves, and then opening them to read. The content could go directly from the agent's output to the browser. The `.pptx` detour adds complexity, burns tokens, and introduces failure modes, all to produce a file that most recipients will view in a web application anyway.

The energy going into MCP PowerPoint connectors would be better spent building good HTML presentation tooling that agents can target directly.

## What This Means for AI Agents

AI agents are already good at producing structured text. They can write Markdown, HTML, and CSS fluently. Ask an agent to create a presentation and it can generate a [reveal.js](https://revealjs.com/) deck in seconds, complete with speaker notes, responsive layout, and clean semantic markup.

Ask the same agent to create a PowerPoint file and it needs a library like `python-pptx` to construct XML inside a ZIP archive, placing elements by absolute coordinates in EMUs. It's fighting the format instead of expressing the content.

<p class="lead">Proprietary document formats are human-interface abstractions. They exist to give people a canvas. Agents don't need a canvas. They need a format that maps directly to the output.</p>

When the output is rendered in a browser, that format is HTML and CSS. Every translation step between the agent's output and the final rendering is a place where fidelity is lost, complexity is added, and things can go wrong.

## Versioning: What Git Already Solved

There's another dimension to this that rarely comes up in discussions about document formats: version control.

Word and Google Docs have their own versioning systems. Word tracks changes through a revision markup layer baked into the document XML. Google Docs stores an operational transform history on Google's servers. Both work, in the narrow sense that you can see what changed. Neither works well.

Word's track changes are fragile. Accept a revision in the wrong order and formatting breaks. Multiple reviewers create a visual mess of colored underlines and strikethroughs that's harder to read than the original document. Google Docs' version history is better for browsing but offers no way to branch, merge, or compare versions structurally. You get a timeline of snapshots. If two people make conflicting edits to the same section, the last writer wins silently.

Git solved this decades ago for plain text. Branch a document, make changes in isolation, review the diff line by line, merge when ready. If two people edit the same paragraph, you get a merge conflict that forces explicit resolution instead of silent overwriting. Every version is preserved, every change is attributed, and the entire history is available offline.

But git only works well with plain text. You can't meaningfully `diff` a `.pptx` file. The XML inside is so verbose and structure-dependent that a one-word text change might touch fifteen lines of markup across three internal files. Binary embedded assets make the repository bloat quickly. This is why document versioning is stuck in the track-changes era: the formats don't support anything better.

Markdown and HTML are plain text. They work with git natively. A presentation stored as a set of Markdown files or HTML sections gets the full power of modern version control for free: branches for draft revisions, pull requests for review workflows, diffs that show exactly what changed in human-readable form. The tooling already exists. The format is the bottleneck.

## Rethinking WYSIWYG

The original promise of WYSIWYG was "what you see is what you get." In practice, it became "what you see is what you get, unless you export it, print it, open it on a different device, or share it with someone using a different version of the software." The visual editing canvas created an illusion of control over the final output that the format couldn't actually guarantee.

A better model for the age of AI agents is something closer to "what you say is what you get." You describe the content and its intent in natural language or structured markup. The system handles rendering, adapting the output to whatever medium it's consumed on: a browser, a projector, a phone screen, a PDF. The author focuses on the message. The system focuses on presentation.

This isn't hypothetical. It's how this blog post works. The content is Markdown. Jekyll renders it to HTML. CSS handles the typography, layout, and responsive behavior. The same content could be rendered as a slide deck, a PDF, or an email newsletter with different stylesheets. The author doesn't position anything on a canvas. The content adapts to its container.

The WYSIWYG editor doesn't disappear in this model. It becomes a review tool rather than the primary authoring tool. You write the content (or an agent writes it for you), then use a visual editor to refine the presentation. The important shift is that the source of truth is the content, not the visual layout. The layout is derived, not authored.

## Publishing: The Missing Harness

The last piece of the puzzle is publishing, and this is where the current tooling falls apart most visibly.

Consider what it takes to share a document today. You create a file in Word or Google Docs. You want to share it externally, so you export to PDF (losing editability), or share a link (requiring the recipient to have a Google account or deal with permission requests), or attach it to an email (creating a copy that immediately diverges from the source). You want to publish a presentation, so you upload it to SlideShare, or export individual slides as images, or screen-record yourself clicking through it.

Google Drive is the closest thing we have to a publishing platform for documents, and it's deeply cumbersome. Sharing permissions are confusing. Links expire or break when files move. There's no concept of a public URL that just works like a web page, because the content isn't a web page. It's a proprietary document rendered through an application.

What we need are publishing harnesses: tools that take content stored as HTML, Markdown, or structured data and publish it to where it needs to go. A blog post goes to a static site. A presentation goes to a URL that renders the slides directly in the browser. A report goes to a shared workspace. A summary goes to Slack. The content is written once, in a clean format, and the harness handles distribution.

This is what static site generators like Jekyll, Hugo, and Astro do for blogs. What's missing is the equivalent for presentations, reports, and business documents. The primitives are ready. The authoring tools are getting there. The publishing layer is the gap.

There's another thing you get for free when a document is a web page: analytics. Right now, if you want to know how many people viewed your presentation or read your report, you're dependent on the platform. SlideShare shows you view counts. Google Docs tells you who opened the link. But that data lives inside the platform, gated by their interface, available on their terms.

When the document is an HTML page served from a URL, views and impressions become plain web traffic. Your hosting provider, your CDN, your analytics tool of choice, they all see it. You can use Cloudflare analytics, or server logs, or a lightweight script. The data comes from the infrastructure layer, not the application layer. You don't need SlideShare to tell you how many people saw your deck. Your ISP-level traffic data already knows. The platform stops being the gatekeeper of your own engagement metrics.

This matters more than it seems. Platforms use viewership data as a moat. They can show you just enough to keep you publishing on their platform, while keeping the detailed analytics behind a paywall or a premium tier. When your content is a web page, the analytics are yours by default. The rendering layer and the measurement layer are decoupled, which is how the rest of the web already works.

<div class="callout">
<strong>The opportunity:</strong> Build communication tools where the storage format and the rendering format are the same thing. No conversion, no export, no compatibility issues. The document is the web page is the presentation. And build the harnesses to publish that content wherever it needs to go.
</div>

## This Post Is the Example

This blog runs on GitHub. The post you're reading is a Markdown file in a git repository. When I push to main, GitHub Actions builds the site and deploys it to GitHub Pages. I don't manage a server. I don't configure a CMS. I don't think about hosting, SSL certificates, or deployment pipelines. I write content, commit it, and it becomes a web page at a public URL.

I didn't set up the build system, the layout templates, or the CSS. An AI agent did. I described what I wanted, and it produced the Jekyll configuration, the HTML layout, the stylesheet, and the deployment workflow. When I want a new post, I describe the topic and direction, and the agent writes it as a Markdown file, commits it to a branch, and opens a pull request. I review, merge, and the post is live.

The same workflow handles presentations. Markdown content, rendered as HTML slides by a tool like reveal.js or Marp, deployed to a URL. No PowerPoint. No export step. No wondering whether the fonts will render correctly on someone else's machine. The presentation is a web page. It works everywhere a browser works.

The point isn't that everyone should learn git and Markdown. The point is that the entire publishing pipeline, from authoring to deployment to analytics, works on plain text and open standards. The nuts and bolts are invisible. I don't care about Jekyll's internals or how GitHub Pages serves static files. I care about the content. Everything else is handled by the toolchain, and because the toolchain operates on primitives (Markdown, HTML, CSS, git), any part of it can be swapped, automated, or extended without touching the content itself.

This is what the publishing harness looks like when it works. Content in, web page out. No format conversion, no platform lock-in, no permission dialogs. Just a URL that anyone can open.

## The Broader Principle

PowerPoint and Word are instances of a pattern that shows up everywhere in software: tools that exist primarily to translate between what a human can comfortably author and what a machine can render. Form builders translate drag-and-drop into HTML. WYSIWYG email editors translate visual layout into inline CSS. GUI configuration tools translate clicks into config files.

These translation layers were necessary when the gap between human authoring comfort and machine-readable formats was wide. That gap is narrowing. Not because humans got better at writing markup, but because AI agents can bridge it from the other direction. An agent that can write clean HTML from a natural language description eliminates the need for a visual editor as the primary authoring tool. The visual editor becomes optional, a convenience for review and refinement rather than the only way to create content.

The primitives win in the end. Not because they're simpler (they're not, for humans), but because they're the actual thing. Everything else is a translation, and every translation has a cost.
