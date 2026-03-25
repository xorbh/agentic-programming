---
layout: post
title: "Back to the Terminal: Why the Future of Coding Looks Like the Past"
date: 2026-03-18
tags: [terminal, vim, slash-commands, abstraction, ai-agents]
---

<div class="tldr">
<strong>TL;DR:</strong> The mouse, the GUI, the visual code editor. Each was an abstraction built to make computers accessible to humans. AI agents don't need any of them. The terminal, with its keyboard-driven commands and text-based interface, is the most direct way to interact with a computer. Slash commands, born in IRC in 1988, are now the interface pattern for AI coding tools. We're going full circle.
</div>

## The Mouse Was an Abstraction

On December 9, 1968, Douglas Engelbart stood in front of a crowd at the ACM/IEEE Fall Joint Computer Conference in San Francisco and gave what would become known as the ["Mother of All Demos."](https://en.wikipedia.org/wiki/The_Mother_of_All_Demos) He demonstrated hypertext, video conferencing, collaborative editing, and a small wooden device with a cord coming out the back. He called it a mouse because the cord looked like a tail.

The mouse solved a real problem. Before it, interacting with a computer meant typing precise commands into a terminal. You had to know the commands. You had to spell them correctly. You had to understand the system. The mouse, and the graphical interfaces that followed, removed that requirement. You could point at things. Click on things. Drag things around.

When Steve Jobs visited Xerox PARC in 1979 and saw the Alto's graphical interface, he understood immediately what it meant. Five years later, Apple shipped the Macintosh, marketed as ["the computer for the rest of us."](https://computerhistory.org/blog/a-computer-for-the-rest-of-us/) The Smithsonian described the shift: the GUI transformed computing from *"daunting and technical"* to *"intuitive and visual,"* moving it *"away from the domain of hobbyists and experts to the average consumer."*

This was an abstraction. A deliberate one. The GUI hid the underlying system behind icons, menus, and windows so that humans who couldn't (or didn't want to) memorize commands could still use a computer.

And it worked. It changed everything.

## The Code Editor Was an Abstraction Too

The same pattern repeated in software development. Early programmers wrote code in terminal-based editors. Then came IDEs with file trees, tab bars, syntax highlighting panels, integrated debuggers, drag-and-drop designers. Visual Studio. Eclipse. Xcode. Each generation added more GUI surface area between the developer and the underlying files.

These tools weren't frivolous. They solved real problems for human developers: navigating large codebases visually, setting breakpoints with a click, viewing git diffs in colored side-by-side panels. The mouse became the developer's primary navigation tool. Click to open a file. Click to set a cursor. Click to run tests.

But there was always a counterculture.

## Vim and the Keyboard Loyalists

In 1976, Bill Joy was a graduate student at UC Berkeley working over a 300-baud modem connection. The connection was so slow that every keystroke mattered. He couldn't afford the luxury of a graphical interface, so he built vi, a modal text editor where every key was a command.

Joy later explained the constraints that shaped vi's design:

> *"I was trying to make it usable over a 300 baud modem... People doing Emacs were sitting in labs at MIT with what were essentially fibre-channel links to the host... So they could have funny commands with the screen shimmering and all that, and meanwhile, I'm sitting at home with a modem and a terminal that can just barely get the cursor off the bottom line."*

Modal editing was born from scarcity. In normal mode, every key does something: `d` deletes, `w` moves forward a word, `y` yanks (copies), `p` pastes. You never reach for the mouse. You never leave the keyboard. Your hands stay on the home row and the editor responds to your thoughts at the speed you can type them.

