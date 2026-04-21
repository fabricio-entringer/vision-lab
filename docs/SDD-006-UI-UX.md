# SDD-006: VisionLab UI/UX Specifications

## Document Information
| Field | Value |
|-------|-------|
| **Document ID** | SDD-006-UI-UX |
| **Version** | 1.0.0 |
| **Status** | Draft |
| **Last Updated** | 2025-04-21 |
| **Author** | VisionLab Team |

---

## 1. Design Principles

| Principle | Description |
|-----------|-------------|
| **Experimentation-first** | UI serves the experiment, not bureaucracy |
| **Progressive disclosure** | Show essentials; reveal advanced options on demand |
| **Immediate feedback** | Every action has visible, immediate feedback |
| **Forgiving** | Easy to undo, retry, recover from errors |
| **Minimal** | Remove everything that isn't the core task |

---

## 2. User Flow Diagram

```mermaid
flowchart TD
    Start([User lands on VisionLab]) --> Home[Home - Feature Selection]
    Home -->|Clicks feature tab| FeaturePage[Feature Page]
    
    FeaturePage --> Upload[Upload Images]
    Upload --> Prompt[Enter Prompt]
    Prompt --> Config[Adjust Parameters (optional)]
    Config --> Submit[Submit Job]
    Submit --> Processing{Job Processing}
    
    Processing -->|Queued| Waiting[Waiting in Queue]
    Processing -->|Processing| Progress[Progress Bar + ETA]
    
    Waiting --> Progress
    Progress -->|Complete| ResultView[Result Display]
    Progress -->|Error| ErrorView[Error Display]
    
    ResultView --> Download{User Action}
    Download -->|Download| FileSave[Save File]
    Download -->|Retry| Upload
    Download -->|Try Another| Home
    
    ErrorView --> Retry[Retry with Different Settings]
    Retry --> Upload
```

---

## 3. Screen Specifications

### 3.1 Home / Feature Selection

**Route:** `/`

**Purpose:** Landing page where users select which CV feature to use.

**Components:**
- Header with VisionLab logo and tagline
- Feature card grid (one card per available feature)
- Each card: icon, name, brief description, status badge
- Footer with links (GitHub, About)

**Layout:**
```
┌──────────────────────────────────────────────────────────────────┐
│  LOGO  VisionLab                                      [GitHub]  │
│  "Your Computer Vision playground"                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    AVAILABLE FEATURES                            │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    🎨           │  │    🔍 [coming]  │  │    🎯 [coming]  │  │
│  │                 │  │                 │  │                 │  │
│  │ Image Montage   │  │ Segmentation    │  │ Object Detect   │  │
│  │ Generation      │  │                 │  │                 │  │
│  │                 │  │                 │  │                 │  │
│  │ Generate images │  │ Segment objects │  │ Find objects in │  │
│  │ from prompts +  │  │ with SAM        │  │ images          │  │
│  │ input images    │  │                 │  │                 │  │
│  │                 │  │                 │  │                 │  │
│  │  [ → Try now ]  │  │  [ Coming Soon] │  │  [ Coming Soon] │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│  About  •  GitHub  •  Documentation              © 2025 VisionLab│
└──────────────────────────────────────────────────────────────────┘
```

---

### 3.2 Feature Page Template (Standard)

**Route:** `/features/{feature_id}`

**Purpose:** The standard layout used by all feature tabs.

**Sections:**
1. **Header** — Feature name, description, model info
2. **Input Area** — Upload zone, form inputs
3. **Configuration** — Collapsible advanced settings
4. **Submit** — Action button with status
5. **Results** — Result display area (hidden until job completes)

