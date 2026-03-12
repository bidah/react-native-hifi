---
name: use-dom
description: Use when embedding web-only libraries or DOM content in React Native apps, including charts with recharts, syntax highlighting, rich text editors, or any component that requires a browser DOM
---

# DOM Components in React Native ('use dom')

## Overview

The `'use dom'` directive lets you embed web code inside React Native apps. Components marked with `'use dom'` render in a native webview (WKWebView on iOS, WebView on Android), giving you access to the full browser DOM while keeping the rest of your app native.

**Core principle:** Use `'use dom'` as a bridge to web-only libraries that have no React Native equivalent. The DOM component runs in an isolated webview, communicating with your native app through serializable props.

## When to Use

- Rendering charts with web libraries (recharts, Chart.js, D3)
- Syntax highlighting code blocks (Prism, Shiki, highlight.js)
- Rich text editors (TipTap, Lexical, Quill)
- Embedding maps with web map libraries (Leaflet, Mapbox GL JS)
- Rendering complex HTML/CSS content (markdown, documentation)
- Using any npm package that depends on browser APIs (DOM, Canvas, WebGL)

### When NOT to Use

- Simple UI components (use React Native components)
- Performance-critical animations (use Reanimated)
- Components needing native device APIs (camera, sensors)
- Content that could be rendered with React Native `Text`/`View`

## Basic Usage

Add `'use dom'` as the first line of a component file:

```tsx
// components/WebChart.tsx
'use dom';

import { BarChart, Bar, XAxis, YAxis, Tooltip } from 'recharts';

type Props = {
  data: Array<{ name: string; value: number }>;
  dom?: import('expo/dom').DOMProps;
};

export default function WebChart({ data }: Props) {
  return (
    <BarChart width={350} height={250} data={data}>
      <XAxis dataKey="name" />
      <YAxis />
      <Tooltip />
      <Bar dataKey="value" fill="#3b82f6" />
    </BarChart>
  );
}
```

Use it in your native React Native screen:

```tsx
// app/analytics.tsx
import WebChart from '../components/WebChart';

const chartData = [
  { name: 'Mon', value: 120 },
  { name: 'Tue', value: 250 },
  { name: 'Wed', value: 180 },
  { name: 'Thu', value: 310 },
  { name: 'Fri', value: 420 },
];

export default function AnalyticsScreen() {
  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Text style={{ fontSize: 20, fontWeight: 'bold', marginBottom: 16 }}>
        Weekly Activity
      </Text>
      <WebChart data={chartData} />
    </View>
  );
}
```

## The DOMProps Type

Every `'use dom'` component receives an optional `dom` prop for controlling the webview:

```tsx
'use dom';

type Props = {
  content: string;
  dom?: import('expo/dom').DOMProps;
};

export default function MyDomComponent({ content }: Props) {
  return <div>{content}</div>;
}
```

The `dom` prop is injected automatically by the framework. You do not pass it manually. Include it in the type definition so TypeScript knows about it.

### DOMProps Options

```tsx
// In your native component:
<WebChart
  data={data}
  dom={{
    scrollEnabled: true,       // Allow scrolling within the webview
    matchContents: true,       // Auto-size webview to content height
  }}
/>
```

## Serializable Props Requirement

All props passed to `'use dom'` components must be JSON-serializable. The props cross a native-to-webview bridge.

### What Works

```tsx
// Primitives
<DomComponent count={42} />
<DomComponent label="hello" />
<DomComponent enabled={true} />

// Plain objects and arrays
<DomComponent data={[{ x: 1, y: 2 }]} />
<DomComponent config={{ theme: 'dark', fontSize: 14 }} />

// Callback functions (serialized as message handlers)
<DomComponent onSelect={(item: string) => console.log(item)} />
```

### What Does NOT Work

```tsx
// React elements
<DomComponent header={<Text>Title</Text>} />  // Cannot serialize JSX

// Class instances
<DomComponent date={new Date()} />  // Send timestamp number instead

// Refs
<DomComponent ref={myRef} />  // Use dom prop for webview control

// Functions with closures over non-serializable data
<DomComponent fn={() => nativeModule.doThing()} />  // nativeModule not available in webview
```

### Workaround for Complex Data

```tsx
// Convert non-serializable to serializable
<DomComponent
  timestamp={Date.now()}           // Instead of Date object
  items={JSON.stringify(mapData)}  // If needed
  onAction={(action: string) => {  // Callback with serializable args
    handleAction(action);
  }}
/>
```

## Use Cases with Examples

### Syntax Highlighting

