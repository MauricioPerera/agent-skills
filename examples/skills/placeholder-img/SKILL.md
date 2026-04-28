---
schema_version: "0.1"
id: "placeholder-img"
version: "1.0.0"
title: "Generate SVG placeholder image"
description: "Returns a deterministic SVG image with given width, height, and optional background color. Useful for UI mockups, design prototypes, and dummy assets."
use_when: "the user asks for a placeholder image, mockup asset, or dummy image with specific dimensions and an optional color"

# Placeholders are in argument position. The bank substitutes each {arg} as a
# single shell argument; values do not need to be pre-quoted in the template.
# See SPEC.md §2.6 for the substitution rule.
command_template: "curl -fsSL --get https://img.automators.work/{dimensions} --data-urlencode bg={bg}"

args:
  dimensions:
    type: string
    description: "WIDTHxHEIGHT, e.g. 800x600. Width and height each ≤ 4000."
    pattern: "^[1-9][0-9]{0,3}x[1-9][0-9]{0,3}$"
  bg:
    type: string
    description: "6-digit hex color code without leading hash"
    pattern: "^[0-9a-fA-F]{6}$"
    default: "cccccc"

license: "MIT"
author:
  name: "Automators"
  url: "https://automators.work"
homepage: "https://img.automators.work"

category: "image-generation"
tags: ["svg", "mockup", "placeholder", "design", "dummy-image"]

shell: "bash"
idempotent: true
required_commands: ["curl"]
required_env: []
network:
  - "https://img.automators.work/"

applicable_when:
  shell_commands_present: ["curl"]

examples:
  - intent: "I need a 800x600 placeholder image for a hero section"
    command: "curl -fsSL --get https://img.automators.work/800x600 --data-urlencode bg=cccccc"
  - intent: "give me a dark blue 1200x400 banner placeholder"
    command: "curl -fsSL --get https://img.automators.work/1200x400 --data-urlencode bg=1e3a5f"
  - intent: "I want a small square placeholder 200x200, orange background"
    command: "curl -fsSL --get https://img.automators.work/200x200 --data-urlencode bg=f97316"
  - intent: "default placeholder, just any size like 400x300"
    command: "curl -fsSL --get https://img.automators.work/400x300 --data-urlencode bg=cccccc"
---

# Placeholder Image Generator

Generates SVG placeholder images suitable for UI mockups and design prototypes.

## When to use this

- Filling image slots in a wireframe or HTML mockup.
- Generating consistent dummy images across a design system.
- Demonstrating layout behavior without real images.
- Any context where you need an image of a specific size but the actual content doesn't matter.

## When NOT to use this

- Production-facing images where users will see real content.
- Image generation for content (use a real generator like DALL-E, SDXL, etc.).
- Photography or detailed graphics — output is a flat-colored SVG with text showing the dimensions.

## Output

Stdout: a complete SVG image. The text inside the image is auto-styled (light or dark) based on the background's luminance for readability.

Cache: responses are cached one year (`Cache-Control: public, max-age=31536000, immutable`) — the same URL always yields the same SVG. Safe to use in loops or batch generation.

## Caveats

- Maximum dimensions: 4000 × 4000 (per the regex constraint on `dimensions`).
- Invalid hex colors silently fall back to `cccccc` (light gray).
- No authentication, no rate limiting beyond Cloudflare defaults.
- The `--data-urlencode bg=...` form puts the bg parameter on the query string after `?` while still being a single shell argument — this is the correct way to combine GET request data with `curl --get`.