**Layout:**
```
┌──────────────────────────────────────────────────────────────────┐
│  ← Back   Feature Name                              [Model Info] │
│  Brief description of the feature                               │
├───────────────────────────┬──────────────────────────────────────┤
│                           │                                       │
│  INPUT                    │  CONFIGURATION (collapsible)          │
│  ┌─────────────────────┐  │  ┌────────────────────────────────┐  │
│  │ Drag & drop images  │  │  │ Guidance Scale:  [━━━•━━] 3.5  │  │
│  │ or click to upload  │  │  │ Steps:           [━━━•━━]  28   │  │
│  │                     │  │  │ Seed:            [42]           │  │
│  └─────────────────────┘  │  │ Output Format:   [PNG ▼]       │  │
│                           │  └────────────────────────────────┘  │
│  [📷][📷][📷]  (uploads)  │                                       │
│                           │  ┌────────────────────────────────┐  │
│  ┌─────────────────────┐  │  │                                │  │
│  │ Write your prompt...│  │  │            [ SUBMIT JOB ]     │  │
│  │                     │  │  │                                │  │
│  └─────────────────────┘  │  └────────────────────────────────┘  │
│  45/500 characters        │                                       │
│                           │                                       │
├───────────────────────────┴──────────────────────────────────────┤
│                                                                   │
│  RESULTS (shown when job completes)                              │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                                                           │   │
│  │                   Generated Image                         │   │
│  │                                                           │   │
│  │                              [📥 Download] [🔄 Retry]     │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### 3.3 Image Montage Page (Detailed)

**Route:** `/features/montage`

This is the first and primary feature implementation.

#### 3.3.1 Upload Zone

| Property | Value |
|----------|-------|
| Max images | 10 |
| Accepted formats | JPG, PNG, WebP |
| Max file size | 10MB per image |
| Input methods | Drag-and-drop, file picker, clipboard paste |

**States:**

| State | Display |
|-------|---------|
| **Empty** | Large drop zone with "Drop images here or click to upload" |
| **Dragging over** | Drop zone highlighted, dashed border, "Drop to add" |
| **Has images** | Grid of thumbnails with ability to reorder/remove |
| **Full (10)** | Drop zone disabled, message "Maximum 10 images reached" |
| **Error** | Red border on invalid files, "Invalid format: only JPG/PNG/WebP" |

#### 3.3.2 Prompt Input

```
┌────────────────────────────────────────────────────────┐
│ Write a prompt describing your desired image...        │
│                                                        │
│                                                        │
│                                                        │
└─────────────────────────────────────────────────── 0/500│
```

**Validation:**
| Condition | Message |
|-----------|---------|
| < 10 chars | "Please be more descriptive (min 10 chars)" |
| = 10-500 | ✅ Green indicator |
| > 500 chars | Disabled, red: "Prompt too long. Max 500 characters." |

**Enhancement (optional):** Prompt suggestions/templates button that inserts starter prompts.

#### 3.3.3 Submit Button States

| State | Display |
|-------|---------|
| **Disabled** | Gray, "Upload images and enter a prompt" |
| **Ready** | Primary color, clickable, "Generate Image" |
| **Submitting** | Primary, disabled, spinner, "Starting..." |
| **Failed** | Red, clickable, "Retry" |

---

### 3.4 Results Display

**Container Layout:**
```
┌──────────────────────────────────────────────────────┐
│  ✓ Generation Complete                    [📥 Download]│
├──────────────────────────────────────────────────────┤
│                                                      │
│                                                      │
│                   [RESULT IMAGE]                      │
│                   1024 x 1024                         │
│                                                      │
│                                                      │
├──────────────────────────────────────────────────────┤
│  Seed: 42     Steps: 28     Guidance: 4.0            │
│                                                      │
│  [🔄 Try Again]    [📋 Copy Prompt]    [🔗 Share]    │
└──────────────────────────────────────────────────────┘
```

**Image Display Rules:**
- Max width: 100% of container
- Centered in viewport
- Click to open lightbox (full view)
- Zoom in/out controls (optional)

---

### 3.5 Error States

#### Upload Errors

```
┌────────────────────────────────────────────────────────┐
│  ⚠ Upload Issues                                       │
│  ┌───────────────────────────────────────────────┐    │
│  │ • beach_photo.gif — Invalid format. Only JPG, │    │
│  │   PNG, and WebP are supported.                │    │
│  │ • panorama.jpg — File too large (15.2 MB).    │    │
│  │   Maximum: 10 MB.                             │    │
│  └───────────────────────────────────────────────┘    │
│                                                      │
│  [Dismiss]                                           │
└────────────────────────────────────────────────────────┘
```

#### Processing Errors

```
┌────────────────────────────────────────────────────────┐
│  ❌ Generation Failed                                  │
│                                                          │
│  Something went wrong while generating your image.       │
│  Error: Model inference failed — CUDA out of memory      │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Suggested fixes:                                  │   │
│  │ • Try reducing number of steps to 20              │   │
│  │ • Retry in a few minutes                          │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  [🔄 Retry]         [← Back to Feature]                   │
└────────────────────────────────────────────────────────┘
```

#### Network/Server Errors

```
┌────────────────────────────────────────────────────────┐
│  🔌 Connection Lost                                    │
│                                                          │
│  Unable to reach the VisionLab servers.                  │
│  Please check your internet connection and try again.    │
│                                                          │
│  [🔄 Retry]           [Details ▼]                        │
└────────────────────────────────────────────────────────┘
```

---

## 4. Component Library

### 4.1 Layout Components

#### Header

```
┌──────────────────────────────────────────┐
│ 🤖 VisionLab           [Feature A] [Help] │
└──────────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Height** | 56px |
| **Background** | `--color-bg-primary` |
| **Position** | Fixed/sticky top |
| **Contains** | Logo, app name, navigation links (minimal) |

