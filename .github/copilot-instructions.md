# AI Comic Factory - Copilot Instructions

## Project Overview

AI Comic Factory is a Next.js 14+ application that generates comic panels using LLMs and image generation models. The app uses a **multi-provider architecture** supporting OpenAI, Replicate, and Hugging Face for both text and image generation.

## Key Architecture Patterns

### State Management with Zustand
- **Global store**: `src/app/store/index.ts` manages all application state
- **Key state**: `panels[]`, `speeches[]`, `captions[]`, `renderedScenes{}`, generation status
- **Panel lifecycle**: Each panel has ID-indexed state tracking (generation, rendering, upscaling)
- Pattern: Always update state immutably via Zustand setters

### Multi-Provider Rendering System
- **Engine abstraction**: `src/app/engine/render.ts` handles 4+ rendering backends
- **Provider precedence**: User credentials override server defaults
- **Status tracking**: "pending" → "completed" | "error" with async polling for Replicate
- Pattern: Check `settings.renderingModelVendor` to determine active provider

### LLM Story Generation Pipeline
- **Two-phase generation**: Story structure → Panel-specific prompts
- **Continuation logic**: `getStoryContinuation()` builds on `existingPanels[]`
- **Degraded fallback**: If LLM fails, use style+story prompt directly
- **Prompt composition**: `lightPanelPromptPrefix` (LLM-controlled) vs `degradedPanelPromptPrefix` (fallback)

### Panel Layout System
- **Dynamic layouts**: `src/app/layouts/` defines grid configurations per page
- **Page-based rendering**: Each page can have different layouts
- **Responsive grids**: CSS Grid with print-specific overrides
- Pattern: Layout changes reset generation state completely

## Critical Development Workflows

### Adding New Rendering Providers
1. Add provider type to `RenderingEngine` in `src/types.ts`
2. Extend `newRender()` and `getRender()` in `src/app/engine/render.ts`
3. Add configuration fields to `Settings` type
4. Update settings dialog with new provider UI

### Modifying Panel Generation Logic
1. **Main flow**: `src/app/main.tsx` effect triggers on prompt changes
2. **LLM integration**: Modify `src/app/queries/predictNextPanels.ts` for prompt engineering
3. **State updates**: Always update `panels`, `speeches`, `captions` atomically
4. **Status tracking**: Use `setGeneratingImages(panelId, boolean)` per panel

### Environment Configuration
- **Server credentials**: `AUTH_*` and `RENDERING_*` env vars override user settings
- **Runtime config**: `src/lib/useDynamicConfig.ts` loads dynamic server settings
- **Local storage**: Settings persist with `localStorageKeys` pattern

## Component Architecture

### UI Structure
- **Route**: `src/app/page.tsx` → `Main` → interface components
- **Layout system**: `TopMenu` + `Page[]` + `BottomBar` + overlays
- **State binding**: Components read from Zustand store, minimal prop passing

### Panel Rendering Flow
1. `Page` component renders grid of `Panel` components
2. Each `Panel` tracks its rendering state via `renderedScenes[panelId]`
3. `Panel` triggers `newRender()` when prompt changes
4. Async polling updates `renderedScenes` until complete

## Essential Integrations

### CLAP Format Support
- **Export**: `convertComicToClap()` creates CLAP project with segments
- **Import**: `convertClapToComic()` parses existing CLAP files
- **Segment mapping**: Image → track 1, Interface → track 2, Dialogue → track 3, Camera → track 4

### Hugging Face OAuth
- **OAuth flow**: `src/lib/useOAuth.ts` handles HF authentication
- **Login wall**: Conditional based on `enableHuggingFaceOAuthWall` config
- **Token storage**: Persisted in localStorage with `usePersistedOAuth`

### Print Support
- **CSS classes**: `print:*` utilities for print-specific layouts
- **Page breaks**: Handled via CSS Grid modifications
- **Export options**: CLAP download + HTML print functionality

## Code Conventions

### File Organization
- **Server actions**: Use `"use server"` directive in `src/app/engine/`
- **Client components**: Use `"use client"` directive for interactive UI
- **Utilities**: Pure functions in `src/lib/` with descriptive names
- **Types**: Centralized in `src/types.ts` with detailed JSDoc

### Error Handling Patterns
- **Degraded modes**: Always provide fallback content when generation fails
- **Status propagation**: Use typed status enums (`"pending" | "completed" | "error"`)
- **User feedback**: Show loading states and error messages contextually

### Performance Considerations
- **Image optimization**: Base64 data URLs for generated images
- **State batching**: Group related state updates to prevent rerenders  
- **Polling optimization**: Use `sleep()` delays to reduce API pressure

## Testing & Debugging

### Local Development
- **Build command**: `npm run build` (uses Turbo for speed)
- **Dev server**: `npm run dev` with hot reload
- **Type checking**: `npm run type-check` for TypeScript validation

### Environment Testing
- **Provider switching**: Test with different `AUTH_*` tokens
- **Generation limits**: Test with various `maxNbPanels` configurations
- **Error scenarios**: Disable providers to test fallback behavior

### Common Issues
- **Panel state sync**: Check Zustand state consistency when panels don't render
- **Provider auth**: Verify API keys and model configurations in settings
- **CORS issues**: Use server actions for external API calls, not client fetch

---

*This codebase prioritizes resilient multi-provider generation with graceful degradation. When adding features, maintain the pattern of user choice + server fallbacks + error recovery.*