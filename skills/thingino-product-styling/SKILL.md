---
name: thingino-product-styling
description: Style Thingino-related product pages and UI blocks to match Thingino web and camera UI patterns.
license: MIT
---
# thingino-product-styling

Use this skill when creating or refining UI for Thingino-related products, catalog pages, and camera-facing web screens.

## Style baseline (from current Thingino sites)

1. Theme and framework:
   - Bootstrap 5 in dark theme (`data-bs-theme="dark"`).
   - Clean, utility-first layout with lightweight custom CSS.
2. Typography:
   - Sans UI text with Montserrat or Noto Sans.
   - Monospace for technical/config data (PT Mono style).
3. Color language:
   - Primary accent is orange (`#f80` / `#ff8800`) for active states, borders, and highlights.
   - Dark neutrals for surfaces and backgrounds.
   - Muted gray secondary text; brighter links on hover.
4. Components and spacing:
   - Card sections with clear headers, small helper text, and compact action buttons.
   - Product grids using responsive columns (`row-cols-*`) and centered images.
   - Keep whitespace generous and readable.

## Product card/content pattern

1. Product title should be prominent and easy to scan.
2. Product image should be centered and responsive (`img-fluid`).
3. Hardware tuple line should be concise and technical:
   - `SOC, SENSOR, WIFI, FLASH`.
4. Optional install/help link should be visually secondary (info-toned/muted).
5. Prefer simple, repeatable markup over heavy custom components.

## Interaction and UX pattern

1. Keep controls straightforward:
   - Secondary buttons for navigation/actions.
   - Orange accent for active/current state.
2. Use subtle feedback:
   - Soft hover color shifts.
   - Light animation only for status/attention (loading, recording, copied).
3. Avoid noisy visual effects:
   - No heavy gradients, glassmorphism, or dense shadows unless required.

## Do / Don't

1. Do:
   - Preserve Thingino visual identity: dark base + orange accent + technical clarity.
   - Use readable contrast and predictable alignment.
   - Keep metadata compact and scannable.
2. Don't:
   - Introduce unrelated color systems as primaries.
   - Over-style cards with large decorative wrappers.
   - Hide critical hardware details behind interactions.

## Quick implementation checklist

1. Set dark theme and Bootstrap baseline.
2. Define fonts and orange accent token first.
3. Build responsive product cards with centered images and hardware tuples.
4. Validate active/hover/button states use Thingino accent language.
5. Ensure technical text remains legible and compact on mobile.