#### Navigation (Feature Tabs)

```
┌──────────────────────────────────────────┐
│ [Montage │] │ Segmentation │] │ Detection │]│
└──────────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Style** | Horizontally aligned tabs |
| **Active** | Underlined, bold text |
| **Inactive** | Muted text, opacity 0.7 |
| **Unavailable** | Locked icon, opacity 0.4, "Coming Soon" tooltip |

#### Footer

```
┌──────────────────────────────────────────┐
│  About • GitHub • Docs • V1.0.0 ©2025   │
└──────────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Position** | Bottom, static |
| **Content** | Links, version number, copyright |

---

### 4.2 Input Components

#### ImageUploader

| Property | Value |
|----------|-------|
| **Type** | Dropzone + file picker |
| **Max files** | 10 |
| **Accept** | `image/jpeg, image/png, image/webp` |
| **Preview** | Thumbnail grid with remove/reorder |
| **Drag active** | Highlighted border + background |
| **Validation** | Inline error per file |

#### PromptInput

| Property | Value |
|----------|-------|
| **Type** | `<textarea>` with auto-resize |
| **Min length** | 10 |
| **Max length** | 500 |
| **Counter** | Bottom-right, color codes (green/yellow/red) |
| **Suggestions** | Optional template button |

#### SubmitButton

| Property | Value |
|----------|-------|
| **Type** | `<button>` primary |
| **States** | disabled, loading, ready, error |
| **Loading** | Inline spinner + label change |

---

### 4.3 Feedback Components

#### LoadingSpinner

```
     ╱╲
    ╱  ╲    ← Circular spinner (rotating)
    ╲  ╱
     ╲╱

Processing your images...
```

| Property | Value |
|----------|-------|
| **Size** | 24px (sm), 48px (lg) |
| **Color** | Primary accent |

#### ProgressBar

```
┌────────────────────────────────────────┐
│ ██████████████░░░░░░░░░░░░░░░░  45%   │
│ Step 12/28 — Inference                 │
└────────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Style** | Segmented bar (blocks fill progressively) |
| **Colors** | Primary accent on neutral bg |
| **Label** | Percentage + step description |
| **ETA** | Displayed below progress bar |

#### ErrorAlert

```
┌────────────────────────────────────────┐
│ ⚠ Error Title                          │
│                                        │
│ Error description text here...         │
│                                        │
│ [Action Button]  [Dismiss]             │
└────────────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| **Background** | `--color-error-bg` (light red) |
| **Border** | `--color-error-border` |
| **Icon** | Warning/Error icon (left) |
| **Dismissible** | Yes, with optional timeout |

