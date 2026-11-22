<script lang="ts">
  // The timeline, Creation to the early church, full screen and running top to
  // bottom. Era names sit large down the left edge, then the gold time axis
  // with its years; events branch off to its right, then the kingdoms track
  // shows who sat on each throne (the united monarchy forking into Israel and
  // Judah, each king coloured by his ruling house), then people's lives run
  // down beside it. Click anyone to light up their connections. The era bar
  // along the top shows the whole story and where you are in it: click an era
  // to see all of it, or drag along the bar to scroll.
  //
  // It is the horizontal timeline turned a quarter turn clockwise, with every
  // label kept upright. Labels now run across the time axis instead of along
  // it, so lanes are narrow columns and a label takes as many of them as it is
  // wide (packLanes' lanesFor). Lanes run left to right and fill in order of
  // time, so the story reads across as well as down: Adam, then Eve to his
  // right, then Cain and Abel. Every label sits to the right of its mark.
  //
  // d3-scale does the math, d3-zoom handles drag/pinch (d3-transition animates
  // jumps), the wheel scrolls, Svelte renders the SVG. Lanes are re-packed on
  // every zoom (not on pan), heaviest first, so zooming in reveals more.
  import { onMount } from "svelte"
  import { fade, scale } from "svelte/transition"
  import { cubicOut } from "svelte/easing"
  import { scaleLinear } from "d3-scale"
  import { zoom, zoomIdentity, type ZoomBehavior, type ZoomTransform } from "d3-zoom"
  import { select } from "d3-selection"
  import "d3-transition"
  import { packInTimeOrder, staggerLabels, type TimelineItem, type TimelinePeriod } from "../lib/timeline.ts"
  import { dynasties, placeLabels, reignNote, type KingdomRow, type KingdomTrack, type Reign } from "../lib/kingdoms.ts"
  import { formatSpan, formatYear, labelWidth } from "../lib/format.ts"
  import { entityUrl, setHash } from "../lib/url.ts"
  import { addInertia } from "../lib/inertia.ts"
  import { remember } from "../lib/remember.ts"
  import { ms } from "../lib/motion.ts"
  // A narrow face for the era bar, so its names can run larger. Bundled, not fetched
  // from a font service, so the installed app keeps it offline.
  import "@fontsource/pt-sans-narrow/latin-400.css"
  import "@fontsource/pt-sans-narrow/latin-700.css"

  let { items, periods, kingdoms }: { items: TimelineItem[]; periods: TimelinePeriod[]; kingdoms: KingdomTrack } =
    $props()

  // Layout, in px. Time runs down from TOP (under the site bar and the era
  // bar) to BOTTOM; horizontal positions follow the window width.
  const BOTTOM = 28
  const COL = 8 // one lane; a label takes as many as it is wide
  const LABEL_H = 16 // a label's height, along the time axis
  const CARD_W = 340

  const domain = $derived([periods[0].start, periods[periods.length - 1].end])
  // A first visit opens on Abraham onward, where most of the story is; the
  // eras before him are a scroll (or a click on the overview bar) away.
  const opening = $derived(periods.find((p) => p.id === "patriarchs")?.start ?? periods[0].start)
  const byId = $derived(new Map(items.map((i) => [i.id, i])))
  const maxWeight = $derived(Math.max(...items.map((i) => i.weight)))

  let width = $state(1200)
  let height = $state(800)
  let transform = $state<ZoomTransform>(zoomIdentity)
  // Raw, not deeply reactive: these hold items from props, compared by identity.
  let selected = $state.raw<TimelineItem | TimelinePeriod>()
  let hovered = $state.raw<{ item: TimelineItem; x: number; y: number; note?: string }>()
  // The server draws the timeline at a guessed size with nothing restored;
  // it stays hidden until the first real view (this window's size, where you
  // left off) is in place, then fades in, so launching doesn't flash (see
  // .reveal in global.css).
  let ready = $state(false)
  let root: HTMLDivElement
  let svg: SVGSVGElement
  let behavior: ZoomBehavior<SVGSVGElement, unknown>

  // Columns, left to right: the era names, tick labels, the axis, event lanes,
  // the kingdoms track, and people lanes.
  // On a phone every pixel across counts: the kingdoms track slims to bare
  // blocks (tap one for its king) and events get the larger share.
  const narrow = $derived(width < 700)
  // Breathing room before Creation and after the early church: fully zoomed
  // out, or scrolled to either end, time stops this far short of the edge. A
  // fixed distance on screen rather than extra years, so zoomed in you don't
  // scroll through centuries of nothing to reach the start.
  const PAD = $derived(narrow ? 32 : 64)

  // Each era has its own hue, worn by its segment on the era bar, its band
  // and big name in the timeline, and its stretch of the overview bar. Picked
  // so neighbours differ; more eras than hues just cycle round.
  const ERA_HUES = [32, 205, 48, 14, 38, 95, 268, 338, 218, 172, 58, 2, 142]
  const hue = (p: TimelinePeriod) => ERA_HUES[periods.indexOf(p) % ERA_HUES.length]

  // The era bar, along the top under the site bar: every era as long as it
  // lasted, named in full, the names alternating above and below the bar. A
  // name sits by the middle of its era if it can, slides along a little if
  // that clears its neighbour, and otherwise steps a row further out; a
  // hairline ties any name that moved to its era. Time starts below the bar.
  const ERA_TOP = 62 // under the site bar
  const ERA_SEG_H = 20 // tall enough to hold its years
  const ERA_LINE = 17 // a line of an era name
  const ERA_GAP = 3 // bar to the nearest names, and between rows of names
  const eraCharW = 5.9 // width of a letter in its names (16px PT Sans Narrow, measured)
  // On a phone the bar is colours alone (tap one for its era): its names would
  // stack too deep to leave room for the timeline.
  const eraNames = $derived(!narrow)
  const eraX = $derived(scaleLinear().domain(domain).range([16, width - 16]))
  // An era's name as one line, or two if one would overrun its segment and
  // two (split where the longer line is shortest) fit it better.
  function eraLines(name: string, room: number): string[] {
    const words = name.split(" ")
    if (name.length * eraCharW <= room || words.length < 2) return [name]
    let best = [name]
    let longest = name.length
    for (let i = 1; i < words.length; i++) {
      const lines = [words.slice(0, i).join(" "), words.slice(i).join(" ")]
      const l = Math.max(...lines.map((line) => line.length))
      if (l < longest) [best, longest] = [lines, l]
    }
    return best
  }
  const eraPlaced = $derived.by(() => {
    const spans = periods.map((p) => {
      const a = eraX(p.start)
      const b = eraX(p.end)
      const lines = eraLines(p.name, b - a - 4)
      return { p, a, b, lines, w: Math.max(...lines.map((l) => l.length)) * eraCharW }
    })
    const side = (n: number): "above" | "below" => (n % 2 === 0 ? "above" : "below")
    const place = (s: "above" | "below") => {
      const mine = spans.filter((_, n) => side(n) === s)
      const spots = staggerLabels(
        mine.map(({ a, b, w }) => ({ at: (a + b) / 2, width: w })),
        16,
        width - 16,
      )
      return new Map(mine.map((m, n) => [m.p, spots[n]!]))
    }
    const spots = { above: place("above"), below: place("below") }
    return spans.map(({ p, a, b, lines, w }, n) => {
      const { x: cx, row } = spots[side(n)].get(p)!
      const mid = (a + b) / 2
      // Where its hairline meets it: the point of the name nearest its era.
      const lx = Math.max(cx - w / 2 + 4, Math.min(cx + w / 2 - 4, mid))
      const moved = row > 0 || cx - w / 2 > b || cx + w / 2 < a
      return { p, a, b, mid, cx, w, lines, lx, moved, side: side(n), row }
    })
  })
  // How far each row of names on a side sits from the bar: rows stack outward,
  // each as tall as its tallest name.
  const rowOffsets = (side: "above" | "below") => {
    const heights: number[] = []
    for (const t of eraNames ? eraPlaced.filter((t) => t.side === side) : [])
      heights[t.row] = Math.max(heights[t.row] ?? 0, t.lines.length * ERA_LINE)
    const offsets = [ERA_GAP]
    for (const h of heights) offsets.push(offsets.at(-1)! + (h ?? 0) + ERA_GAP)
    return offsets // offsets[r] is where row r starts; the last is the side's depth
  }
  const above = $derived(rowOffsets("above"))
  const below = $derived(rowOffsets("below"))
  const ERA_BAR_Y = $derived(ERA_TOP + above.at(-1)!)

  // Years where eras meet, inside the bar: each era's first year at the left
  // end of its segment, and the last era's end at its right, wherever they
  // fit. A segment too short for its year goes without (its era's dates are on
  // its big name). BC is said once at the start and on the last year before AD.
  const YEAR_CHAR = 5.1 // width of a character in the year labels (11px PT Sans Narrow, measured)
  const yearLabel = (year: number, n: number, all: number[]) =>
    year > 0 ? `AD ${year}` : n === 0 || (all[n + 1] ?? 1) > 0 ? formatYear(year) : `${1 - year}`
  const eraYears = $derived.by(() => {
    const years = [periods[0].start, ...periods.map((p) => p.end)]
    const last = periods.length - 1
    const labels = years.map((year, n) => {
      const text = yearLabel(year, n, years)
      return { year, text, x: eraX(year), w: text.length * YEAR_CHAR, era: periods[Math.min(n, last)] }
    })
    const room = (p: TimelinePeriod) => eraX(p.end) - eraX(p.start)
    return labels.flatMap((l, n) => {
      const end = n === periods.length
      // The end of the last era shares its segment with that era's start.
      const taken = end && labels[last].w + 8 <= room(l.era) ? labels[last].w + 6 : 0
      if (l.w + 8 + taken > room(l.era)) return []
      return [{ ...l, cx: end ? l.x - 4 : l.x + 4, anchor: end ? "end" : "start" }]
    })
  })

  // Each name's first baseline and the block it fills, and where its hairline
  // leaves the bar and meets it. Names above grow upward from the bar, names
  // below downward.
  const eraTabs = $derived(
    eraPlaced.map((t) => {
      const h = t.lines.length * ERA_LINE
      if (t.side === "above") {
        const bottom = ERA_BAR_Y - above[t.row]
        return { ...t, top: bottom - h, h, y: bottom - h + 13.5, from: ERA_BAR_Y, to: bottom }
      }
      const top = ERA_BAR_Y + ERA_SEG_H + below[t.row]
      return { ...t, top, h, y: top + 13.5, from: ERA_BAR_Y + ERA_SEG_H, to: top }
    }),
  )
  const TOP = $derived(ERA_BAR_Y + ERA_SEG_H + below.at(-1)! + 10)
  const CARD_TOP = $derived(TOP + 8)
  // What's in view, marked on the era bar.
  const eraWindow = $derived.by(() => {
    const [v0, v1] = y.domain()
    const x = Math.max(16, eraX(v0))
    return { x, w: Math.max(3, Math.min(width - 16, eraX(v1)) - x) }
  })
  const AXIS_GAP = $derived(narrow ? 18 : 26) // axis to the first event lane
  const TRACK_GAP = $derived(narrow ? 8 : 14) // either side of the kingdoms track
  const eraW = $derived(narrow ? 0 : Math.round(Math.min(240, Math.max(140, width * 0.15))))
  const contentL = $derived(eraW + 8)
  const contentR = $derived(width - (narrow ? 10 : 22))
  const tickW = $derived(narrow ? 48 : 62)
  const trackCol = $derived(!kingdoms.kingdoms.length ? 0 : narrow ? 18 : width < 1100 ? 66 : 86)
  const colGap = $derived(narrow ? 5 : 12) // between track columns
  const trackW = $derived(kingdoms.rows * trackCol)
  const free = $derived(Math.max(120, contentR - contentL - tickW - AXIS_GAP - trackW - 2 * TRACK_GAP))
  const axisX = $derived(contentL + tickW)
  const eventsL = $derived(axisX + AXIS_GAP)
  const eventsR = $derived(eventsL + Math.round(free * (narrow ? 0.6 : 0.45))) // right edge of the event lanes
  const trackL = $derived(eventsR + TRACK_GAP)
  const peopleL = $derived(trackL + trackW + TRACK_GAP)
  const eventLanes = $derived(Math.max(1, Math.floor((eventsR - eventsL) / COL)))
  const peopleLanes = $derived(Math.max(1, Math.floor((contentR - peopleL) / COL)))
  // Where a lane's mark sits: the item's first column, its label to the right.
  const eventX = (lane: number) => eventsL + lane * COL + COL / 2
  const personX = (lane: number) => peopleL + lane * COL + COL / 2

  const base = $derived(scaleLinear().domain(domain).range([TOP + PAD, height - BOTTOM - PAD]))
  const y = $derived(transform.rescaleY(base))
  const ticks = $derived(y.ticks(Math.max(2, Math.floor((height - TOP) / 80))).filter((t) => Number.isInteger(t) && t >= domain[0] && t <= domain[1]))

  const isItem = (s: TimelineItem | TimelinePeriod | undefined): s is TimelineItem => !!s && "kind" in s
  const focus = $derived(isItem(selected) ? selected : undefined)
  // A selected kingdom also lights up its own line of kings.
  const related = $derived(new Set([...(focus?.related ?? []), ...(focus ? ownReigns(focus.id) : [])]))
  // Anything on the timeline (an item, a reign, a kingdom) by its id.
  const dimmed = (id: string) => !!focus && id !== focus.id && !related.has(id)

  const dotRadius = (i: TimelineItem) => 3 + 2.5 * Math.sqrt(i.weight / maxWeight)
  const ribbon = (i: TimelineItem) => 4 + 4 * Math.sqrt(i.weight / maxWeight)
  const strong = (i: TimelineItem) => i.weight > maxWeight * 0.12

  // Labels, written across the time axis. A label never runs wider than its
  // side of the axis: on a phone a long name is cut short rather than lost.
  const fontScale = (i: TimelineItem) => (i === focus ? 1.3 : strong(i) ? 1.12 : 1)
  function nameOf(i: TimelineItem) {
    const room = (i.kind === "event" ? eventLanes : peopleLanes) * COL - 24
    if (labelWidth(i.name) * fontScale(i) <= room) return i.name
    const chars = Math.max(1, Math.floor(room / (6.6 * fontScale(i))) - 1)
    return i.name.slice(0, chars).trimEnd() + "…"
  }
  const nameWidth = (i: TimelineItem) => labelWidth(nameOf(i)) * fontScale(i)
  // How many lanes an item takes across the axis: its mark, then its label.
  const lanesFor = (i: TimelineItem) => Math.ceil((COL / 2 + 12 + nameWidth(i)) / COL)

  // Lanes are packed in "world" pixels: the whole timeline at the current zoom
  // with no pan applied, not just what's in view. Dragging only pans, so
  // nothing changes lane mid-drag; lanes change only when the zoom does.
  const k = $derived(transform.k)
  const world = $derived(scaleLinear().domain(domain).range([(TOP + PAD) * k, (height - BOTTOM - PAD) * k]))
  // An item's footprint along time: its mark, padded, with room at its start
  // for the label. Attested people's traces fade out past their first and
  // last sighting.
  const FADE = 16
  const pad = (i: TimelineItem) => (i.kind === "person" && !i.lifetime ? FADE : 6)
  function extent(i: TimelineItem): [number, number] {
    const top = world(i.start) - Math.max(pad(i), LABEL_H / 2)
    return [top, Math.max(world(i.end) + pad(i), top + LABEL_H)]
  }
  // The selection and its connections jump the queue, so they're always shown.
  const priority = (i: TimelineItem) => (i === focus ? 2e9 : related.has(i.id) ? 1e9 + i.weight : i.weight)

  // Reigns and kingdoms are drawn in their own track, not in the event lanes.
  const onTrack = $derived(
    new Map(
      kingdoms.kingdoms.flatMap((k) => [
        [k.event, { kingdom: k }] as const,
        ...k.reigns.map((r) => [r.id, { kingdom: k, reign: r }] as const),
      ]),
    ),
  )
  const lanes = (kind: TimelineItem["kind"], maxLanes: number) =>
    packInTimeOrder(items.filter((i) => i.kind === kind && !onTrack.has(i.id)), priority, (i) => i.start, extent, maxLanes, 4, lanesFor)
  const eventLayout = $derived(lanes("event", eventLanes))
  const peopleLayout = $derived(lanes("person", peopleLanes))
  // World to screen is a shift by the pan, so culling is cheap.
  const inView = (placed: typeof eventLayout) =>
    placed.filter(({ x0, x1 }) => x1 + transform.y > 0 && x0 + transform.y < height)
  const visible = $derived([inView(eventLayout), inView(peopleLayout)])

  // --- Kingdoms track ----------------------------------------------------------

  // A kingdom's column. Rows count outward from the axis, so row 0 is nearest
  // it; the united monarchy spans every row, so the track forks where the
  // kingdom split. The gap on the far side holds co-regents' stripes.
  function rowBox(k: KingdomRow) {
    const [first, last] = k.row === "all" ? [0, kingdoms.rows - 1] : [k.row, k.row]
    return { x0: trackL + first * trackCol, x1: trackL + (last + 1) * trackCol - colGap }
  }
  // Labels sit at the top of what they name, two lines each: a kingdom and its
  // region, or a house ("House of" / "Omri"). They stick to the top of the view
  // while what they name runs off it, and never overlap each other.
  const LABEL_LINES = 24
  const houseName = (name: string) => name.replace(/^House of /, "")

  const trackView = $derived(
    kingdoms.kingdoms
      .filter((k) => y(k.end) > 0 && y(k.start) < height)
      .map((k) => {
        const box = rowBox(k)
        const reigns = k.reigns.filter((r) => y(r.end) > -4 && y(r.start) < height + 4)
        const labels = narrow ? [] : placeLabels(
          [
            { text: k.name.toUpperCase(), sub: k.region?.toUpperCase(), start: y(k.start), end: y(k.end), width: LABEL_LINES, tone: -1 },
            ...k.houses
              .filter((h) => h.named)
              .map((h) => ({ text: houseName(h.name), sub: undefined, start: y(h.start), end: y(h.end), width: LABEL_LINES, tone: h.tone })),
          ],
          TOP + 2,
          6,
        )
        return { k, box, reigns, labels }
      }),
  )
  // Where a reign's block is drawn. The main line is a segmented bar: a hairline
  // between kings, a wider gap where the house changes. A co-regent or rival
  // is a thin stripe just outside the column.
  function reignBox(r: Reign, box: ReturnType<typeof rowBox>) {
    if (r.sub) {
      const y0 = y(r.start)
      return { x: box.x1 + Math.min(5, colGap - 1) - 3 + (r.sub - 1) * 4, w: 3, y: y0, h: Math.max(3, y(r.end) - y0) }
    }
    const y0 = y(r.start) + (r.opens ? 1.5 : 0.5)
    const y1 = y(r.end) - (r.closes ? 1.5 : 0.5)
    return { x: box.x0, w: box.x1 - box.x0, y: y0, h: Math.max(3, y1 - y0) }
  }
  const tall = (k: KingdomRow) => k.row === "all" && kingdoms.rows > 1
  const kingWidth = (r: Reign, k: KingdomRow) => r.king.length * (tall(k) ? 7.6 : 6.4)
  // A king's name, centred in what's visible of his block, if it fits there
  // and stays clear of the track's labels.
  function kingY(b: ReturnType<typeof reignBox>, labels: { x: number }[]) {
    const top = Math.max(b.y, TOP)
    const bottom = Math.min(b.y + b.h, height)
    if (bottom - top < 15) return undefined
    const at = (top + bottom) / 2
    if (labels.some((l) => at > l.x - 8 && at < l.x + LABEL_LINES + 8)) {
      const below = Math.max(...labels.filter((l) => l.x < bottom && l.x + LABEL_LINES > top).map((l) => l.x + LABEL_LINES + 12))
      return below + 4 < bottom ? below : undefined
    }
    return at + 4
  }
  // Words wrapped to lines no wider than `room` px, at `perChar` px a letter.
  function wrap(text: string, room: number, perChar: number) {
    const lines: string[] = []
    for (const word of text.split(" ")) {
      const last = lines.at(-1)
      if (last && (last.length + 1 + word.length) * perChar <= room) lines[lines.length - 1] = `${last} ${word}`
      else lines.push(word)
    }
    return lines
  }

  function pickTrack(id: string) {
    const item = byId.get(id)
    if (item) selected = item
  }
  function ownReigns(id: string) {
    const at = onTrack.get(id)
    return at && !at.reign ? at.kingdom.reigns.map((r) => r.id) : []
  }
  const trackKind = (id: string) => {
    const at = onTrack.get(id)
    return at ? (at.reign ? "Reign" : "Kingdom") : "Event"
  }
  // What the card and tooltip say about a reign or kingdom.
  function trackNote(id: string) {
    const at = onTrack.get(id)
    if (!at) return undefined
    const { kingdom: k, reign } = at
    const name = k.region ? `${k.name} (${k.region.toLowerCase()})` : k.name
    const note = reign && reignNote(reign)
    if (reign) return `${name} · ${reign.house}${note ? ` · ${note}` : ""}`
    const houses = dynasties(k).length
    return `${k.reigns.length} kings · ${houses} royal ${houses === 1 ? "house" : "houses"}${k.fate ? ` · ${k.fate}` : ""}`
  }

  // Where each placed item is drawn: [top, bottom, mark x, left, right], for
  // links and the card.
  type Anchor = readonly [number, number, number, number, number]
  const anchors = $derived(
    new Map<string, Anchor>([
      ...visible.flatMap((placed) =>
        placed.map(({ item, lane }) => {
          const event = item.kind === "event"
          const mx = event ? eventX(lane) : personX(lane)
          const l = mx - COL / 2
          return [item.id, [y(item.start), y(item.end), mx, l, l + lanesFor(item) * COL] as const] as const
        }),
      ),
      ...trackView.flatMap(({ box, reigns }) =>
        reigns.map((r) => {
          const b = reignBox(r, box)
          return [r.id, [b.y, b.y + b.h, b.x + b.w / 2, b.x, b.x + b.w] as const] as const
        }),
      ),
    ]),
  )
  // Lineage: where a person's parent is on screen too, a hairline at the
  // child's first year runs from the parent's life to the child, so Adam's
  // line at 3874 BC leads to the start of Seth's. Only when the parent's span
  // covers that year: a line never says a parent was alive when the dates
  // don't.
  const lineage = $derived(
    visible[1].flatMap(({ item }) =>
      (item.parents ?? []).flatMap((id) => {
        const parent = byId.get(id)
        const from = anchors.get(id)
        const to = anchors.get(item.id)
        if (!parent || !from || !to || item.start < parent.start || item.start > parent.end + 1) return []
        return [{ key: `${id}>${item.id}`, parent: id, child: item.id, y: y(item.start), x0: from[2], x1: to[2] }]
      }),
    ),
  )

  // Across at the birth year, then a rounded turn down into the child's line.
  function elbow({ x0, x1, y }: { x0: number; x1: number; y: number }) {
    const r = Math.min(8, Math.abs(x1 - x0))
    const turn = x1 - Math.sign(x1 - x0) * r
    return `M${x0},${y} H${turn} Q${x1},${y} ${x1},${y + r} V${y + r + 6}`
  }

  // A label's centre line sticks to the top of the view while its item runs
  // off it, but never slides past the item's end.
  const labelY = (ys: number, ye: number) => Math.max(ys, Math.min(TOP + 10, ye))

  // When an item is: a person we only see at a few dated moments says so, so
  // the span doesn't read as a lifetime.
  function when(i: TimelineItem) {
    const span = formatSpan(i.start, i.end)
    if (i.kind !== "person" || i.lifetime) return span
    const n = (i.marks ?? [i.start]).length
    return `Mentioned ${span} (${n} dated ${n === 1 ? "event" : "events"})`
  }

  // Large era names down the left edge, upright, wrapped to the column and
  // sized to fit it and their band; each sticks to the top of its band and is
  // clipped to it.
  // The serif capitals, letter-spaced, run about 0.82 em a letter.
  const TITLE_EM = 0.82
  const TITLE_LINE = 1.05 // line height, in em
  function fitTitle(text: string, room: number, tall: number) {
    const words = text.split(" ")
    let best = { lines: [text], size: 0 }
    for (let n = 1; n <= words.length; n++) {
      const lines = wrap(text, Math.max(...words.map((w) => w.length), Math.ceil(text.length / n)), 1)
      if (lines.length > n) continue
      const longest = Math.max(...lines.map((l) => l.length))
      const size = Math.min(56, room / (longest * TITLE_EM), tall / (lines.length * TITLE_LINE))
      if (size > best.size) best = { lines, size }
    }
    return best
  }
  const eraTitles = $derived(
    periods.flatMap((p) => {
      const a = Math.max(0, y(p.start))
      const b = Math.min(height, y(p.end))
      if (b <= a) return []
      const top = Math.max(y(p.start), TOP) + 18
      const room = (narrow ? Math.min(150, width * 0.4) : eraW) - 24
      return [{ p, a, b, top, ...fitTitle(p.name.toUpperCase(), room, b - top - 36) }]
    }),
  )
  const eraLeft = $derived(narrow ? 12 : 22)
  const tickLabel = (year: number) => (year === 0 ? "BC | AD" : formatYear(year))

  // --- Dragging along the era bar ------------------------------------------------

  // A press on the era bar that moves scrolls the timeline to where the
  // pointer is, like a scrollbar; one that doesn't is a click on an era.
  let scrub: { x: number; moved: boolean } | undefined
  const barPoint = (e: PointerEvent) => e.clientX - svg.getBoundingClientRect().left
  function onBarDown(e: PointerEvent) {
    if (!e.button) scrub = { x: barPoint(e), moved: false }
  }
  function onBarMove(e: PointerEvent) {
    if (!scrub || !(e.buttons & 1)) return
    if (!scrub.moved) {
      if (Math.abs(barPoint(e) - scrub.x) < 4) return
      scrub.moved = true
      ;(e.currentTarget as Element).setPointerCapture(e.pointerId)
    }
    behavior.translateTo(select(svg), width / 2, base(eraX.invert(barPoint(e))))
  }
  // Cleared after the click that ends a drag, so the drag doesn't pick an era.
  const onBarUp = () => setTimeout(() => (scrub = undefined))

  // --- Zoom and navigation ----------------------------------------------------

  // The zoom works on the stretch of the window time runs through (for where
  // buttons and keys zoom around).
  function bound() {
    behavior?.extent([
      [0, TOP],
      [width, height - BOTTOM],
    ])
  }
  // Time never scrolls further than PAD past either end, and never sideways.
  function constrain(t: ZoomTransform) {
    const [first, last] = [t.applyY(base(domain[0])), t.applyY(base(domain[1]))]
    const shift = first > TOP + PAD ? TOP + PAD - first : last < height - BOTTOM - PAD ? height - BOTTOM - PAD - last : 0
    return zoomIdentity.translate(0, t.y + shift).scale(t.k)
  }

  onMount(() => {
    // Measure now rather than waiting for the resize observer behind
    // bind:clientWidth, so the first view is laid out at the real size.
    width = root.clientWidth
    height = root.clientHeight

    behavior = zoom<SVGSVGElement, unknown>()
      .scaleExtent([1, 800])
      .constrain(constrain)
      // The wheel scrolls (below); a pinch, or Ctrl/⌘ and the wheel, zooms. The
      // overview bar handles its own drags.
      .filter(
        (e) =>
          !e.button &&
          !(e.target as Element).closest?.(".eras") &&
          (e.type !== "wheel" || e.ctrlKey || e.metaKey),
      )
      .on("zoom", (e) => {
        transform = e.transform
        // What was under the pointer has moved on; its tooltip goes with it.
        if (e.sourceEvent) hovered = undefined
      })
    bound()
    select(svg).call(behavior)
    addInertia(behavior, svg, "y")

    const onWheel = (e: WheelEvent) => {
      if (e.ctrlKey || e.metaKey) return // a pinch: d3-zoom's
      e.preventDefault()
      hovered = undefined
      const dy = e.deltaY * (e.deltaMode === 1 ? 16 : e.deltaMode === 2 ? height : 1)
      behavior.translateBy(select(svg), 0, -dy / transform.k)
    }
    svg.addEventListener("wheel", onWheel, { passive: false })

    const lookup = (id: string) => byId.get(id) ?? periods.find((p) => p.id === id)
    // A link to something on the timeline shows it; otherwise come back to
    // where you were.
    const view = remember<{ span: number[]; selected?: string }>("timeline")
    const fromHash = lookup(decodeURIComponent(location.hash.slice(1)))
    const saved = view.load()
    if (fromHash) choose(fromHash, false)
    else if (saved) {
      behavior.transform(select(svg), span(saved.span[0], saved.span[1]))
      selected = saved.selected ? lookup(saved.selected) : undefined
    } else behavior.transform(select(svg), span(opening, domain[1]))
    const forget = view.onLeave(() => ({ span: shown(), selected: selected?.id }))
    // Show it once that view is drawn (two frames: past this update and its
    // layout), so nothing slides or fades into place on the way in.
    let frame = requestAnimationFrame(() => (frame = requestAnimationFrame(() => (ready = true))))

    // The site search shows what's on the timeline here, in place.
    const onPick = (e: Event) => {
      const target = lookup((e as CustomEvent<{ id: string }>).detail.id)
      if (!target) return
      e.preventDefault()
      choose(target)
    }
    addEventListener("search-pick", onPick)
    return () => {
      cancelAnimationFrame(frame)
      svg.removeEventListener("wheel", onWheel)
      removeEventListener("search-pick", onPick)
      forget()
    }
  })

  // Keep zoom bounds in step with the window.
  $effect(bound)

  // A selection is shareable: it lives in the URL hash.
  $effect(() => {
    const hash = focus ? `#${focus.id}` : ""
    if (location.hash !== hash) setHash(hash.slice(1))
  })

  // The view that shows exactly [start, end].
  // It fills the stretch between the PAD margins, the same breathing room the
  // ends of time get.
  function span(start: number, end: number) {
    const [top, bottom] = [TOP + PAD, height - BOTTOM - PAD]
    const k = (bottom - top) / (base(end) - base(start))
    return zoomIdentity.translate(0, top - k * base(start)).scale(k)
  }
  // The years span() was given for the current view, to save and restore it.
  const shown = () => [y.invert(TOP + PAD), y.invert(height - BOTTOM - PAD)]
  // Glide to show [start, end], with some room around it unless `room` is 0
  // (or jump there).
  function flyTo(start: number, end: number, glide = true, room = 0.08) {
    const pad = room ? Math.max(8, (end - start) * room) : 0
    const view = span(start - pad, end + pad)
    if (glide) select(svg).transition().duration(ms(750)).call(behavior.transform, view)
    else behavior.transform(select(svg), view)
  }
  const zoomBy = (factor: number) => select(svg).transition().duration(ms(250)).call(behavior.scaleBy, factor)

  function pickPeriod(p: TimelinePeriod, glide = true) {
    flyTo(p.start, p.end, glide, 0) // an era fills the view, start to end
    selected = p
  }

  // Select an item and bring it into view, with enough context around it.
  function choose(target: TimelineItem | TimelinePeriod, glide = true) {
    if (!isItem(target)) return pickPeriod(target, glide)
    selected = target
    const mid = (target.start + target.end) / 2
    const half = Math.max(45, (target.end - target.start) * 0.75)
    flyTo(mid - half, mid + half, glide)
  }

  function onKey(e: KeyboardEvent) {
    if (e.target instanceof HTMLInputElement) return
    const step = (height - TOP) / 8 / transform.k
    if (e.key === "Escape") selected = undefined
    else if (e.key === "ArrowUp") behavior.translateBy(select(svg), 0, step)
    else if (e.key === "ArrowDown") behavior.translateBy(select(svg), 0, -step)
    else if (e.key === "PageUp") behavior.translateBy(select(svg), 0, step * 6)
    else if (e.key === "PageDown") behavior.translateBy(select(svg), 0, -step * 6)
    else if (e.key === "+" || e.key === "=") zoomBy(1.6)
    else if (e.key === "-") zoomBy(1 / 1.6)
    else return
    e.preventDefault()
  }

  // --- Selection card -----------------------------------------------------------

  const card = $derived.by(() => {
    if (!focus) return undefined
    const a = anchors.get(focus.id)
    if (!a) return { left: contentR - CARD_W - 8, top: CARD_TOP }
    // Beside the item if there's room on either side; otherwise tucked into
    // the top right. Level with the item's start (or the top of the view).
    const [ys, , , x0, x1] = a
    const top = Math.max(CARD_TOP, Math.min(Math.max(ys, TOP) - 40, height - 330))
    if (x1 + 24 + CARD_W < width - 16) return { left: x1 + 24, top }
    if (x0 - 24 - CARD_W > contentL) return { left: x0 - 24 - CARD_W, top }
    return { left: contentR - CARD_W - 8, top: CARD_TOP }
  })
  const relatedPeople = $derived(
    (focus?.related ?? [])
      .map((id) => byId.get(id)!)
      .filter((i) => i.kind === "person")
      .sort((a, b) => b.weight - a.weight),
  )
  const relatedEvents = $derived((focus?.related ?? []).filter((id) => byId.get(id)?.kind === "event").length)
