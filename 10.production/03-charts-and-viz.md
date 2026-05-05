# Production — 03. Charts & Viz: Recharts 2 vs ECharts vs visx vs Nivo

> **What / Why / How** — Recharts for dashboards, ECharts for serious viz, visx when you need D3 power without rendering D3, Tremor / shadcn-charts for "looks-right-now" with zero design work.

---

## 1. The Real Choices in 2026

| Library | npm | Engine | Bundle gzipped | Best for |
|---------|-----|--------|----------------|----------|
| **Recharts 2.12** | `recharts@2.12` | SVG, opinionated React API | ~95 KB (tree-shakes to ~30 KB per chart type) | Dashboards, the **shadcn/ui Charts** default |
| **shadcn/ui Charts** (Recharts wrapper) | copy-paste from CLI | Recharts under the hood | Same as Recharts | "I want a chart that looks polished and matches my Tailwind theme right now" |
| **Tremor 3** | `@tremor/react@3.18` | Recharts under the hood | ~150 KB (full library) | Drop-in dashboard kit (KPI cards, bar lists, area charts) |
| **ECharts 5** + `echarts-for-react@3` | `echarts@5.5`, `echarts-for-react@3.0` | Canvas (default) or SVG | ~200 KB (~80 KB if tree-shaken to needed series) | Heavy data viz, financial charts, large datasets, full feature set |
| **visx 3** | `@visx/group@3`, `@visx/scale@3`, `@visx/shape@3` | SVG, low-level D3 + React primitives | Per-package (~5–15 KB each) | Custom visualizations where you'd otherwise use D3 directly |
| **Nivo 0.87** | `@nivo/core@0.87`, `@nivo/bar@0.87`, etc. | SVG (some Canvas variants) | ~50–80 KB per chart type | Beautiful defaults, motion-rich charts |
| **Apache ECharts (raw)** | `echarts@5.5` | Canvas | Same as above | Don't use raw — wrap with `echarts-for-react` |
| **Chart.js 4** + `react-chartjs-2@5` | `chart.js@4`, `react-chartjs-2@5` | Canvas | ~85 KB | Simple charts on legacy projects |
| **D3 7** directly | `d3@7` | SVG via you | Per-module (~50 KB total typical) | When visx isn't custom enough |
| **Plotly 2** | `react-plotly.js@2` | SVG/Canvas | ~3 MB (huge) | Scientific viz with built-in zoom/pan/hover; cost is bundle size |
| **Vega-Lite 5 + react-vega 7** | `react-vega@7` | SVG/Canvas | ~150 KB | Grammar-of-graphics, declarative spec |

**The 80% rule for new React/Next.js apps in 2026:**
- Polished dashboard charts → **shadcn/ui Charts** (Recharts under the hood).
- Custom design or unusual chart types → **visx 3**.
- 50K+ data points or interactive financial-style charts → **`echarts-for-react@3`**.
- Drop-in dashboard kit with metrics + charts → **Tremor 3** (or shadcn/ui Charts now that they overlap).

Skip Chart.js for new code. Skip raw D3 unless you've measured that visx isn't expressive enough.

---

## 2. The Mental Split — Three Categories of "Charts"

Different libraries are good at different shapes:

| Category | What it looks like | Right tool |
|----------|---------------------|------------|
| **Dashboard charts** | KPIs, line/area/bar/donut, < 1k points, polished defaults | shadcn/ui Charts, Tremor 3, Nivo |
| **Custom visualization** | Network graph, custom map, treemap with weird interactions, story-driven | visx 3, D3 7 |
| **Data-heavy / interactive** | 50k+ points, zoom/pan, financial candlesticks, real-time tickers | ECharts 5, Plotly |

Pick the category first. Within the category, the libraries are interchangeable enough that team familiarity matters more than micro-benchmarks.

---

## 3. shadcn/ui Charts — The 2026 Default for Dashboards

### What