---

### 4.4 Display Components

#### ImageGallery

```
┌──────┐ ┌──────┐ ┌──────┐
│ 📷   │ │ 📷   │ │ 📷   │  ← Thumbnails
│ img1 │ │ img2 │ │ img3 │
└──────┘ └──────┘ └──────┘
```

| Property | Value |
|----------|-------|
| **Layout** | Horizontal scroll / grid |
| **Thumbnail size** | 80x80px |
| **Interaction** | Click = view lightbox, X = remove |

#### ResultViewer

| Property | Value |
|----------|-------|
| **Size** | Responsive, max-width 1024px |
| **Actions** | Download, Copy, Share, Retry |
| **Lightbox** | Click image to open full-screen overlay |

---

## 5. Wireframes

### 5.1 Desktop Wireframe (Full Flow)

```
┌────────────────────────────────────────────────────────────────┐
│ 🤖 VisionLab                              [GitHub]  [Help]  [⚙│
├────────────────────────────────────────────────────────────────┤
│ [★ Montage] │ Segmentation │ Detection │ Style Transfer │ Video│
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ╔═════════════════════════╗  ╔═══════════════════════════════╗│
│  ║  Upload Images          ║  ║  Parameters                   ║│
│  ║  ╔═══════════════════╗  ║  ║  ┌─────────────────────────┐  ║│
│  ║  ║ Drop images here  ║  ║  ║  │ Guidance Scale: ███•█ 3.5│  ║│
│  ║  ║ or click to upload║  ║  ║  ├─────────────────────────┤  ║│
│  ║  ╚═══════════════════╝  ║  ║  │ Steps:          ███•█ 28 │  ║│
│  ║                         ║  ║  ├─────────────────────────┤  ║│
│  ║  ┌───┐ ┌───┐ ┌───┐     ║  ║  │ Seed: [42]              │  ║│
│  ║  │📷 │ │📷 │ │📷 │     ║  ║  ├─────────────────────────┤  ║│
│  ║  └───┘ └───┘ └───┘     ║  ║  │ Output: [PNG ▼]         │  ║│
│  ║  3/10 images            ║  ║  └─────────────────────────┘  ║│
│  ╚═════════════════════════╝  ╚═══════════════════════════════╝│
│                                                                │
│  ╔═══════════════════════════════════════════════════════════╗│
│  ║  Prompt                                                    ║│
│  ║  ┌───────────────────────────────────────────────────┐    ║│
│  ║  │ A dreamy landscape blending ocean waves with      │    ║│
│  ║  │ mountain peaks at sunset, cinematic lighting     │    ║│
│  ║  └────────────────────────────────────────────── 58/500║│
│  ╚═══════════════════════════════════════════════════════════╝│
│                                                                │
│                  ┌─────────────────────┐                       │
│                  │   🎨 Generate Image │                       │
│                  └─────────────────────┘                       │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│  About  •  GitHub  •  Documentation                  ©2025    │
└────────────────────────────────────────────────────────────────┘
```

---

### 5.2 Mobile Wireframe

```
┌──────────────────────┐
│ 🤖 VisionLab    [☰]  │
├──────────────────────┤
│ [Montage] [Seg] [...]│ ← Scrollable tabs
├──────────────────────┤
│                      │
│  Upload Images       │
│  ┌────────────────┐  │
│  │  ┌──┐ ┌──┐┌──┐ │  │
│  │  │📷│ │📷│ │📷 │ │  │
│  │  └──┘ └──┘└──┘ │  │
│  └────────────────┘  │
│                      │
│  Prompt              │
│  ┌────────────────┐  │
│  │ Describe your  │  │
│  │ image...       │  │
│  └────────────────┘  │
│                      │
│  Parameters [▼]      │
│                      │
│  ┌────────────────┐  │
│  │  Generate      │  │
│  └────────────────┘  │
│                      │
├──────────────────────┤
│  © 2025 VisionLab    │
└──────────────────────┘
```

