---
name: project
description: To set up a new app project. You have to open this skill before setting up a new project. Also contains documentation for generating image assets.
---

## How to set up a new project

You just unlocked the `createApp` tool. On the **first message** when the workspace is empty, you MUST call `createApp` before writing any code.

Call it with:
- `name`: A short, user-friendly app name (e.g. "My Fitness Tracker")
- `framework`: The framework for the project (`"react-native"`, `"swift"`, `"kotlin"`, or `"web"`)
- `path`: A short, lowercase folder name for the app. Pick a simple, descriptive name that matches what the app is:
  - `expo` for React Native / Expo apps
  - `ios` for native iOS / Swift apps
  - `android` for Android apps
  - `web` for web apps
  - `admin` for admin panels / dashboards
  - `api` for backend APIs
  - Use only lowercase letters, numbers, and hyphens. No spaces or special characters.

This writes the template files into a subfolder and creates a root `rork.json` manifest. After it completes, you can start editing code.

## Image and asset generation

**To generate app icons or App Store screenshots**, you MUST read `skills/image-gen/SKILL.md` first. It unlocks the `generateImage`, `generateIcon`, and `generateScreenshot` tools with full instructions on how to use them.

### How to generate image assets (backgrounds, illustrations, UI graphics)

Use the `generateImageAsset` tool for image assets. Do NOT use `generateImage` — that tool is only for app icons.

Generate custom image assets using AI (OpenAI gpt-image-1.5). Returns image URLs hosted on R2.

**Workflow:**
1. Decide what images the app needs (backgrounds, illustrations, banners, etc.)
2. Call `listExistingImageAssets` to check for duplicates
3. Call `generateImageAsset` with `runInBackground: true` for each image
4. Continue building the app — use the returned URL to reference images in code
5. At the end, call `waitImageAssetResult` for each background generation

**Sizes:** `"1024x1024"` (square, default), `"1024x1536"` (portrait), `"1536x1024"` (landscape)

**Background:** `"transparent"` for cutout images/mascots, `"opaque"` for scenes/photos, `"auto"` lets the model decide

**Edit mode:** Pass `inputImages` array with URLs to edit/composite existing images

**Tips for prompts:**
- Be specific about style, colors, composition, and where it will be used
- For app backgrounds: describe the mood, gradient, pattern
- For illustrations: describe the subject, style (flat, 3D, watercolor), colors
- Use `assetName` as a short snake_case identifier: `hero_banner`, `onboarding_bg`, `empty_state_illustration`

**NEVER retry failed image generations.** If an image fails, the user sees the error on the card and can retry manually.

NOTE: This is for "asset" type images only. For app icons, thumbnails, and App Store screenshots, use the existing `generateImage` tool with the appropriate `type` parameter.
