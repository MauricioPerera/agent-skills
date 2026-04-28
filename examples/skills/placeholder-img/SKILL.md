---
schema_version: "1.0"
id: "placeholder-img"
version: "1.0.0"
title: "Generate SVG placeholder image"
description: "Returns a deterministic SVG image with given width, height, and optional background color. Useful for UI mockups, design prototypes, and dummy assets."
use_when: "the user asks for a placeholder image, mockup asset, or dummy image with specific dimensions and an optional color"

command_template: "curl -fsS 'https://img.automators.work/{width}x{height}?bg={bg}'"

args:
  width:
    type: integer
    description: "image width in pixels"
    range: [1, 4000]
  height:
    type: integer
    description: "image height in pixels"
    range: [1, 4000]
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

required_commands: ["curl"]
required_env: []
network:
  - "https://img.automators.work/*"

examples:
  - intent: "I need a 800x600 placeholder image for a hero section"
    command: "curl -fsS 'https://img.automators.work/800x600?bg=cccccc'"
  - intent: "give me a dark blue 1200x400 banner placeholder"
    command: "curl -fsS 'https://img.automators.work/1200x400?bg=1e3a5f'"
  - intent: "I want a small square placeholder 200x200, orange background"
    command: "curl -fsS 'https://img.automators.work/200x200?bg=f97316'"
  - intent: "default placeholder, just any size like 400x300"
    command: "curl -fsS 'https://img.automators.work/400x300?bg=cccccc'"

provenance:
  source: "git"
  repo: "github.com/example/agent-skills"
  commit: "0000000000000000000000000000000000000000"
  tag: "v1.0.0"
  published_at: "2026-04-28T00:00:00Z"
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

- Maximum dimensions: 4000 × 4000.
- Invalid hex colors silently fall back to `cccccc` (light gray).
- No authentication, no rate limiting beyond Cloudflare defaults.
