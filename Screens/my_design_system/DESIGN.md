---
name: My Design System
colors:
  surface: '#131314'
  surface-dim: '#131314'
  surface-bright: '#393939'
  surface-container-lowest: '#0e0e0e'
  surface-container-low: '#1b1c1c'
  surface-container: '#1f2020'
  surface-container-high: '#2a2a2a'
  surface-container-highest: '#353535'
  on-surface: '#e4e2e2'
  on-surface-variant: '#c4c7ca'
  inverse-surface: '#e4e2e2'
  inverse-on-surface: '#303030'
  outline: '#8e9194'
  outline-variant: '#44474a'
  surface-tint: '#c1c7cc'
  primary: '#e5eaef'
  on-primary: '#2b3135'
  primary-container: '#c9ced3'
  on-primary-container: '#52585c'
  inverse-primary: '#595f63'
  secondary: '#c6c6c8'
  on-secondary: '#2f3132'
  secondary-container: '#454749'
  on-secondary-container: '#b5b5b7'
  tertiary: '#f8e6db'
  on-tertiary: '#392e27'
  tertiary-container: '#dbcabf'
  on-tertiary-container: '#61544c'
  error: '#ffb4ab'
  on-error: '#690005'
  error-container: '#93000a'
  on-error-container: '#ffdad6'
  primary-fixed: '#dee3e8'
  primary-fixed-dim: '#c1c7cc'
  on-primary-fixed: '#161c20'
  on-primary-fixed-variant: '#42484b'
  secondary-fixed: '#e2e2e4'
  secondary-fixed-dim: '#c6c6c8'
  on-secondary-fixed: '#1a1c1d'
  on-secondary-fixed-variant: '#454749'
  tertiary-fixed: '#f1dfd4'
  tertiary-fixed-dim: '#d4c3b8'
  on-tertiary-fixed: '#231a13'
  on-tertiary-fixed-variant: '#50453c'
  background: '#131314'
  on-background: '#e4e2e2'
  surface-variant: '#353535'
typography:
  headline-lg:
    fontFamily: Epilogue
    fontSize: 32px
    fontWeight: '600'
    lineHeight: 40px
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: '400'
    lineHeight: 24px
  label-md:
    fontFamily: Google Sans
    fontSize: 14px
    fontWeight: '500'
    lineHeight: 20px
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  gutter: 1rem
  margin: 1.5rem
  space-xs: 0.25rem
  space-sm: 0.5rem
  space-md: 1rem
  space-lg: 1.5rem
  space-xl: 2rem
---

# Design System

## Brand & Style
The design system embraces a modern, high-fidelity dark mode aesthetic. The style is built on precision and refined neutrality, featuring an industrial yet approachable dark theme (`#c9ced3` seed) that uses subtle tones to convey focus and stability.

## Colors
The palette is optimized for dark mode (`dark`), utilizing subdued, professional tones. 
- **Primary (`#72787c`)**: Used for key actions and primary interface highlights.
- **Secondary (`#767779`)**: Supports secondary interactions and muted UI elements.
- **Tertiary (`#dbcabf`)**: Provides a warm accent contrast against the cool neutrals.
- **Neutral (`#787777`)**: Forms the structural foundation for surfaces, text, and borders.

## Typography
Typography is distributed across three carefully selected font families to optimize legibility and hierarchy:
- **Headlines**: *Epilogue* provides striking geometric weight and modern character for high-level headers.
- **Body**: *Inter* ensures maximum readability and clean rendering across paragraphs and dense data.
- **Labels**: *Google Sans* offers crisp, readable UI labels and interactive text elements.

## Layout & Spacing
A balanced spacing scale (`2`) establishes rhythm and consistency across components. Layouts rely on a fluid grid system with standard 1rem gutters and 1.5rem outer margins, scaling predictably across viewports.

## Elevation & Depth
Elevation is achieved using tonal layering combined with low-contrast outlines suited for the dark color mode. Surfaces use subtle shifts in neutrality to separate content layers without relying on heavy drop shadows.

## Shapes
The shape language uses a roundedness level of `2`, providing a friendly yet structured feel with moderately rounded corners (0.5rem base) across containers, buttons, and inputs.

## Components
Components inherit the dark mode aesthetic, utilizing the primary and secondary color accents, *Inter* for body content, *Google Sans* for labels, and a consistent `roundedness` of 2 for approachable geometry.