# Technical Stories: Web (SPA)

Stories técnicas específicas para proyectos Web SPA. Complementan las stories de `_base.md`.

> **Stack típico:** Frontend SPA (React/Vue/Angular) + Backend API (.NET/Node/Python)

## Stories

```yaml
stories:
  - ref: "web-skeleton"
    title: "Walking skeleton - Web SPA end-to-end"
    effort: 3
    area: "walking-skeleton"
    description: |
      As a development team,
      I want a minimal vertical slice deployed,
      so that we validate the architecture works end-to-end before building features.
    acceptanceCriteria:
      - "Frontend SPA loads in browser at public URL"
      - "Frontend calls backend API endpoint"
      - "API returns data that frontend displays"
      - "Works in deployed environment (not just localhost)"
    verifiableValue: "Visit URL → see 'Hello World' from API displayed in UI"
    priority: "High"
    notes: "This should be the FIRST story completed"

  - ref: "web-responsive"
    title: "Responsive layout foundation"
    effort: 2
    area: "ux"
    description: |
      As a user,
      I want the app to adapt to my screen size,
      so that I can use it comfortably on desktop, tablet, or mobile browser.
    acceptanceCriteria:
      - "Layout adapts to viewport widths: mobile (<768px), tablet (768-1024px), desktop (>1024px)"
      - "No horizontal scroll on any breakpoint"
      - "Touch targets minimum 44x44px on mobile"
      - "Text remains readable without zooming"
    verifiableValue: "Resize browser → layout adapts smoothly"

  - ref: "web-browsers"
    title: "Cross-browser compatibility"
    effort: 1
    area: "ux"
    description: |
      As a user,
      I want the app to work in my preferred browser,
      so that I don't have to switch browsers to use it.
    acceptanceCriteria:
      - "Works in Chrome (last 2 versions)"
      - "Works in Firefox (last 2 versions)"
      - "Works in Safari (last 2 versions)"
      - "Works in Edge (last 2 versions)"
      - "Graceful degradation for unsupported features"
    verifiableValue: "Test core flow in each browser → all work"
    notes: "Include in Definition of Done for all stories"

  - ref: "web-navigation"
    title: "Navigation layout and menu"
    effort: 2
    area: "ux"
    description: |
      As a user,
      I want clear navigation,
      so that I can find features easily without getting lost.
    acceptanceCriteria:
      - "Consistent header/navigation across all pages"
      - "Current page/section visually indicated"
      - "Mobile: hamburger menu or bottom navigation"
      - "Desktop: visible navigation bar"
      - "Navigation preserves URL for bookmarking/sharing"
    verifiableValue: "Navigate app → always know where you are and how to get elsewhere"
    dependsOn: ["web-skeleton"]

  - ref: "web-i18n"
    title: "Internationalization foundation"
    effort: 3
    area: "ux"
    description: |
      As a user,
      I want the app in my language,
      so that I can understand all content without translation.
    acceptanceCriteria:
      - "i18n library integrated (react-intl/vue-i18n/ngx-translate)"
      - "All UI strings externalized to translation files"
      - "Language switcher or auto-detection from browser"
      - "At least 2 languages supported (e.g., en, es)"
      - "Date/number formats respect locale"
    verifiableValue: "Switch language → all UI text changes"
    notes: "Can be deferred if single-language MVP is acceptable"
    optional: true
```

## Architecture Notes

- SPA routing should use history API (not hash routing)
- API should be behind reverse proxy or API gateway
- Consider CDN for static assets
- Implement proper CORS configuration between frontend and API

## Typical Epic Structure

```
Epic: Technical Infrastructure (Web)
├── Walking skeleton - Web SPA end-to-end
├── [from _base] Infrastructure as Code setup
├── [from _base] Basic CI/CD pipeline
├── [from _base] Tests gate in CI pipeline
├── [from _base] Basic telemetry and error tracking
├── [from _base] Distributed tracing setup
├── [from _base] User authentication
├── [from _base] Role-based authorization
├── Responsive layout foundation
├── Cross-browser compatibility
├── Navigation layout and menu
└── Internationalization foundation (optional)
```