The `shadcn/ui` family added a **`charts`** module in late 2024. It's a thin styled wrapper around Recharts 2.12 that ships:
- Theming via the same CSS variables as the rest of shadcn (`--chart-1`, `--chart-2`, ..., `--chart-5`).
- A `<ChartContainer>` that handles tooltip, legend, and responsive sizing.
- A `<ChartTooltip>` with proper accessibility.
- Copy-paste source files like every other shadcn component (covered in `08.ecosystem/02-ui-libraries.md`).

### Setup

```bash
npx shadcn@2.1 add chart
```

Adds `components/ui/chart.tsx` and updates `globals.css` with the `--chart-*` variables. Then any Recharts chart wrapped in `<ChartContainer>` automatically picks up your theme.

### Real example — area chart for a dashboard

```tsx
'use client';
import { Area, AreaChart, CartesianGrid, XAxis } from 'recharts';
import { ChartContainer, ChartTooltip, ChartTooltipContent, type ChartConfig } from '@/components/ui/chart';

const data = [
  { month: 'Jan', revenue: 18600, refunds: 800 },
  { month: 'Feb', revenue: 22500, refunds: 600 },
  { month: 'Mar', revenue: 28000, refunds: 1100 },
  { month: 'Apr', revenue: 30500, refunds: 950 },
  { month: 'May', revenue: 34200, refunds: 1300 },
  { month: 'Jun', revenue: 39800, refunds: 1450 },
];

const chartConfig = {
  revenue: { label: 'Revenue', color: 'hsl(var(--chart-1))' },
  refunds: { label: 'Refunds', color: 'hsl(var(--chart-2))' },
} satisfies ChartConfig;

export function RevenueChart() {
  return (
    <ChartContainer config={chartConfig} className="aspect-video w-full">
      <AreaChart data={data}>
        <CartesianGrid vertical={false} />
        <XAxis dataKey="month" tickLine={false} axisLine={false} />
        <ChartTooltip content={<ChartTooltipContent />} />
        <Area type="monotone" dataKey="revenue" stroke="var(--color-revenue)" fill="var(--color-revenue)" fillOpacity={0.2} />
        <Area type="monotone" dataKey="refunds" stroke="var(--color-refunds)" fill="var(--color-refunds)" fillOpacity={0.2} />
      </AreaChart>
    </ChartContainer>
  );
}
```

### Why shadcn/ui Charts won as the default

- **Looks right out of the box.** No manually picking colors, no fighting tooltip styles to match the rest of the app.
- **Dark mode just works.** The `--chart-*` variables flip with the rest of the theme.
- **Same ownership model** as the rest of shadcn/ui — the chart component lives in your repo; modify it freely.
- **AI-friendly.** v0.dev and Cursor reliably output shadcn/ui Charts code.
- **Free.** Tremor 3 is also free but ships an opinionated pre-built kit; shadcn lets you compose primitives.

### When shadcn/ui Charts loses

- Dataset > 5K points → Recharts SVG rendering gets slow. Switch to ECharts (Canvas).
- Need uncommon chart types (Sankey, Sunburst, Network graph) → not in Recharts. Use ECharts or visx.

---

## 4. Recharts 2.12 — The Engine Underneath

If you're not using shadcn/ui Charts directly, raw Recharts is still excellent:

```tsx
import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer, CartesianGrid } from 'recharts';

<ResponsiveContainer width="100%" height={300}>
  <LineChart data={data}>
    <CartesianGrid strokeDasharray="3 3" />
    <XAxis dataKey="month" />
    <YAxis />
    <Tooltip />
    <Line type="monotone" dataKey="users" stroke="#2563eb" strokeWidth={2} />
  </LineChart>
</ResponsiveContainer>
```

### Why Recharts won the 2018–2026 dashboard race

