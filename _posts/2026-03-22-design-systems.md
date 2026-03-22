---
title: "Design systems"
date: 2026-03-22
---

# Design systems

> A design system is a structured visual language.

More specifically, a design system is a shared set of rules and decisions for how a product looks and behaves. These decisions are encoded as **design tokens**.

A design system is NOT "a bunch of colors". It is:

> Raw decisions → structured as tokens → mapped through rules → consumed by components

## Design tokens

Let's answer the question what's a design token?

- Tokens are **visual decisions**
- Tokens are the **single source of truth**
- They are the **building blocks** of a design system

> _Design tokens meaningfully connect style choices that would otherwise lack a clear relationship.
	For example, if a designer's mock-ups and an engineer's implementation both reference the same token for the "secondary container color", then they can be confident that the same color is being used in both places. This applies even if the hex value assigned
	to that token gets updated._  
	Reference: [m3.material.io](https://m3.material.io/foundations/design-tokens/overview)

There are usually 3 token layers:

1. Raw tokens (primitive)

	You can think of raw tokens as the universal physics constants of your system. When you change a raw token, you're adjusting the system's physics.

	> "Make the entire system rounder."  
	"Make everything denser."  
	"Shift brand hue."  
	"Increase overall contrast."  

	Examples:

	```css
	--neutral-0: oklch(0.99 0.01 250);
	--neutral-900: oklch(0.01 0.01 250);
	--accent-500: oklch(0.5 0.01 250);
	--radius-sm: 4px;
	--spacing-4: 16px;
	```

	Another useful analogy is to think of this layer as the tweaking knobs of your system.

2. Semantic tokens

	If raw tokens are the tweaking knobs, then semantic tokens determine which knob controls which part of the UI.

	Example:

	```css
	--color-bg: var(--neutral-0);
	--color-text: var(--neutral-900);
	--color-accent: var(--accent-500);
	```

	Semantic tokens are the "API" of your design system. Components depend on them, not on raw tokens.

3. Component tokens

	This is where all the previous layers come together. Your components are now constrained by the design system and the design language it implements.

	```css
	--button-color-bg: var(--color-accent);
	--button-color-text: var(--neutral-0);
	--button-color-border: var(--neutral-200);
	```

If you are an engineer you probably already think of tokens as variables.

```
Token = CSS variable
```

But there's also the designer's perspective which is equally valid and important to understand.

For designers:

```
Token = system constraint
```

- Not a variable.
- Not a CSS value.

A constraint.

When a designer uses tokens in Figma, they aren’t thinking:

> "I'm using neutral-200."

They’re thinking:

> "This is a secondary surface."  
"This is muted text."  
"This is elevation level 2."

Tokens are not implementation. They are structure. Tokens let you separate meaning from implementation details. For example "primary action color" is meaning. Whether it’s blue, purple, or green is an implementation detail.

How do you tell raw tokens from semantic tokens?

Quick rule of thumb

If the name describes:

- a number (500)
- a hue (blue)
- a shade (light, dark)

Then it is a raw token.

If the name describes:

- a role (bg, text, border)
- a purpose (accent*, danger*)
- a context (surface-raised, on-accent)

> \* Notice how we have "accent" in both the raw and semantic tokens. This might be confusing at first but the difference is that the raw token points to a value, whereas the semantic token points to a raw token. Also, the semantic one tells you the usage intent, whether it is a bg, text, border, etc., unlike the raw token which is agnostic.

Then this is a semantic token

### On naming tokens

Naming philosophy reveals system philosophy.

There are two common conventions 

1. Hue-based naming

	```css
	--blue-500
	--red-500
	--green-500
	--neutral-100
	```

	This describes what the color looks like. It is appearance-driven.

2. Intent-based naming

	```css
	--accent-500
	--danger-500
	--success-500
	--neutral-100
	```

	This describes what the color means. It is role-driven.

Why you might want to avoid `red-500`, `blue-500`, etc.?

- Because color names are unstable.
- Today primary brand is blue, but tomorrow after a brand refresh, it might be purple. If you named your token `--blue-500`, now you either need to make it purple, or rename it to `--primary-500`. Neither option is great.
- Intent-based tokens are more stable. For example `--accent-500` can be blue, purple, or green. It doesn't matter. The meaning of the token is the same.

There are legitimate reasons to use hue-based naming though. Some systems like TailwindCSS, they define only raw primitives, e.g. `--blue-50`, `--blue-100`, `--blue-500`. This is purely a color inventory. Such systems need to be as generic and unopinionated as possible. They are not prescribing how to use these colors. That's up to the designer to decide.


### Colors

**Raw tokens** (the palette):
```css
--neutral-0: oklch(0.99 0.01 250);
--neutral-100: oklch(0.96 0.01 250);
--neutral-200: oklch(0.90 0.01 250);
--neutral-600: oklch(0.50 0.01 250);
--neutral-900: oklch(0.20 0.01 250);
--accent-500: oklch(0.60 0.18 250);
--danger-500: oklch(0.58 0.22 25);
--success-500: oklch(0.65 0.20 145);
```

**Semantic tokens** (the system API):
```css
--color-bg: var(--neutral-0);
--color-bg-subtle: var(--neutral-100);
--color-bg-muted: var(--neutral-200);
--color-text: var(--neutral-900);
--color-text-subtle: var(--neutral-600);
--color-text-inverse: var(--neutral-0);
--color-border: var(--neutral-200);
--color-accent: var(--accent-500);
--color-danger: var(--danger-500);
--color-success: var(--success-500);
```

**Component tokens** (component-specific):
```css
--button-bg: var(--color-accent);
--button-text: var(--color-text-inverse);
--button-border: var(--color-accent);
--input-bg: var(--color-bg);
--input-text: var(--color-text);
--input-border: var(--color-border);
--card-bg: var(--color-bg);
--card-border: var(--color-border);
```

### Typography

**Raw tokens** (the type scale):
```css
--font-sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
--font-mono: 'SF Mono', Monaco, monospace;
--font-size-xs: 12px;
--font-size-sm: 14px;
--font-size-base: 16px;
--font-size-lg: 18px;
--font-size-xl: 20px;
--font-size-2xl: 24px;
--font-weight-normal: 400;
--font-weight-medium: 500;
--font-weight-bold: 700;
--line-height-tight: 1.25;
--line-height-normal: 1.5;
--line-height-relaxed: 1.75;
```

**Semantic tokens** (text roles):
```css
--text-body-size: var(--font-size-base);
--text-body-weight: var(--font-weight-normal);
--text-body-line-height: var(--line-height-normal);
--text-heading-size: var(--font-size-2xl);
--text-heading-weight: var(--font-weight-bold);
--text-heading-line-height: var(--line-height-tight);
--text-caption-size: var(--font-size-sm);
--text-caption-weight: var(--font-weight-normal);
```

**Component tokens:**
```css
--button-font-size: var(--text-body-size);
--button-font-weight: var(--font-weight-medium);
--button-line-height: var(--line-height-tight);
--input-font-size: var(--text-body-size);
--input-font-weight: var(--font-weight-normal);
--heading-font-size: var(--text-heading-size);
--heading-font-weight: var(--text-heading-weight);
```

### Spacing

**Raw tokens** (the spacing scale):
```css
--spacing-0: 0;
--spacing-1: 4px;
--spacing-2: 8px;
--spacing-3: 12px;
--spacing-4: 16px;
--spacing-6: 24px;
--spacing-8: 32px;
--spacing-12: 48px;
--spacing-16: 64px;
```

**Semantic tokens** (spacing purposes):
```css
--space-xs: var(--spacing-1);
--space-sm: var(--spacing-2);
--space-md: var(--spacing-4);
--space-lg: var(--spacing-6);
--space-xl: var(--spacing-8);
--space-gap: var(--spacing-4);
--space-section: var(--spacing-8);
```

**Component tokens:**
```css
--button-padding-x: var(--space-md);
--button-padding-y: var(--space-sm);
--button-gap: var(--space-sm);
--input-padding-x: var(--space-md);
--input-padding-y: var(--space-sm);
--card-padding: var(--space-lg);
--card-gap: var(--space-md);
```

### Border radius

**Raw tokens** (the radius scale):
```css
--radius-none: 0;
--radius-sm: 4px;
--radius-md: 8px;
--radius-lg: 12px;
--radius-xl: 16px;
--radius-full: 9999px;
```

**Semantic tokens** (radius purposes):
```css
--radius-interactive: var(--radius-md);
--radius-surface: var(--radius-lg);
--radius-pill: var(--radius-full);
```

**Component tokens:**
```css
--button-radius: var(--radius-interactive);
--input-radius: var(--radius-interactive);
--card-radius: var(--radius-surface);
--badge-radius: var(--radius-pill);
--avatar-radius: var(--radius-full);
```

### Shadows

**Raw tokens** (the elevation scale):
```css
--shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
--shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
--shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1);
```

**Semantic tokens** (elevation levels):
```css
--elevation-low: var(--shadow-sm);
--elevation-medium: var(--shadow-md);
--elevation-high: var(--shadow-lg);
--elevation-highest: var(--shadow-xl);
```

**Component tokens:**
```css
--button-shadow: var(--elevation-low);
--card-shadow: var(--elevation-medium);
--dropdown-shadow: var(--elevation-high);
--modal-shadow: var(--elevation-highest);
--tooltip-shadow: var(--elevation-medium);
```

### Z-index

**Raw tokens** (the stacking scale):
```css
--z-0: 0;
--z-10: 10;
--z-20: 20;
--z-30: 30;
--z-40: 40;
--z-50: 50;
```

**Semantic tokens** (stacking contexts):
```css
--z-base: var(--z-0);
--z-dropdown: var(--z-20);
--z-sticky: var(--z-30);
--z-overlay: var(--z-40);
--z-modal: var(--z-50);
```

**Component tokens:**
```css
--dropdown-z: var(--z-dropdown);
--tooltip-z: var(--z-dropdown);
--modal-z: var(--z-modal);
--modal-backdrop-z: var(--z-overlay);
--toast-z: var(--z-modal);
```

### Animation

**Raw tokens** (timing primitives):
```css
--duration-instant: 0ms;
--duration-fast: 150ms;
--duration-normal: 250ms;
--duration-slow: 350ms;
--easing-linear: linear;
--easing-ease-in: cubic-bezier(0.4, 0, 1, 1);
--easing-ease-out: cubic-bezier(0, 0, 0.2, 1);
--easing-ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
```

**Semantic tokens** (motion purposes):
```css
--motion-instant: var(--duration-instant);
--motion-quick: var(--duration-fast);
--motion-smooth: var(--duration-normal);
--motion-deliberate: var(--duration-slow);
--motion-ease: var(--easing-ease-in-out);
```

**Component tokens:**
```css
--button-transition: var(--motion-quick) var(--easing-ease-out);
--input-transition: var(--motion-quick) var(--easing-ease-out);
--modal-transition: var(--motion-smooth) var(--motion-ease);
--dropdown-transition: var(--motion-quick) var(--easing-ease-out);
--tooltip-transition: var(--motion-instant) var(--easing-ease-out);
```

### Breakpoints

Breakpoint tokens define responsive design boundaries.

**Raw tokens:**
```css
--breakpoint-sm: 640px;
--breakpoint-md: 768px;
--breakpoint-lg: 1024px;
--breakpoint-xl: 1280px;
--breakpoint-2xl: 1536px;
```

### Other

Additional tokens for specific use cases.

**Opacity:**
```css
--opacity-disabled: 0.5;
--opacity-hover: 0.8;
--opacity-overlay: 0.75;
```

**Border width:**
```css
--border-width-thin: 1px;
--border-width-medium: 2px;
--border-width-thick: 4px;
```

**Max width:**
```css
--max-width-prose: 65ch;
--max-width-container: 1280px;
```

## Token organization

One way to organize tokens is to keep foundational and semantic tokens together, but co-locate component tokens with their components.

```
tokens/
  foundation.css
  semantic.css

components/
  Button/
    button.css
    button.tokens.css
  Input/
    input.css
	input.tokens.css
	...
```

This makes sense because:

- Component tokens only matter to that component
- They change when that component evolves
- They don’t need to pollute global namespace

This keeps the system modular.

You can also centralize tokens like this

```
tokens/
  foundation/
  semantic/
  components/
    button.json
    input.json
```

But even then, component tokens are conceptually “owned” by components.

## Variant systems

What is a variant system?

When we have:

**Button**: `primary`, `secondary`, `danger`  
**Badge**: `success`, `warning`, `info`  
**Alert**: `subtle`, `solid`, `outline`  

we are introducing stylistic branches inside a component. Where do those style differences live in the token hierarchy?

This is where many systems become messy.

First let's define a variant as: 

> Remapping of semantic tokens within a component. 

Not new foundations. Not new semantics.

Here is a simple button without variants

```css
.button {
  background: var(--color-accent);
  color: var(--color-text-inverse);
}
```

We could add primary and secondary variants like this:

```css
.button--primary {
  background: var(--color-accent);
  color: var(--color-text-inverse);
}
```

and

```css
.button--secondary {
  background: var(--color-surface-2);
  color: var(--color-text);
  border: 1px solid var(--color-border);
}
```

Variants directly use semantic tokens. This is fine for small systems, but we can go further.

To scale things nicely, we need the component structure to stay the same:

```css
.button {
  background: var(--button-bg);
  color: var(--button-text);
}
```

Now, to apply the variants we will just override local variables

```css
.button--primary {
  --button-bg: var(--color-accent);
  --button-text: var(--color-text-inverse);
}
```

and

```css
.button--secondary {
  --button-bg: var(--color-surface-2);
  --button-text: var(--color-text);
}
```

Now adding the hover state will simply look like this:

```css
.button:hover {
  background: var(--button-bg-hover);
}

.button:active {
  background: var(--button-bg-active);
  transform: translateY(1px);
}
```

and then, the primary and secondary will simply define the hover and active states in terms of the local tokens:

```css
.button--primary {
  --button-bg: var(--color-accent);
  --button-bg-hover: var(--color-accent-hover);
  --button-bg-active: var(--accent-600);

  --button-text: var(--color-text-inverse);
  --button-border: var(--color-accent);
}

.button--secondary {
  --button-bg: transparent;
  --button-bg-hover: var(--neutral-100);
  --button-bg-active: var(--neutral-200);

  --button-text: var(--color-accent);
  --button-border: var(--color-accent);
}
```

This is more scalable because:

- Variants override local tokens
- Component structure stays stable
- State styles (`:hover`, `:active`) are written once
- You can add new variants without editing base styles
- Adding a new variant requires only token overrides

## Dark mode

> Dark mode is just a remapping of semantic tokens to a different set of raw tokens.

That is one of the reasons why the semantic layer exists. It is an abstraction layer that allows us to do those remappings. This also means that the foundations (raw tokens) stay the same across light and dark mode.

Here is how light and dark modes usually compare

Light mode:
```
bg: L 0.99
text: L 0.16
```

Depth in light mode is usually achieved by reducing lightness in combination of adding shadows, which adds elevation. Text is dark but not black.

Dark mode:
```
bg: L 0.16
text: L 0.99
```

In dark mode, we start with the darkest color for background and do the inverse. However shadows are almost not used in dark mode, because they barely work. You can instead use a subtle border to achieve better visual separation. Notice also that the bg is not black and the text is not white. This is because using pure black and white is harsh on the eyes and creates sort of a glow effect.

> Note: It is true that most traditional designs in light mode start with the lightest color and use darker colors for surfaces. But it doesn't have to be this way. This is mostly due to historical reasons, because the default bg in browsers was chosen to be white. You can definitely go for more physically accurate model, which takes into account the fact that light usually comes from above and more elevated surfaces receive more light. This also has the added benfit that light and dark modes follow exactlly the same logic.

Supporting both dark and light mode is the de-facto standard today and fortunately it is fairly easy to achieve with modern CSS.

From technical perspective, what we just need to do is to define two sets of semantic tokens, one for light mode and one for dark mode, like this:

```css
:root[data-theme="light"] { ... }
:root[data-theme="dark"] { ... }
```

We can use a combination of two newly introduced APIs, `window.matchMedia` and `prefers-color-scheme` to detect the user's preferred color scheme and apply the correct theme. For example:

```js
(function() {
	const saved = localStorage.getItem('theme');
	if (saved) {
		document.documentElement.setAttribute('data-theme', saved);
	} else if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
		document.documentElement.setAttribute('data-theme', 'dark');
	}
})();
```

and then for a switch:

```js
function toggleTheme() {
	const theme = document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
	document.documentElement.setAttribute('data-theme', theme);
	localStorage.setItem('theme', theme);
}
```

```html
<button onclick="toggleTheme()">Toggle theme</button>
```

Now, remapping the tokens is the tricky part, because dark tokens are not necessarily the inverse of light tokens. This is up to the designers to decide, but here is an example

```css
:root[data-theme="light"] {
	--color-bg: var(--neutral-0);
	--color-bg-subtle: var(--neutral-50);
	--color-bg-muted: var(--neutral-100);

	--color-surface-1: var(--neutral-50);
	--color-surface-2: var(--neutral-200);
	--color-surface-3: var(--neutral-300);

	--color-text: var(--neutral-900);
	--color-text-subtle: var(--neutral-600);
	--color-text-muted: var(--neutral-400);
	--color-text-inverse: var(--neutral-0);

	--color-border: var(--neutral-200);
	--color-divider-subtle: var(--neutral-300);

	--color-accent: var(--accent-500);
	--color-accent-hover: var(--accent-600);
	--color-accent-active: var(--accent-700);
	--color-accent-soft: var(--accent-50);

	--color-secondary: var(--neutral-500);
	--color-secondary-hover: var(--neutral-600);
	--color-secondary-active: var(--neutral-700);

	--color-danger: var(--danger-500);
	--color-danger-hover: var(--danger-600);
	--color-danger-active: var(--danger-700);

	--color-success: var(--success-500);
}

:root[data-theme="dark"] {
	--color-bg: var(--neutral-900);
	--color-bg-subtle: var(--neutral-800);
	--color-bg-muted: var(--neutral-700);

	--color-surface-1: var(--neutral-800);
	--color-surface-2: var(--neutral-700);
	--color-surface-3: var(--neutral-600);

	--color-text: var(--neutral-100);
	--color-text-subtle: var(--neutral-400);
	--color-text-muted: var(--neutral-500);
	--color-text-inverse: var(--neutral-900);

	--color-border: var(--neutral-700);
	--color-divider-subtle: var(--neutral-600);

	--color-accent: var(--accent-500);
	--color-accent-hover: var(--accent-400);
	--color-accent-active: var(--accent-300);
	--color-accent-soft: var(--accent-50);

	--color-secondary: var(--neutral-500);
	--color-secondary-hover: var(--neutral-400);
	--color-secondary-active: var(--neutral-300);

	--color-danger: var(--danger-500);
	--color-danger-hover: var(--danger-400);
	--color-danger-active: var(--danger-300);

	--color-success: var(--success-500);
}
```

## Colors palettes

Color on their own could be a separate science. Fortunately, with `oklch` you can achieve pretty sensible palettes, even if you are not a designer.

Here are some ground rules.

1.  Use `oklch`

	**OKLCH gives you independent control of:**

	- (L) - brightness / base lightness range
	- (C) - intensity / contrast policy
	- (H) - identity / brand hue

	This is an important property because we can darken a color without shifting the hue.

	**The L value in OKLCH closely matches how humans perceive brightness.**

	That means that equal steps in L will produce equal visual contrast steps
	

2. Start with the neutral colors
	- If you set C to 0, then H doesn't matter. That's ok, but you can give a bit of personality by adding a tiny bit of C (e.g. 0.01) and tint the H (e.g. 250)
	- Then it's only a matter of changing the L to produce the palette. 

To be continued...

## Runtime theme picker

To be continued...