In 1991, Bram Moolenaar released [Vim](https://en.wikipedia.org/wiki/Vim_(text_editor)) ("Vi IMproved"), extending Joy's design with multi-level undo, split windows, and a plugin system. Vim became one of the most enduring pieces of software in computing history. Moolenaar maintained it for over 30 years until his passing in 2023.

The Vim philosophy was never about nostalgia. It was about **directness**. No menus between you and the text. No mouse movements burning milliseconds. No panels competing for your attention. Just a keyboard, a buffer, and a command language designed to express editing operations as concisely as possible.

Decades later, millions of developers still use Vim or Vim-keybindings. Not because they're stubborn. Because the interface is faster when you know the language.

## Slash Commands: A Pattern That Never Died

While GUI applications dominated consumer software, a parallel tradition survived in text-based communication tools.

In **August 1988**, Jarkko Oikarinen created [IRC (Internet Relay Chat)](https://en.wikipedia.org/wiki/Internet_Relay_Chat) at the University of Oulu in Finland. IRC used a simple convention: regular text was a message, but text starting with `/` was a command. `/join #channel`. `/nick newname`. `/msg user hello`. `/quit`.

This was elegant. No separate command palette. No special mode. Just a single character that switched the context from "I'm talking" to "I'm instructing." The interface was the same text input used for everything else.

The pattern proved so natural that it outlived IRC itself:

| Year | Platform | How slash commands are used |
|---|---|---|
| 1988 | **IRC** | `/join`, `/nick`, `/msg`, `/quit` |
| 2013 | **Slack** | `/remind`, `/status`, built-in and custom integrations |
| 2015 | **Telegram** | Bot commands: `/start`, `/help`, custom bot interactions |
| 2020 | **Discord** | Formalized bot interactions with autocomplete and parameter validation |
| 2025 | **Claude Code** | `/help`, `/compact`, `/init`, `/review` and more |

Each of these platforms independently arrived at the same design. Not because they were copying IRC, but because the pattern is inherently efficient. A single text input that handles both content and commands. No mode switching. No mouse required. Just type.

## The Terminal Renaissance

Something interesting has been happening over the past few years. Developers who grew up with VS Code and IntelliJ are discovering (or rediscovering) terminal-based workflows.

The tools driving this aren't the crude text interfaces of the 1990s. They're modern, polished, and genuinely good:

- **Neovim**: a modernized Vim with Lua configuration, built-in LSP support, and a thriving plugin ecosystem
- **tmux**: terminal multiplexer for persistent sessions, splits, and window management
- **lazygit**: a terminal UI for git that makes the command line visual without leaving it
- **Helix**: a post-modern modal editor with LSP built in from the start
- **[Charm](https://charm.sh/)**: a framework for building beautiful terminal UIs in Go, proving that TUIs can be as polished as GUIs

Developers describe workflows like: Helix on the left, lazygit on the top right, Claude Code on the bottom right. Everything keyboard-driven. Everything composable through Unix pipes. Everything reproducible through version-controlled dotfiles.

The appeal isn't retro aesthetics. It's **speed and focus**. No notification badges. No sidebar panels fighting for attention. No extension marketplace pop-ups. Just text, commands, and output.

## Upgrade Your Terminal

If you're going back to the terminal, your terminal emulator matters more than you think.

The default terminal apps on macOS and most Linux distros were fine when you opened one window to run a quick command. That's not how terminal-based development works. When you're running Claude Code on one project, editing another, monitoring a dev server for a third, and reviewing logs for a fourth, your terminal emulator becomes your window manager. Each window or tab is a different context, and managing them well is the difference between flow and chaos.

Modern terminal emulators are worth the switch:

- **[Ghostty](https://ghostty.org/)**: Built by Mitchell Hashimoto (of Terraform and Vagrant fame). Native on macOS and Linux, GPU-accelerated, fast, and thoughtfully designed. Splits, tabs, and a configuration file that stays out of your way.
- **[Kitty](https://sw.kovidgoyal.net/kitty/)**: GPU-accelerated, extensible with kittens (plugins), and supports image rendering in the terminal. Fast and feature-rich.
- **[iTerm2](https://iterm2.com/)**: The long-standing macOS power-user terminal. Splits, profiles, search, triggers, and deep macOS integration. Not GPU-accelerated like the newer entries, but mature, stable, and packed with features. If you're already on macOS and don't want to change habits, iTerm2 with a good profile setup is a solid choice.

The common thread: GPU acceleration, proper font rendering, fast input handling, and native split/tab support. These aren't cosmetic upgrades. When you're managing four terminal panes at once, input lag and rendering jank break your flow in ways that an IDE never would because the IDE handled all of that for you.

Pick one. Configure your splits. Learn the keybindings. Your terminal windows are your new IDE panels, and managing them well is not optional.

## Claude Code: An AI Agent in the Terminal

This brings us to Claude Code, and why it matters for this discussion.

Anthropic could have built Claude Code as a GUI application. A visual IDE with drag-and-drop code blocks, a chat panel with rich formatting, inline suggestion widgets. That's what most AI coding tools look like.

Instead, they built a [terminal application](https://claude.com/product/claude-code). You type natural language. You use slash commands. The agent reads files, writes code, runs tests, and manages git, all from a text interface.

SitePoint described this as a deliberate architectural choice: *"Claude Code's terminal-native architecture is not a limitation born of engineering convenience. It is a deliberate design choice that unlocks composability with existing Unix toolchains, eliminates context-switching latency, and enables autonomous multi-step operations directly on a live codebase."*

Think about what this means. The AI agent interacts with the same underlying systems that vi and IRC interacted with decades ago. Text in, text out. Commands and responses. No GUI abstraction layer mediating the interaction.

And the slash command pattern slots right in. `/compact` to manage context. `/init` to set up a project. `/review` to check code. The same `/` prefix that Jarkko Oikarinen chose in 1988 for IRC. The same pattern that Slack adopted, that Discord formalized, that Telegram bots use.

It works because it always worked.

## Removing the Abstraction Stack

<p class="lead">This is the thread that connects vi, IRC, Vim, and Claude Code. Each chose directness over abstraction.</p>

Consider the layers between a developer's intent and the computer's action in a typical modern workflow:

1. Developer thinks "delete this function"
2. Reaches for mouse
3. Scrolls to find the function visually
4. Clicks to place cursor at the start
5. Click-drags to select the function (or uses keyboard shortcuts to select, then deletes)
6. The IDE processes the selection through its UI framework
7. The underlying text buffer is modified

In Vim: `dap` (delete a paragraph). Three keystrokes. No mouse. No visual scanning. The intent maps almost directly to the action.

Now consider an AI agent. It doesn't have eyes to scan a GUI. It doesn't have a hand to move a mouse. It interacts with the system through text: reading files, writing files, executing commands. Every GUI layer between the agent and the filesystem is an obstacle, not an aid.

This is why the terminal is the natural home for AI coding agents. Not because terminals are trendy. Because terminals are the thinnest possible interface between intent and execution. The same property that made them difficult for casual human users makes them ideal for AI agents.

<div class="callout">
<strong>The pattern:</strong> The mouse made computers accessible to humans who couldn't type commands. AI agents don't need that accessibility. They need the directness that the mouse was designed to replace.
</div>

## The Wheel Turns

The history of computing interfaces is a story of abstraction and de-abstraction.

**1970s**: Terminals and command lines. Direct, powerful, inaccessible to most people.

**1984**: The GUI. Icons, windows, the mouse. Computing becomes visual and accessible. The abstraction layer grows.

**1990s-2020s**: IDEs, visual editors, drag-and-drop builders. Each generation adds more GUI surface between the user and the system. The abstraction layer thickens.

**2020s**: AI agents arrive. They don't need icons. They don't need mice. They don't need visual layouts. They need text input, text output, and direct access to the filesystem.

The interfaces that were "primitive" turn out to be the most compatible with the most capable coding tools we've ever built. Vi's modal commands, IRC's slash convention, Unix pipes, the terminal itself: these weren't limitations waiting to be outgrown. They were design patterns waiting for a user that could fully exploit them.

We spent forty years building abstractions to make computers easier for humans. Now we have a collaborator that works best when those abstractions are removed.

The future of the coding interface looks a lot like 1976. And that's not a step backward.