---

## 6. Responsive Design Strategy

### 6.1 Breakpoints

| Breakpoint | Width | Target |
|------------|-------|--------|
| **sm** | ≥ 640px | Mobile landscape |
| **md** | ≥ 768px | Tablet |
| **lg** | ≥ 1024px | Laptop |
| **xl** | ≥ 1280px | Desktop |
| **2xl** | ≥ 1536px | Large desktop |

### 6.2 Responsive Behavior

| Component | Mobile (< 768px) | Tablet (768px+) | Desktop (1024px+) |
|-----------|-------------------|-----------------|-------------------|
| **Layout** | Single column | Single column | 2-column (input + config) |
| **Tabs** | Horizontally scrollable | Scrollable | All visible |
| **Upload** | Swipeable thumbnails | Grid 2x2 | Grid 5x2 |
| **Prompt** | Full width 3 rows | Full width | 3 rows |
| **Parameters** | Collapsible | Visible | Side panel |
| **Result image** | Full viewport width | 100% of container | 1024px max |

---

## 7. Accessibility Requirements (WCAG 2.1 AA)

### 7.1 Requirements Checklist

| Requirement | Standard | Implementation |
|-------------|----------|----------------|
| **Color contrast** | 4.5:1 text, 3:1 UI | All colors tested with axe DevTools |
| **Keyboard navigation** | Tab order logical | Full keyboard flow, visible focus rings |
| **Screen reader** | ARIA labels | `aria-label`, `aria-live` for progress |
| **Focus indicators** | Visible focus ring | 2px outline, `:focus-visible` |
| **Form labels** | All forms labeled | `<label>` elements linked to inputs |
| **Error messages** | Descriptive, linked | `aria-describedby` on invalid fields |
| **Image alt text** | All images described | Generated results have prompt as alt |
| **Reduced motion** | Respects `prefers-reduced-motion` | Disable animations when set |

### 7.2 Color Contrast Table

| Text Pair | Background | Foreground | Ratio |
|-----------|------------|------------|-------|
| Body text | `#18181B` (zinc-900) | `#FAFAFA` (zinc-50) | 15.4:1 ✅ |
| Body text | `#FFFFFF` | `#18181B` (zinc-900) | 15.4:1 ✅ |
| Muted text | `#FFFFFF` | `#71717A` (zinc-500) | 4.6:1 ✅ |
| Success text | `#052E16` (green-950) | `#4ADE80` (green-400) | 5.3:1 ✅ |
| Error text | `#450A0A` (red-950) | `#F87171` (red-400) | 5.4:1 ✅ |

### 7.3 ARIA Implementation

```html
<!-- Progress update region (screen reader) -->
<div aria-live="polite" aria-atomic="true" class="sr-only">
  Processing your image. Progress: 45%. Step 12 of 28.
</div>

<!-- Upload dropzone with drag state -->
<div role="region" aria-label="Image upload zone">
  <div 
    role="button" 
    tabindex="0"
    aria-describedby="upload-help">
  </div>
</div>

<!-- Results with loading -->
<img 
  src="result.png" 
  alt="Generated image: A dreamy landscape blending ocean waves" 
  aria-describedby="result-details"
/>
```

---

## 8. Theme / Color System

### 8.1 Design Tokens (Tailwind Config)

```typescript
export default {
  theme: {
    extend: {
      colors: {
        primary: {
          50:  '#EEF2FF',
          100: '#E0E7FF',
          200: '#C7D2FE',
          300: '#A5B4FC',
          400: '#818CF8',
          500: '#6366F1',  // Main accent
          600: '#4F46E5',
          700: '#4338CA',
          800: '#3730A3',
          900: '#312E81',
        },
        success: { ... },  // green
        warning: { ... },  // amber
        error:   { ... },  // red
        info:    { ... },  // blue
      },
      borderRadius: {
        'card': '12px',
        'button': '8px',
      },
      boxShadow: {
        'card': '0 1px 3px rgba(0,0,0,0.1)',
        'card-hover': '0 4px 12px rgba(0,0,0,0.15)',
      },
    }
  }
}
```