</script>

<svelte:window onkeydown={onKey} />

<div class="timeline reveal" class:ready bind:this={root} bind:clientWidth={width} bind:clientHeight={height}>
  <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_noninteractive_element_interactions -->
  <svg
    id="timeline-canvas"
    class="veiled"
    bind:this={svg}
    viewBox="0 0 {width} {height}"
    role="application"
    aria-label="Timeline from Creation to the early church, top to bottom. Up and down arrows scroll, plus and minus zoom, slash searches."
    onclick={(e) => {
      if (e.target === svg || (e.target as Element).classList.contains("band")) selected = undefined
    }}
  >
    <defs>
      <linearGradient id="tl-axis" gradientUnits="userSpaceOnUse" x1="0" x2="0" y1="0" y2={height}>
        <stop offset="0" style="stop-color: var(--gold); stop-opacity: 0" />
        <stop offset="0.12" style="stop-color: var(--gold)" />
        <stop offset="0.88" style="stop-color: var(--gold)" />
        <stop offset="1" style="stop-color: var(--gold); stop-opacity: 0" />
      </linearGradient>
      {#each [["tl-person", "--people"], ["tl-focus", "--accent"]] as [id, color] (id)}
        <linearGradient {id} x1="0" y1="0" x2="0" y2="1">
          <stop offset="0" style="stop-color: var({color}); stop-opacity: 0.35" />
          <stop offset="0.15" style="stop-color: var({color})" />
          <stop offset="0.85" style="stop-color: var({color})" />
          <stop offset="1" style="stop-color: var({color}); stop-opacity: 0.35" />
        </linearGradient>
      {/each}
      <!-- An attested person's trace: no hard ends, since we don't know when they lived -->
      {#each [["tl-trace", "--people"], ["tl-trace-focus", "--accent"]] as [id, color] (id)}
        <linearGradient {id} x1="0" y1="0" x2="0" y2="1">
          <stop offset="0" style="stop-color: var({color}); stop-opacity: 0" />
          <stop offset="0.3" style="stop-color: var({color}); stop-opacity: 0.7" />
          <stop offset="0.7" style="stop-color: var({color}); stop-opacity: 0.7" />
          <stop offset="1" style="stop-color: var({color}); stop-opacity: 0" />
        </linearGradient>
      {/each}
      <filter id="tl-glow" x="-300%" y="-20%" width="700%" height="140%">
        <feGaussianBlur stdDeviation="3" result="blur" />
        <feMerge><feMergeNode in="blur" /><feMergeNode in="SourceGraphic" /></feMerge>
      </filter>
      <radialGradient id="tl-vignette" cx="0.55" cy="0.5" r="0.8">
        <stop offset="0.55" style="stop-color: var(--edge); stop-opacity: 0" />
        <stop offset="1" style="stop-color: var(--edge)" />
      </radialGradient>
      {#each eraTitles as { p, a, b } (p.id)}
        <clipPath id="tl-clip-{p.id}"><rect x="0" y={a} {width} height={b - a} /></clipPath>
      {/each}
      <clipPath id="tl-right-of-axis"><rect x={axisX} y="0" width={Math.max(0, width - axisX)} {height} /></clipPath>
    </defs>

    <!-- Eras: alternating bands, a dotted edge, and the era's name writ large -->
    {#each eraTitles as { p, a, b, top, lines, size } (p.id)}
      <rect class="band" class:alt={periods.indexOf(p) % 2 === 1} style="--hue: {hue(p)}" x="0" y={a} {width} height={b - a} />
      <line class="era-edge" x1="10" x2={contentR + 6} y1={y(p.start)} y2={y(p.start)} />
      {#if size >= 12}
        <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_static_element_interactions -->
        <g class="era" style="--hue: {hue(p)}" onclick={() => pickPeriod(p)}>
          <text class="era-name" clip-path="url(#tl-clip-{p.id})" x={eraLeft} y={top + size * 0.8} style="font-size: {size}px"
            >{#each lines as line, n (n)}<tspan x={eraLeft} dy={n ? size * TITLE_LINE : 0}>{line}</tspan>{/each}</text
          >
          <text class="era-dates" clip-path="url(#tl-clip-{p.id})" x={eraLeft + 4} y={top + size * 0.8 + (lines.length - 1) * size * TITLE_LINE + 22}
            >{p.when}</text
          >
        </g>
      {/if}
    {/each}

    <!-- The time axis -->
    <line x1={axisX} x2={axisX} y1="0" y2={height} stroke="url(#tl-axis)" stroke-width="1.5" />
    {#each ticks as t (t)}
      <line class="tick-mark" x1={axisX - 5} x2={axisX + 5} y1={y(t)} y2={y(t)} />
      <text class="tick" class:small={narrow} x={axisX - 8} y={y(t) + 4} text-anchor="end">{tickLabel(t)}</text>
    {/each}

    <!-- Kingdoms: who sat on each throne, each king coloured by his house -->
    {#each trackView as { k, box, reigns, labels } (k.id)}
      {@const cx = (box.x0 + box.x1) / 2}
      <g class="kingdom">
        {#each reigns as r (r.id)}
          {@const b = reignBox(r, box)}
          {@const ky = !r.sub && b.w > kingWidth(r, k) + 6 ? kingY(b, labels) : undefined}
          <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_static_element_interactions -->
          <g
            class="reign tone-{r.tone}"
            class:sub={r.sub > 0}
            class:tall={tall(k)}
            class:dim={dimmed(r.id)}
            class:focus={focus?.id === r.id}
            onclick={(e) => {
              e.stopPropagation()
              pickTrack(r.id)
            }}
            onpointerenter={(e) => {
              if (e.pointerType !== "mouse") return
              const item = byId.get(r.id)
              if (item) hovered = { item, x: b.x + b.w, y: e.offsetY, note: `${r.king} · ${r.note ?? r.house}` }
            }}
            onpointerleave={() => (hovered = undefined)}
          >
            {#if r.sub}<rect class="hit" x={b.x - 3} y={b.y} width={b.w + 6} height={b.h} />{/if}
            <rect class="block" x={b.x} y={b.y} width={b.w} height={b.h} rx={r.sub ? 1.5 : 2.5} />
            {#if ky !== undefined}
              <text x={cx} y={ky} text-anchor="middle">{r.king}</text>
            {/if}
          </g>
        {/each}
        {#each labels as l (l.text + l.start)}
          {#if l.tone === -1}
            <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_static_element_interactions -->
            <g
              class="kingdom-name"
              class:dim={dimmed(k.event)}
              onclick={(e) => {
                e.stopPropagation()
                pickTrack(k.event)
              }}
            >
              <text x={cx} y={l.x + 10} text-anchor="middle">{l.text}</text>
              {#if l.sub}<text class="region" x={cx} y={l.x + 21} text-anchor="middle">{l.sub}</text>{/if}
            </g>
          {:else}
            <g class="house tone-{l.tone}" class:dim={!!focus}>
              <text class="house-of" x={cx} y={l.x + 9} text-anchor="middle">HOUSE OF</text>
              <text x={cx} y={l.x + 21} text-anchor="middle">{l.text}</text>
            </g>
          {/if}
        {/each}
        {#if k.fate && !narrow && y(k.end) + 8 < height}
          <text class="fate" x={cx} y={y(k.end) + 16} text-anchor="middle"
            >{#each wrap(k.fate, box.x1 - box.x0 + 8, 6) as line, n (n)}<tspan x={cx} dy={n ? 14 : 0}>{line}</tspan>{/each}</text
          >
        {/if}
      </g>
    {/each}

    <!-- Lineage: a parent's life to a child's first year -->
    {#each lineage as l (l.key)}
      <g
        class="lineage"
        class:dim={!!focus && focus.id !== l.parent && focus.id !== l.child}
        class:lit={focus?.id === l.parent || focus?.id === l.child}
      >
        <path d={elbow(l)} />
        <circle cx={l.x0} cy={l.y} r="2.5" />
      </g>
    {/each}

    <!-- Links from the selection to its connections -->
    {#if focus && anchors.has(focus.id)}
      {@const [fy0, fy1, fx] = anchors.get(focus.id)!}
      {@const fy = (Math.max(fy0, 0) + Math.min(fy1, height)) / 2}
      {#each focus.related as id (id)}
        {#if anchors.has(id)}
          {@const [ty0, ty1, tx] = anchors.get(id)!}
          {@const ty = (Math.max(ty0, 0) + Math.min(ty1, ty0 + 60, height)) / 2}
          {@const mx = (fx + tx) / 2}
          <path
            pathLength="1"
            class="link"
            class:to-event={byId.get(id)?.kind === "event"}
            d="M{fx},{fy} C{mx},{fy} {mx},{ty} {tx},{ty}"
          />
        {/if}
      {/each}
    {/if}

    <!-- Event stems, back to the axis: a layer of their own under every mark and
         label, so a stem passes behind the names nearer the axis rather than
         through them. Each runs long and is clipped at the axis, and moves with
         its event's lane. -->
    <g clip-path="url(#tl-right-of-axis)">
      {#each visible[0] as { item } (item.id)}
        {@const [ys, , mx] = anchors.get(item.id)!}
        <line
          class="stem"
          class:dim={dimmed(item.id)}
          class:lit={item === focus || hovered?.item === item}
          style="transform: translateX({mx}px)"
          x1="0"
          x2={-width}
          y1={ys}
          y2={ys}
        />
      {/each}
    </g>

    <!-- Events right of the axis, people right of the kingdoms track, labels to
         the right of their marks. Each item sits at its lane via a CSS transform,
         so a lane change on zoom glides rather than jumps. -->
    {#each visible as placed, section (section)}
      <g clip-path={section === 0 ? "url(#tl-right-of-axis)" : undefined}>
        {#each placed as { item } (item.id)}
          {@const [ys, ye, mx] = anchors.get(item.id)!}
          {@const span = item.end > item.start}
          {@const name = nameOf(item)}
          {@const w = nameWidth(item)}
          <!-- Pointer layer; every item also has its own page and index entry. -->
          <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_static_element_interactions -->
          <g
            class="item {item.kind}"
            class:dim={dimmed(item.id)}
            class:focus={item === focus}
            style="transform: translateX({mx}px)"
            onclick={(e) => {
              e.stopPropagation()
              selected = item
            }}
            onpointerenter={(e) => {
              if (e.pointerType === "mouse") hovered = { item, x: e.offsetX, y: e.offsetY }
            }}
            onpointerleave={() => (hovered = undefined)}
          >
            {#if item.kind === "event"}
              <rect class="hit" x="-8" y={ys - 11} width={w + 24} height={Math.max(0, ye - ys) + 22} />
              {#if span}
                <rect class="bar" x="-2.5" y={ys} width="5" height={ye - ys} rx="2.5" />
              {/if}
              <circle class="dot" cx="0" cy={ys} r={dotRadius(item)} />
              <text class:strong={strong(item)} x={span ? 9 : dotRadius(item) + 5} y={(span ? labelY(ys, ye) : ys) + 4}>{name}</text>
            {:else if !item.lifetime}
              <!-- Lifespan unknown: a dot for each dated sighting on a faded trace -->
              <rect class="hit" x="-8" y={ys - FADE} width={w + 24} height={ye - ys + 2 * FADE} />
              <rect
                class="trace"
                x="-1"
                y={ys - FADE}
                width="2"
                height={ye - ys + 2 * FADE}
                fill="url(#{item === focus ? 'tl-trace-focus' : 'tl-trace'})"
              />
              {#each item.marks ?? [item.start] as year (year)}
                <circle class="sighting" cx="0" cy={y(year)} r={dotRadius(item) - 0.5} />
              {/each}
              <text class:strong={strong(item)} x="9" y={labelY(ys, ye) + 4}>{name}</text>
            {:else}
              {@const h = ribbon(item)}
              <rect class="hit" x={-h / 2 - 6} y={ys - 8} width={w + h + 16} height={Math.max(0, ye - ys) + 16} />
              <rect
                class="ribbon"
                x={-h / 2}
                y={ys}
                width={h}
                height={Math.max(6, ye - ys)}
                rx={h / 2}
                style={ye - ys > 40 ? `fill: url(#${item === focus ? "tl-focus" : "tl-person"})` : ""}
                filter={item === focus ? "url(#tl-glow)" : undefined}
              />
              <text class:strong={strong(item)} x={h / 2 + 5} y={labelY(ys, ye) + 4}>{name}</text>
            {/if}
          </g>
        {/each}
      </g>
    {/each}

    <rect class="vignette" {width} {height} fill="url(#tl-vignette)" />

    <!-- Era bar: every era along the top, as long as it lasted; click one to
         see all of it. The window marks what's in view. -->
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <g class="eras" class:scrubbing={scrub?.moved} onpointerdown={onBarDown} onpointermove={onBarMove} onpointerup={onBarUp} onpointercancel={onBarUp}>
      <rect class="eras-back" x="0" y="0" {width} height={TOP - 6} />
      {#each eraTabs as { p, a, b, mid, cx, w, lines, lx, moved, top: ty, h, y: ly, from, to } (p.id)}
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <g
          class="era-tab"
          class:on={selected === p}
          style="--hue: {hue(p)}"
          role="button"
          tabindex="0"
          aria-label="Show {p.name}, {p.when}"
          onclick={() => {
            if (!scrub?.moved) pickPeriod(p)
          }}
          onkeydown={(e) => {
            if (e.key !== "Enter" && e.key !== " ") return
            e.preventDefault()
            pickPeriod(p)
          }}
        >
          <title>{p.name} · {p.when}</title>
          <rect class="era-seg" x={a} y={ERA_BAR_Y} width={Math.max(1, b - a - 1.5)} height={ERA_SEG_H} rx="2" />
          <rect class="hit" x={a} y={ERA_BAR_Y - 4} width={Math.max(4, b - a)} height={ERA_SEG_H + 8} />
          {#if eraNames}
            {#if moved}<line class="era-lead" x1={mid} x2={lx} y1={from} y2={to} />{/if}
            <rect class="hit" x={cx - w / 2 - 4} y={ty} width={w + 8} height={h} />
            <text x={cx} y={ly} text-anchor="middle"
              >{#each lines as line, n (n)}<tspan x={cx} dy={n ? ERA_LINE : 0}>{line}</tspan>{/each}</text
            >
          {/if}
        </g>
      {/each}
      {#each eraYears as l (l.year)}
        <text class="era-year inside" style="--hue: {hue(l.era)}" x={l.cx} y={ERA_BAR_Y + ERA_SEG_H / 2 + 4} text-anchor={l.anchor}>{l.text}</text>
      {/each}
      <rect class="eras-window" x={eraWindow.x} y={ERA_BAR_Y - 3} width={eraWindow.w} height={ERA_SEG_H + 6} rx="3" />
    </g>
  </svg>

  {#if hovered && hovered.item !== focus}
    <div class="tooltip" style="left: {hovered.x + 14}px; top: {hovered.y + 10}px" in:fade={{ duration: 120 }}>
      <strong>{when(hovered.item)}</strong>
      <span>{hovered.note ?? hovered.item.name}</span>
    </div>
  {/if}

  {#if selected && card}
    <div class="card" style="left: {card.left}px; top: {card.top}px" aria-live="polite" in:scale={{ start: 0.96, duration: 200, easing: cubicOut }} out:fade={{ duration: 120 }}>
      <button type="button" class="close" aria-label="Close" onclick={() => (selected = undefined)}>×</button>
      {#if isItem(selected)}
        <p class="kind">
          {selected.kind === "person" ? (selected.lifetime ? "Person" : "Person · lifespan unknown") : trackKind(selected.id)}
        </p>
        <h2>{selected.name}</h2>
        <p class="when">{when(selected)}{selected.eras ? ` · ${selected.eras}` : ""}</p>
        {#if trackNote(selected.id)}<p class="when throne">{trackNote(selected.id)}</p>{/if}
        {#if selected.summary}<p class="summary">{selected.summary}</p>{/if}
        {#if relatedPeople.length}
          <div class="chips">
            <span class="label">{selected.kind === "person" ? "Family" : onTrack.get(selected.id)?.reign ? "King" : "People"}</span>
            {#each relatedPeople.slice(0, 5) as p (p.id)}
              <button type="button" class="chip" onclick={() => choose(p)}>{p.name}</button>
            {/each}
            {#if relatedPeople.length > 5}<span class="more">+{relatedPeople.length - 5} more</span>{/if}
          </div>
        {/if}
        {#if relatedEvents}
          <p class="events">{relatedEvents} {relatedEvents === 1 ? "event" : "events"} on the timeline</p>
        {/if}
        <a href={entityUrl(selected)}>Open {selected.kind === "person" ? "person" : trackKind(selected.id).toLowerCase()} →</a>
      {:else}
        <p class="kind">Era</p>
        <h2>{selected.name}</h2>
        <p class="when">{selected.when}</p>
        {#if selected.summary}<p class="summary">{selected.summary}</p>{/if}
      {/if}
    </div>
  {/if}

  <div class="corner">
    <p>Scroll to move · pinch or Ctrl-scroll to zoom · click anyone</p>
  </div>
  <div class="zoom-buttons" role="group" aria-label="Zoom">
    <button type="button" class="icon-button" aria-label="Zoom out" onclick={() => zoomBy(1 / 1.6)}>−</button>
    <button type="button" class="icon-button" aria-label="Zoom in" onclick={() => zoomBy(1.6)}>+</button>
  </div>
</div>

<style>
  .timeline {
    position: relative;
    height: 100%;
    overflow: hidden;
    background: var(--paper);
  }
  svg {
    display: block;
    width: 100%;
    height: 100%;
    cursor: grab;
    user-select: none;
    touch-action: none; /* every pinch and drag is the timeline's, never the browser's */
    font-family: var(--sans);
  }
  svg:active {
    cursor: grabbing;
  }

  /* Eras wear their hue (--hue, set per era) faintly in the timeline and
     fully on the era bar. */
  .band {
    fill: hsl(var(--hue) 55% 50% / 0.05);
  }
  .band.alt {
    fill: hsl(var(--hue) 55% 50% / 0.08);
  }
  .era-edge {
    stroke: var(--gold);
    stroke-opacity: 0.25;
    stroke-dasharray: 2 6;
  }
  .era {
    cursor: pointer;
  }
  .era-name {
    font-family: var(--serif);
    font-weight: 600;
    letter-spacing: 0.1em;
    fill: hsl(var(--hue) 45% 38% / 0.2);
    transition: fill 0.2s;
  }
  .era:hover .era-name {
    fill: var(--gold);
    fill-opacity: 0.35;
  }
  .era-dates {
    font-size: 11px;
    letter-spacing: 0.18em;
    fill: var(--ink-3);
  }
  .tick-mark {
    stroke: var(--gold);
    stroke-opacity: 0.6;
  }
  .tick {
    font-size: 11px;
    fill: var(--ink-3);
    font-variant-numeric: tabular-nums;
  }
  .tick.small {
    font-size: 9.5px;
  }

  .item {
    cursor: pointer;
    transition:
      opacity 0.2s,
      transform 0.28s cubic-bezier(0.2, 0.7, 0.3, 1);
    /* Fade in when added: CSS, as a zoom can add dozens of items a frame */
    animation: appear 0.18s;
  }
  @keyframes appear {
    from {
      opacity: 0;
    }
  }
  .item.dim {
    opacity: 0.18;
  }
  .hit {
    fill: transparent;
  }
  .item text {
    font-size: 12px;
    fill: var(--ink-2);
  }
  /* Names keep a paper halo, so a stem or lineage line passing behind one
     doesn't strike it through. */
  .item text {
    paint-order: stroke;
    stroke: var(--paper);
    stroke-width: 3px;
    stroke-linejoin: round;
  }
  .item text.strong {
    fill: var(--ink);
    font-weight: 600;
  }
  .person text.strong {
    font: 600 14px var(--serif);
  }
  .item:hover text {
    fill: var(--ink);
  }
  .item.focus text {
    fill: var(--accent);
    font: 600 16px var(--serif);
  }
  .stem {
    stroke: var(--rule);
    transition:
      opacity 0.2s,
      transform 0.28s cubic-bezier(0.2, 0.7, 0.3, 1);
  }
  .stem.dim {
    opacity: 0.18;
  }
  .stem.lit {
    stroke: var(--gold);
  }
  .bar,
  .dot {
    fill: var(--events);
  }
  .dot {
    stroke: var(--paper);
    stroke-width: 2;
    paint-order: stroke;
  }
  .ribbon {
    fill: var(--people);
  }
  .sighting {
    fill: var(--people);
    stroke: var(--paper);
    stroke-width: 1.5;
    paint-order: stroke;
  }
  .item.focus .ribbon,
  .item.focus .sighting {
    fill: var(--accent);
  }
  .lineage {
    pointer-events: none;
    transition: opacity 0.2s;
  }
  .lineage path {
    fill: none;
    stroke: var(--people);
    stroke-opacity: 0.45;
  }
  .lineage circle {
    fill: var(--people);
    opacity: 0.7;
  }
  .lineage.dim {
    opacity: 0.2;
  }
  .lineage.lit path {
    stroke: var(--accent);
    stroke-opacity: 0.85;
    stroke-width: 1.3;
  }
  .lineage.lit circle {
    fill: var(--accent);
    opacity: 1;
  }
  .link {
    fill: none;
    stroke: var(--accent);
    stroke-width: 1.2;
    opacity: 0.6;
  }
  /* Links draw themselves out from the selection (pathLength="1"). */
  .link:not(.to-event) {
    stroke-dasharray: 1;
    stroke-dashoffset: 1;
    animation: draw 0.6s ease-out forwards;
  }
  @keyframes draw {
    to {
      stroke-dashoffset: 0;
    }
  }
  .link.to-event {
    animation: appear 0.6s ease-out;
  }
  @keyframes appear {
    from {
      opacity: 0;
    }
  }
  .link.to-event {
    stroke-dasharray: 3 3;
    opacity: 0.45;
  }

  /* Kingdoms track. Each house has a tone (.tone-N in global.css): David's
     line is gold, so it reads at a glance through Judah. */
  .reign {
    cursor: pointer;
    transition: opacity 0.2s;
  }
  .reign .block {
    fill: var(--tone);
    fill-opacity: 0.26;
    transition: fill-opacity 0.15s;
  }
  .reign.tone-0 .block {
    fill-opacity: 0.36;
  }
  .reign.sub .block {
    fill-opacity: 0.85;
  }
  .reign:hover .block {
    fill-opacity: 0.5;
  }
  .reign.focus .block {
    fill-opacity: 0.55;
    stroke: var(--accent);
    stroke-width: 1.5;
  }
  .reign text {
    font-size: 11.5px;
    fill: var(--ink-2);
    pointer-events: none;
  }
  .reign:hover text,
  .reign.focus text {
    fill: var(--ink);
  }
  .reign.tall text {
    font: 600 13px var(--serif);
    fill: var(--ink);
  }
  .reign.dim,
  .house.dim,
  .kingdom-name.dim {
    opacity: 0.3;
  }
  .kingdom-name {
    cursor: pointer;
  }
  .kingdom-name text {
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.16em;
    fill: var(--ink-3);
    paint-order: stroke;
    stroke: var(--paper);
    stroke-width: 3px;
    stroke-linejoin: round;
  }
  .kingdom-name .region {
    font-size: 8.5px;
    font-weight: 500;
  }
  .kingdom-name:hover text {
    fill: var(--accent);
  }
  .house {
    pointer-events: none;
  }
  .house text {
    font: italic 11.5px var(--serif);
    fill: color-mix(in srgb, var(--tone) 65%, var(--ink));
    paint-order: stroke;
    stroke: var(--paper);
    stroke-width: 3px;
    stroke-linejoin: round;
  }
  .house .house-of {
    font: 600 7.5px var(--sans);
    letter-spacing: 0.16em;
  }
  .fate {
    font: italic 11px var(--serif);
    fill: var(--ink-3);
    pointer-events: none;
  }
  .card .throne {
    color: var(--gold);
  }

  .vignette {
    pointer-events: none;
  }

  /* The era bar */
  .eras-back {
    fill: var(--paper);
    opacity: 0.94;
  }
  .era-tab {
    cursor: pointer;
    outline: none;
  }
  .era-seg {
    fill: hsl(var(--hue) 45% 66%);
    stroke: hsl(var(--hue) 35% 45% / 0.35);
    transition: fill 0.15s;
  }
  .era-lead {
    stroke: var(--rule);
  }
  .era-tab text {
    font-family: "PT Sans Narrow", var(--sans);
    font-size: 16px;
    fill: hsl(var(--hue) 40% 30%);
    paint-order: stroke;
    stroke: var(--paper);
    stroke-width: 3px;
    stroke-linejoin: round;
    transition: fill 0.15s;
  }
  .era-tab:hover .era-seg,
  .era-tab:focus-visible .era-seg,
  .era-tab.on .era-seg {
    fill: hsl(var(--hue) 55% 50%);
  }
  .era-tab:hover text,
  .era-tab:focus-visible text,
  .era-tab.on text {
    fill: var(--accent);
  }
  .era-tab:hover .era-lead,
  .era-tab.on .era-lead {
    stroke: var(--gold);
  }
  /* Dark theme: the same hues, deeper and softer */
  :global([data-theme="dark"]) .band {
    fill: hsl(var(--hue) 50% 65% / 0.04);
  }
  :global([data-theme="dark"]) .band.alt {
    fill: hsl(var(--hue) 50% 65% / 0.07);
  }
  :global([data-theme="dark"]) .era-name {
    fill: hsl(var(--hue) 40% 75% / 0.13);
  }
  :global([data-theme="dark"]) .era-seg {
    fill: hsl(var(--hue) 32% 40%);
    stroke: hsl(var(--hue) 30% 60% / 0.3);
  }
  :global([data-theme="dark"]) .era-tab text {
    fill: hsl(var(--hue) 40% 78%);
  }
  :global([data-theme="dark"]) .era-tab:hover text,
  :global([data-theme="dark"]) .era-tab.on text {
    fill: var(--accent);
  }
  :global([data-theme="dark"]) .era-tab:hover .era-seg,
  :global([data-theme="dark"]) .era-tab.on .era-seg {
    fill: hsl(var(--hue) 45% 52%);
  }
  .eras {
    cursor: pointer;
  }
  .eras.scrubbing {
    cursor: grabbing;
  }
  .era-year {
    font: 11px "PT Sans Narrow", var(--sans);
    fill: var(--ink-3);
    font-variant-numeric: tabular-nums;
    pointer-events: none;
  }
  .era-year.inside {
    font-weight: 700;
    fill: hsl(var(--hue) 45% 22%);
  }
  :global([data-theme="dark"]) .era-year.inside {
    fill: hsl(var(--hue) 40% 88%);
  }
  .eras-window {
    fill: none;
    stroke: var(--gold);
    stroke-width: 1.5;
    pointer-events: none;
  }

  .card {
    position: absolute;
    width: 340px;
    padding: 1rem 1.15rem 1.1rem;
    border-radius: 12px;
    background: var(--surface);
    border: 1px solid color-mix(in srgb, var(--gold) 45%, transparent);
    box-shadow: 0 14px 40px rgb(40 30 15 / 0.18);
    transition:
      left 0.3s,
      top 0.3s;
  }
  .card .kind {
    margin: 0;
    font: 600 0.65rem var(--sans);
    letter-spacing: 0.16em;
    text-transform: uppercase;
    color: var(--gold);
  }
  .card h2 {
    margin: 0.2rem 0 0.1rem;
    font-size: 1.5rem;
  }
  .card .when {
    margin: 0;
    font-size: 0.82rem;
    color: var(--ink-2);
  }
  .card .summary {
    margin: 0.6rem 0;
    font: 0.95rem/1.5 var(--serif);
    color: var(--ink-2);
  }
  .card .close {
    position: absolute;
    top: 0.5rem;
    right: 0.6rem;
    border: 0;
    background: none;
    font-size: 1.2rem;
    color: var(--ink-3);
    cursor: pointer;
  }
  .chips {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 0.35rem;
    margin: 0.5rem 0;
    font-size: 0.78rem;
  }
  .chips .label {
    width: 3.5rem;
    color: var(--ink-3);
  }
  .chip {
    font: inherit;
    padding: 0.1rem 0.55rem;
    border: 0;
    border-radius: 999px;
    background: var(--wash);
    color: var(--ink);
    cursor: pointer;
  }
  .chip:hover {
    background: var(--rule);
  }
  .more,
  .events {
    color: var(--ink-3);
    font-size: 0.78rem;
  }
  .events {
    margin: 0 0 0.6rem;
  }
  .card a {
    font-size: 0.85rem;
  }

  /* Bottom left, over the era names column: the overview bar has the right
     edge. The hint sits above the zoom buttons. */
  .zoom-buttons {
    left: max(1.5rem, env(safe-area-inset-left));
    right: auto;
  }
  .corner {
    position: absolute;
    left: max(1.5rem, env(safe-area-inset-left));
    bottom: calc(max(1.6rem, env(safe-area-inset-bottom)) + 2.9rem);
    max-width: 11rem;
    font-size: 0.75rem;
    color: var(--ink-3);
    pointer-events: none;
  }
  .corner p {
    margin: 0;
  }

  /* Phones: the card becomes a bottom sheet; the corner hint gives way to the lanes. */
  @media (max-width: 700px) {
    .card {
      left: 0.75rem !important;
      right: 0.75rem;
      top: auto !important;
      bottom: 5.5rem;
      width: auto;
    }
    .corner {
      display: none;
    }
  }
</style>