- **Composable React API** (each axis, line, tooltip is a real React component).
- **Tree-shakeable.** Importing `LineChart` only pulls in line + axes, not all chart types.
- **Responsive container** that resizes on window changes.
- **Tooltip and legend customization** through React props, not config objects.
- Mature enough that every dashboard template (Vercel's analytics dashboard, the shadcn/ui demos, Tremor 3) builds on it.

### Why raw Recharts (without shadcn) loses today

- Theming is verbose. Every chart needs explicit colors. shadcn/ui Charts solves this.
- Tooltips require styling work. Again, shadcn/ui Charts solves this.

For new code: prefer the shadcn wrapper.

---

## 5. Tremor 3 — Drop-in Dashboard Kit

`@tremor/react@3.18` is a Tailwind-styled dashboard kit. It's built on Recharts but ships full components: KPI cards, badge deltas, bar lists, area/line/bar/donut charts, callouts.

```bash
npm i @tremor/react@3.18
```

```tsx
import { Card, Title, AreaChart, BarList } from '@tremor/react';

<Card>
  <Title>Monthly revenue</Title>
  <AreaChart
    className="h-72"
    data={data}
    index="month"
    categories={['revenue', 'refunds']}
    colors={['blue', 'red']}
    valueFormatter={(n) => `$${n.toLocaleString()}`}
  />
</Card>
```

### When Tremor still wins

- You want a complete dashboard kit (cards, badges, lists, charts) with one install.
- Time-to-first-pixel matters more than fine-grained customization.

### When Tremor loses to shadcn/ui Charts

- Tremor's design system is its own (Tailwind-based but not aligned with shadcn). Mixing Tremor + shadcn/ui in one app produces visual mismatch.
- shadcn/ui Charts now covers the same chart types with better integration.

For projects already on shadcn/ui: stick with shadcn Charts. Pick Tremor only for greenfield admin dashboards where you want it all turn-key.

---

## 6. ECharts 5 — When Data Gets Serious

`echarts@5.5` is Apache's flagship charting library — battle-tested in Chinese fintech and enterprise dashboards. The React wrapper is `echarts-for-react@3`.

### Why ECharts wins for heavy charts

- **Canvas rendering** by default — handles 100K+ points smoothly. SVG-based libs (Recharts, Nivo, visx) start to struggle past 5K.
- **Massive built-in chart catalog**: candlestick, sankey, treemap, sunburst, parallel-coordinates, graph (network), gauge, funnel, heatmap, pictorial, themeRiver, ~30 types.
- **Built-in toolbox**: zoom, dataView, restore, save-as-image — config flag, not custom code.
- **Built-in dataZoom slider** — drag a window to zoom into a date range. Standard for financial charts.
- **Server-side rendering to PNG/SVG** via `echarts/lib/echarts` in Node.

### Setup

```bash
npm i echarts@5.5 echarts-for-react@3
```

```tsx
'use client';
import ReactECharts from 'echarts-for-react';

export function StockChart({ data }: { data: { date: string; open: number; close: number; low: number; high: number }[] }) {
  const option = {
    xAxis: { type: 'category', data: data.map(d => d.date) },
    yAxis: { type: 'value', scale: true },
    dataZoom: [{ type: 'inside' }, { type: 'slider' }],
    tooltip: { trigger: 'axis' },
    series: [{
      type: 'candlestick',
      data: data.map(d => [d.open, d.close, d.low, d.high]),
      itemStyle: { color: '#16a34a', color0: '#dc2626' },
    }],
  };
  return <ReactECharts option={option} style={{ height: 400 }} notMerge lazyUpdate />;
}
```

### Tree-shaking ECharts (real concern)

Default `import ReactECharts from 'echarts-for-react'` pulls in the full ECharts (~200 KB gzipped). Tree-shake to what you actually use:

```ts
// echarts-setup.ts
import * as echarts from 'echarts/core';
import { CandlestickChart, LineChart } from 'echarts/charts';
import { GridComponent, TooltipComponent, DataZoomComponent } from 'echarts/components';
import { CanvasRenderer } from 'echarts/renderers';

echarts.use([CandlestickChart, LineChart, GridComponent, TooltipComponent, DataZoomComponent, CanvasRenderer]);

import ReactEChartsCore from 'echarts-for-react/lib/core';
export const Chart = (props: any) => <ReactEChartsCore echarts={echarts} {...props} />;
```

This drops the bundle to ~80 KB for typical chart sets.

### When NOT to pick ECharts

- Standard dashboard with < 5K points → shadcn/ui Charts is simpler and shows better next to the rest of your app.
- You want React-component-style API → ECharts is config-object-driven; it'll feel non-React.

---

## 7. visx 3 — D3 Power Without Rendering D3

`@visx/*@3` is Airbnb's "expressive low-level visualization primitives for React." It pairs **D3's math** (scales, shapes, hierarchies) with **React's rendering**.

You don't import a `<LineChart>`. You import scales, axes, lines, markers and compose them.

```bash
npm i @visx/group@3 @visx/shape@3 @visx/scale@3 @visx/axis@3 @visx/responsive@3
```

```tsx
'use client';
import { Group } from '@visx/group';
import { LinePath } from '@visx/shape';
import { scaleLinear, scaleTime } from '@visx/scale';
import { AxisLeft, AxisBottom } from '@visx/axis';
import { ParentSize } from '@visx/responsive';

type Point = { date: Date; value: number };

function Chart({ data, width, height }: { data: Point[]; width: number; height: number }) {
  const margin = { top: 16, right: 24, bottom: 32, left: 48 };
  const innerW = width - margin.left - margin.right;
  const innerH = height - margin.top - margin.bottom;

  const xScale = scaleTime({
    range: [0, innerW],
    domain: [Math.min(...data.map(d => +d.date)), Math.max(...data.map(d => +d.date))],
  });
  const yScale = scaleLinear({
    range: [innerH, 0],
    domain: [0, Math.max(...data.map(d => d.value))],
    nice: true,
  });

  return (
    <svg width={width} height={height}>
      <Group left={margin.left} top={margin.top}>
        <AxisLeft scale={yScale} />
        <AxisBottom scale={xScale} top={innerH} />
        <LinePath
          data={data}
          x={(d) => xScale(d.date) ?? 0}
          y={(d) => yScale(d.value) ?? 0}
          stroke="#2563eb"
          strokeWidth={2}
        />
      </Group>
    </svg>
  );
}

export function ResponsiveChart({ data }: { data: Point[] }) {
  return (
    <ParentSize>
      {({ width, height }) => <Chart data={data} width={width} height={Math.max(height, 300)} />}
    </ParentSize>
  );
}
```

### Why visx

- **Each module is small** (5–15 KB) and tree-shakeable. Your bundle reflects only what you use.
- **Full D3 power** for scales, hierarchies (treemap, partition, pack, tree), interpolators, force layouts.
- **You render with React**, so you can use any animation library, any state, any composition pattern.
- **No black boxes** — when a chart doesn't look right, you can debug it because you wrote every element.

### When visx wins

- Custom chart that doesn't fit Recharts' opinions (e.g., chord diagram, calendar heatmap, custom hierarchy).
- You want fine control over animation (pair with `framer-motion@11`, covered in `08.ecosystem/01-animation.md`).
- You'd otherwise use D3 directly — visx is a strict upgrade because React owns the DOM.

### When visx loses

- You want a chart in 5 lines. visx is composition; the simplest line chart is ~30 lines.
- The chart is "give me a bar chart with a tooltip." That's Recharts territory.

---

## 8. Nivo 0.87 — Beautiful Defaults, Motion-Rich

`@nivo/core@0.87` and the dozen `@nivo/<chart-type>` packages ship beautiful default-styled charts with smooth animations.

```bash
npm i @nivo/core@0.87 @nivo/bar@0.87
```

```tsx
'use client';
import { ResponsiveBar } from '@nivo/bar';

<div style={{ height: 400 }}>
  <ResponsiveBar
    data={[
      { country: 'US', sales: 480 },
      { country: 'UK', sales: 380 },
      { country: 'JP', sales: 220 },
    ]}
    keys={['sales']}
    indexBy="country"
    margin={{ top: 50, right: 30, bottom: 50, left: 60 }}
    padding={0.3}
    colors={{ scheme: 'category10' }}
    animate
  />
</div>
```

### When Nivo wins

- You want polished, animated charts and don't want to design them yourself.
- Time-series with smooth transitions when data updates.

### When Nivo loses

- shadcn/ui Charts is closer to "modern Tailwind dashboard" aesthetics by 2026.
- Per-chart bundle is heavier than tree-shaken Recharts.

For new projects: pick shadcn/ui Charts unless you specifically want Nivo's motion defaults.

---

## 9. Real Decision Cases

### "I'm building a SaaS analytics dashboard."
→ **shadcn/ui Charts** (or Tremor 3 if you want a full kit). Maximum polish, minimum work.

### "I need a candlestick chart with zoom/pan for a trading UI."
→ **`echarts-for-react@3`** with tree-shaken imports. Recharts can't do candlestick well; visx is too low-level for this scale.

### "I'm visualizing a network of 500 nodes."
→ **`@visx/network@3`** + force layout, OR ECharts' `graph` series. visx if you need custom rendering per node; ECharts if you just want it to work.

### "I need a calendar heatmap (GitHub contribution-style)."
→ **`@visx/heatmap@3`**, or `react-calendar-heatmap@1` if you want a one-import answer.

### "Server-side render a chart to a PNG for an OG image."
→ ECharts via `echarts/lib/echarts` in Node, or render an SVG with visx and use `next/og` `ImageResponse` (covered in `07.nextjs/07-seo-metadata.md`).

### "Real-time streaming chart updating 60 times/sec."
→ **ECharts** (Canvas) with `notMerge: true, lazyUpdate: true`. Recharts SVG can't keep up at 60Hz.

### "I'm just rendering one sparkline next to a number."
→ Tiny SVG by hand, `react-sparklines@1`, or shadcn/ui Charts' minimal area chart with axes hidden.

---

## 10. Performance — Real Numbers

| Scenario | Recharts | ECharts | visx | Nivo |
|----------|----------|---------|------|------|
| 100 points, line | <1ms render | <1ms | <1ms | <1ms |
| 1K points, line | ~5ms | ~3ms | ~5ms | ~7ms |
| 10K points, line | ~80ms (laggy on hover) | ~5ms (Canvas) | ~80ms | ~120ms |
| 100K points, line | Often hangs | ~30ms (Canvas) | DIY downsample needed | Hangs |
| Tooltip hover at 60fps | ✅ | ✅ | ✅ | ⚠ slight jitter |

**Rule of thumb**: SVG-based libs (Recharts, visx, Nivo) are smooth up to ~5K data points. Above that, switch to Canvas (ECharts) or downsample. `largestTriangleThreeBuckets` (LTTB) from `downsample-lttb@1` is the standard algorithm.

---

## 11. Real Production Touches

### Always responsive

```tsx
<ResponsiveContainer width="100%" height={300}>...</ResponsiveContainer>  // Recharts
<ResponsiveBar ... />                                                       // Nivo
<ParentSize>{({ width, height }) => ...}</ParentSize>                       // visx
<ReactECharts style={{ height: 300 }} />                                    // ECharts (auto-resizes)
```

For SSR (Next.js): the chart needs a client boundary. Wrap in `'use client'` and dynamic import if it's heavy:

```tsx
import dynamic from 'next/dynamic';
const Chart = dynamic(() => import('./RevenueChart'), { ssr: false });
```

### Empty states

A chart with no data is the #1 source of confused users. Always:

```tsx
{data.length === 0 ? <EmptyState message="No data for this range" /> : <RevenueChart data={data} />}
```

### Color palettes

For accessibility (color-blind friendliness):
- Don't use red+green only as a difference. Use blue+orange or shape differences.
- The shadcn `--chart-1` through `--chart-5` defaults are designed for this.
- For 5+ series, switch to a categorical palette like `colors={{ scheme: 'category10' }}` (Nivo) or D3's `d3-scale-chromatic@3` schemes.

### Dark mode

| Library | Dark mode pattern |
|---------|-------------------|
| shadcn/ui Charts | Free — `--chart-*` variables flip with the theme |
| Recharts (raw) | Pass `stroke` / `fill` from CSS variables |
| ECharts | `option.theme = 'dark'` or roll your own theme object |
| Nivo | `theme={{ ... }}` prop with text/grid/tooltip colors |
| visx | Pass colors as props from your theme |

### Animations on data change

- shadcn/ui Charts / Recharts: `<Line isAnimationActive />` (default true).
- ECharts: `animation: true` in `option`.
- Nivo: `animate` prop.
- visx: pair with `framer-motion@11` `<motion.path>` for control.

---

## 12. Tooltips — Real Patterns

The default tooltip is rarely good enough. Custom-content tooltips are the most-asked feature in every chart library.

### shadcn/ui Charts custom tooltip

```tsx
import { ChartTooltip, ChartTooltipContent } from '@/components/ui/chart';

<ChartTooltip
  content={
    <ChartTooltipContent
      formatter={(value, name, item) => (
        <>
          <span className="font-medium">{name}</span>
          <span>{`$${(value as number).toLocaleString()}`}</span>
          <span className="text-muted-foreground">{item.payload.month}</span>
        </>
      )}
    />
  }
/>
```

### ECharts custom tooltip

```ts
const option = {
  tooltip: {
    trigger: 'axis',
    formatter: (params: any[]) =>
      params.map(p => `${p.seriesName}: $${p.value.toLocaleString()}`).join('<br/>'),
  },
};
```

### visx — you own the tooltip from scratch

```tsx
import { useTooltip, TooltipWithBounds } from '@visx/tooltip';
const { tooltipData, tooltipLeft, tooltipTop, showTooltip, hideTooltip } = useTooltip<Point>();

<rect
  onMouseMove={(e) => showTooltip({ tooltipData: point, tooltipLeft: e.clientX, tooltipTop: e.clientY })}
  onMouseLeave={hideTooltip}
/>
{tooltipData && (
  <TooltipWithBounds left={tooltipLeft} top={tooltipTop}>
    {tooltipData.value}
  </TooltipWithBounds>
)}
```

---

## 13. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Bundle 200+ KB for one chart (ECharts default import) | Use the tree-shaking pattern in §6 with `echarts-for-react/lib/core` |
| Recharts laggy past 5K points | Downsample with `largestTriangleThreeBuckets` or switch to ECharts |
| Tooltip text mismatches dark mode | shadcn/ui Charts solves it; manual chart needs CSS-variable colors |
| Server renders an empty SVG | Charts need `'use client'`; or render server-side via the library's headless path |
| Y-axis cuts off labels | Increase left margin (`margin={{ left: 60 }}`) or use a number formatter that produces shorter strings |
| ResponsiveContainer reports `0×0` initially | The parent must have a defined height. `ResponsiveContainer` uses 100% of parent, which is `0` if unset |
| Two ResponsiveContainers re-rendering each other infinitely | Don't nest them; one per chart |
| ECharts doesn't update after data change | Pass `notMerge` and `lazyUpdate`, or pass a new `option` object reference |
| Numbers in tooltip show 12 decimals | Format with `Intl.NumberFormat` or `valueFormatter` prop |
| Animation on every data refetch is jarring | Set `isAnimationActive={false}` for charts that update frequently (real-time) |

---

## 14. Decision Tree

```
What kind of chart UI?
│
├── Polished SaaS dashboard (KPIs, area, bar, donut, line)
│   → shadcn/ui Charts (Recharts under the hood). Default for 2026.
│
├── Drop-in dashboard kit with cards + charts + lists
│   → Tremor 3, OR shadcn/ui Charts + shadcn/ui Cards
│
├── Heavy interactive charts (50K+ points, candlesticks, financial UIs)
│   → echarts-for-react@3 with tree-shaken imports
│
├── Custom chart type not covered by Recharts (network, sankey, treemap, calendar heatmap)
│   → @visx/* if you want React composition; ECharts if you just want it built-in
│
├── Highly motion-rich animated charts
│   → Nivo 0.87 (built-in animations) or visx + framer-motion@11
│
├── Need to render charts on the server (PNG for OG image, PDF export)
│   → ECharts headless OR visx → SVG → next/og ImageResponse
│
├── Real-time streaming (60Hz updates)
│   → ECharts with notMerge + lazyUpdate, or visx with manual requestAnimationFrame
│
└── One sparkline next to a number
   → react-sparklines@1, or Recharts' tiny `<LineChart>` with axes hidden
```

---

## 15. What This Topic Connects To

- **`08.ecosystem/02-ui-libraries.md`** — shadcn/ui Charts is part of the shadcn ecosystem.
- **`08.ecosystem/01-animation.md`** — visx pairs with framer-motion for custom animated charts.
- **`07.nextjs/07-seo-metadata.md`** — server-rendered charts for OG images via `ImageResponse`.
- **`06.testing-perf/02-performance.md`** — bundle analysis catches the "ECharts is 200 KB" problem.
- **`08.ecosystem/04-real-project.md`** — adding a "tasks completed per week" chart to the Task Manager is a textbook shadcn/ui Charts case.

---

## Summary

| Library | Pick when |
|---------|-----------|
| **shadcn/ui Charts** | Default for new dashboards in 2026 |
| **Tremor 3** | Want a complete dashboard kit with cards + lists + charts |
| **Recharts 2.12 (raw)** | Pre-shadcn project; team already comfortable |
| **ECharts 5 + echarts-for-react 3** | 5K+ points, candlesticks, complex types, server-rendering |
| **visx 3** | Custom visualizations; D3 power with React rendering |
| **Nivo 0.87** | Beautiful animated defaults out of the box |
| **react-sparklines 1** | Tiny inline sparklines |
| **Plotly** | Don't pick for new code (3 MB bundle) |
| **Chart.js** | Maintenance only; don't pick for new code |

| Rule | Why |
|------|-----|
| Default to shadcn/ui Charts for dashboards | Themed, accessible, AI-friendly out of the box |
| Tree-shake ECharts via `echarts-for-react/lib/core` | Default import is ~200 KB; tree-shaken is ~80 KB |
| SVG works to ~5K points; Canvas above that | Recharts/Nivo/visx struggle past 5K |
| Always handle the empty-data state | Empty charts confuse users |
| Server-render charts via ECharts headless or visx → SVG → next/og | Charts are too valuable to leave client-only for OG images |

---

## Further reading

- [shadcn/ui Charts](https://ui.shadcn.com/charts)
- [Recharts docs](https://recharts.org/en-US/)
- [Tremor docs](https://tremor.so/docs/getting-started/installation)
- [ECharts docs](https://echarts.apache.org/en/option.html)
- [echarts-for-react GitHub](https://github.com/hustcc/echarts-for-react)
- [visx docs](https://airbnb.io/visx/)
- [Nivo docs](https://nivo.rocks/)
- [`largestTriangleThreeBuckets` (LTTB) algorithm](https://skemman.is/handle/1946/15343)
- [Observable Plot](https://observablehq.com/plot/) — alternative grammar-of-graphics with React adapters appearing