### 8.2 Dark Mode

VisionLab **defaults to dark mode**. Light mode toggle available in Phase 2.

| Element | Dark | Light |
|---------|------|-------|
| Background | `#09090B` (zinc-950) | `#FFFFFF` |
| Surface | `#18181B` (zinc-900) | `#F4F4F5` (zinc-100) |
| Border | `#27272A` (zinc-800) | `#E4E4E7` (zinc-200) |
| Text | `#FAFAFA` (zinc-50) | `#09090B` (zinc-950) |
| Muted | `#A1A1AA` (zinc-400) | `#71717A` (zinc-500) |

### 8.3 Typography

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Headings (h1-h3) | Inter | 24-36px | 700 (bold) |
| Body text | Inter | 16px | 400 (regular) |
| Labels | Inter | 14px | 500 (medium) |
| Captions | Inter | 12px | 400 (regular) |
| Code | JetBrains Mono | 14px | 400 (regular) |

---

## 9. Animation & Transition Guidelines

### 9.1 Animation Principles

| Principle | Rule |
|-----------|------|
| **Purposeful** | Every animation communicates state change |
| **Fast** | 150-300ms for UI transitions |
| **Subtle** | Fade, slide, scale only |
| **Respectful** | Disabled when `prefers-reduced-motion` |

### 9.2 Transition Catalog

| Interaction | Animation | Duration | Easing |
|-------------|-----------|----------|--------|
| **Hover state** | Background color | 150ms | `ease-out` |
| **Click/tap** | Press scale (0.98) | 100ms | `ease-out` |
| **Focus ring** | Border appear | 0ms | Instant |
| **Modal open** | Fade + scale up | 200ms | `ease-out` |
| **Modal close** | Fade + scale down | 150ms | `ease-in` |
| **Tab switch** | Slide content | 200ms | `ease-out` |
| **Upload complete** | Thumbnail pop-in | 300ms | `ease-out` |
| **Success state** | Green fade + checkmark | 400ms | `ease-out` |
| **Error state** | Red shake + appear | 300ms | `ease-in-out` |
| **Progress bar update** | Width animate | 300ms | `ease-out` |

### 9.3 Micro-interactions

1. **Drag over dropzone** — Border dashed, subtle background glow
2. **File added** — Thumbnail slides in from bottom
3. **Submit hover** — Button lifts slightly (box-shadow)
4. **Result complete** — Image fades in with soft glow
5. **Copy to clipboard** — Brief tooltip "Copied!" with checkmark

---

## 10. Responsive Image Handling

### 10.1 srcset Strategy

```html
<img 
  srcset="result-480.webp 480w,
          result-768.webp 768w,
          result-1024.webp 1024w"
  sizes="(min-width: 1024px) 50vw, 100vw"
  src="result-1024.webp"
  alt="Generated image"
  loading="lazy"
/>
```

### 10.2 Thumbnail Generation

| Type | Size | Format | Use |
|------|------|--------|-----|
| Thumbnail | 120x120 | WebP | Upload preview grid |
| Preview | 512x512 | WebP | Result display (before download) |
| Full size | 1024x1024 | PNG | Download |

---

## 11. References

| Reference | Description |
|-----------|-------------|
| [SDD-001](./SDD-001-OVERVIEW.md) | Project overview |
| [SDD-003](./SDD-003-FEATURES.md) | Features catalog |
| [WCAG 2.1 AA](https://www.w3.org/WAI/WCAG2AA-Conformance) | Accessibility standard |
| [Tailwind CSS](https://tailwindcss.com/) | Styling framework |
| [Radix UI](https://www.radix-ui.com/) | Accessible primitives |

---

## 12. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-04-21 | VisionLab Team | Initial UI/UX specification |