```tsx
// components/CodeBlock.tsx
'use dom';

import { useEffect, useRef } from 'react';
import hljs from 'highlight.js';
import 'highlight.js/styles/github-dark.css';

type Props = {
  code: string;
  language: string;
  dom?: import('expo/dom').DOMProps;
};

export default function CodeBlock({ code, language }: Props) {
  const codeRef = useRef<HTMLElement>(null);

  useEffect(() => {
    if (codeRef.current) {
      hljs.highlightElement(codeRef.current);
    }
  }, [code]);

  return (
    <pre style={{ margin: 0, borderRadius: 8, overflow: 'auto' }}>
      <code ref={codeRef} className={`language-${language}`}>
        {code}
      </code>
    </pre>
  );
}
```

### Rich Text Editor

```tsx
// components/RichEditor.tsx
'use dom';

import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';

type Props = {
  initialContent: string;
  onChange: (html: string) => void;
  dom?: import('expo/dom').DOMProps;
};

export default function RichEditor({ initialContent, onChange }: Props) {
  const editor = useEditor({
    extensions: [StarterKit],
    content: initialContent,
    onUpdate: ({ editor }) => {
      onChange(editor.getHTML());
    },
  });

  return (
    <div style={{ border: '1px solid #ddd', borderRadius: 8, padding: 12 }}>
      <div style={{ borderBottom: '1px solid #eee', paddingBottom: 8, marginBottom: 8 }}>
        <button onClick={() => editor?.chain().focus().toggleBold().run()}>
          <strong>B</strong>
        </button>
        <button onClick={() => editor?.chain().focus().toggleItalic().run()}>
          <em>I</em>
        </button>
      </div>
      <EditorContent editor={editor} />
    </div>
  );
}
```

### Interactive Map

```tsx
// components/WebMap.tsx
'use dom';

import { MapContainer, TileLayer, Marker, Popup } from 'react-leaflet';
import 'leaflet/dist/leaflet.css';

type Props = {
  markers: Array<{ lat: number; lng: number; label: string }>;
  center: { lat: number; lng: number };
  zoom: number;
  onMarkerClick: (label: string) => void;
  dom?: import('expo/dom').DOMProps;
};

export default function WebMap({ markers, center, zoom, onMarkerClick }: Props) {
  return (
    <MapContainer
      center={[center.lat, center.lng]}
      zoom={zoom}
      style={{ height: '100%', width: '100%' }}
    >
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
      {markers.map((m, i) => (
        <Marker
          key={i}
          position={[m.lat, m.lng]}
          eventHandlers={{ click: () => onMarkerClick(m.label) }}
        >
          <Popup>{m.label}</Popup>
        </Marker>
      ))}
    </MapContainer>
  );
}
```

## Host Wrapping

When a `'use dom'` component needs to be wrapped in native views for layout purposes:

```tsx
// Native screen
import WebChart from '../components/WebChart';

export default function DashboardScreen() {
  return (
    <ScrollView style={{ flex: 1 }}>
      <View style={{ height: 300, marginHorizontal: 16 }}>
        <WebChart data={salesData} />
      </View>

      <View style={{ height: 400, marginHorizontal: 16, marginTop: 16 }}>
        <WebMap markers={locations} center={center} zoom={12} />
      </View>
    </ScrollView>
  );
}
```

Set explicit dimensions on the parent `View` since the webview needs a known frame to render into.

## Performance Considerations

- Each `'use dom'` component creates a webview instance (heavyweight)
- Limit to 2-3 DOM components per screen for good performance
- Use `matchContents` to avoid unnecessary scrolling within the webview
- Prefer React Native components for simple UI; reserve DOM for web-only libraries
- DOM components have a small render delay on first mount (webview initialization)
- Avoid frequent prop updates; batch changes when possible

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `'use dom'` directive | Must be the very first line of the file, before all imports |
| Passing non-serializable props | Convert to primitives, plain objects, or arrays |
| Not setting parent View dimensions | Webview needs explicit `width`/`height` from parent layout |
| Using too many DOM components on one screen | Keep to 2-3 max; combine related web content into one component |
| Trying to access React Native APIs inside DOM component | DOM components run in a browser context; no access to `NativeModules` |
| Missing `dom` prop in type definition | Include `dom?: import('expo/dom').DOMProps` in your Props type |
| Using DOM for simple text/layout | Only use for web-only libraries; native components are faster |

## Quick Reference

| Task | Pattern |
|------|---------|
| Create DOM component | Add `'use dom'` as first line of component file |
| Type the dom prop | `dom?: import('expo/dom').DOMProps` |
| Auto-size to content | `dom={{ matchContents: true }}` |
| Enable scrolling | `dom={{ scrollEnabled: true }}` |
| Pass data | Only serializable props (primitives, objects, arrays) |
| Pass callbacks | Functions with serializable arguments work |
| Set dimensions | Wrap in `<View style={{ height: N }}>` |
| Use web CSS | Standard CSS imports work inside DOM components |
| Use web libraries | Any npm package with DOM deps works inside `'use dom'` |
