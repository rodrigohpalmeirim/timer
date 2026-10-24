# Timer

A minimal, keyboard-driven timer for running through a sequence of intervals (e.g. Pomodoro-style focus and break blocks).

**Live:** https://rodrigohpalmeirim.github.io/timer/

I couldn't quickly find a good timer app that did what I wanted. Nowadays, it's really easy to prompt something simple like this. So I did.

## Features

- **Timer sequences:** define any number of named timers (defaults to 25 min _Focus_ + 5 min _Short break_) and cycle through them. The sequence is saved in `localStorage`.
- **Overtime:** when a timer reaches zero it keeps counting how far over you went.
- **Themes:** Void, Terminal, Chalk, Dusk and Ice.
- **Distraction-free mode:** hide all UI except the clock.

## Keyboard shortcuts

| Key | Action |
| --- | --- |
| `Space` / `K` | Play / pause |
| `R` | Reset |
| `N` | Next timer |
| `S` | Toggle sequence sidebar |
| `H` | Hide / show UI |
| `Q` | Cycle theme |

## Development

Built with SvelteKit, Svelte 5 and Tailwind CSS, deployed to GitHub Pages as a static site.

```sh
bun install
bun run dev
```
