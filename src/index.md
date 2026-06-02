---
toc: false
---

<h1>Global Tech Layoffs Over Time</h1>
<p class="subtitle">An interactive exploration of layoffs across tech companies worldwide from 2020–2025</p>

```js
import * as d3 from "npm:d3";
const raw = await FileAttachment("data/layoffs.csv").csv({ typed: true });
const data = raw.filter(d => d.Date_layoffs && d.Laid_Off > 0);
```

---

<h2 class="section-title">Layoffs Over Time</h2>
<p class="section-sub">Brush the timeline to explore changes in layoffs across different time periods</p>

```js
const brushRange = Mutable([null, null]);
const setBrush = (range) => { brushRange.value = range; };
```

```js
const parseDate = d3.timeParse("%Y-%m-%d");
const formatMonth = d3.timeFormat("%Y-%m");

const byMonth = d3.rollups(
  data,
  v => d3.sum(v, d => d.Laid_Off),
  d => formatMonth(typeof d.Date_layoffs === "string" ? parseDate(d.Date_layoffs) : d.Date_layoffs)
)
.map(([month, total]) => ({ date: new Date(month + "-01"), total }))
.sort((a, b) => a.date - b.date);
```

```js
const margin = { top: 20, right: 30, bottom: 35, left: 70 };
const W = width - margin.left - margin.right;
const H = 280;

const svg = d3.create("svg").attr("width", width).attr("height", H + margin.top + margin.bottom);

const container = htl.html`
  <style>
    #tooltip-timeline {
      font: 10pt sans-serif;
      background-color: white;
      border: 1pt solid grey;
      padding: 5px;
      box-shadow: 3px 3px 3px darkgrey;
      max-width: 40ch;
      z-index: 1;
      visibility: hidden;
      position: absolute;
      pointer-events: none;
    }
  </style>
  <div id="tooltip-timeline"></div>
  ${svg.node()}
`;

const tooltip = d3.select(container).select("#tooltip-timeline");

const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

const x = d3.scaleTime().domain(d3.extent(byMonth, d => d.date)).range([0, W]);
const y = d3.scaleLinear().domain([0, d3.max(byMonth, d => d.total)]).nice().range([H, 0]);

const area = d3.area().x(d => x(d.date)).y0(H).y1(d => y(d.total)).curve(d3.curveMonotoneX);
const line = d3.line().x(d => x(d.date)).y(d => y(d.total)).curve(d3.curveMonotoneX);

g.append("g")
  .attr("transform", `translate(0,${H})`)
  .call(d3.axisBottom(x).ticks(d3.timeMonth.every(3)).tickFormat(d3.timeFormat("%b %Y")))
  .selectAll("text").style("text-anchor", "middle");

g.append("g").call(d3.axisLeft(y).ticks(6).tickFormat(d => d3.format(",")(d)));

g.append("text").attr("x", -H / 2).attr("y", -50).attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle").style("font-size", "12px").text("Employees Laid Off");

g.append("path").datum(byMonth).attr("fill", "steelblue").attr("fill-opacity", 0.3).attr("d", area);
g.append("path").datum(byMonth).attr("fill", "none").attr("stroke", "steelblue").attr("stroke-width", 2).attr("d", line);

// Vertical line for tooltip crosshair
const hoverLine = g.append("line")
  .attr("class", "hover-line")
  .attr("y1", 0).attr("y2", H)
  .attr("stroke", "#999").attr("stroke-width", 1)
  .attr("stroke-dasharray", "4")
  .style("visibility", "hidden");

const bisect = d3.bisector(d => d.date).left;

const brush = d3.brushX()
  .extent([[0, 0], [W, H]])
  .on("brush end", ({ selection }) => {
    setBrush(selection ? selection.map(x.invert) : [null, null]);
  });

const brushG = g.append("g").attr("class", "brush").call(brush);

// Attach tooltip to the brush overlay so it works alongside brushing
brushG.select(".overlay")
  .on("mousemove", function(event) {
    const [mx] = d3.pointer(event);
    const date = x.invert(mx);
    const i = bisect(byMonth, date);
    const d = byMonth[Math.min(i, byMonth.length - 1)];
    if (!d) return;
    hoverLine.attr("x1", x(d.date)).attr("x2", x(d.date)).style("visibility", "visible");
    tooltip.style("visibility", "visible")
      .style("left", (event.pageX + 12) + "px")
      .style("top", (event.pageY - 28) + "px")
      .html(`<strong>${d3.timeFormat("%B %Y")(d.date)}</strong><br/>Laid off: ${d3.format(",")(d.total)}`);
  })
  .on("mouseout", function() {
    hoverLine.style("visibility", "hidden");
    tooltip.style("visibility", "hidden");
  });
display(container);
```

```js
const [start, end] = brushRange;
const brushFiltered = (start && end)
  ? data.filter(d => {
      const t = typeof d.Date_layoffs === "string" ? new Date(d.Date_layoffs) : d.Date_layoffs;
      return t >= start && t <= end;
    })
  : data;

const totalLaidOff = d3.sum(brushFiltered, d => d.Laid_Off);
const numCompanies = new Set(brushFiltered.map(d => d.Company)).size;
const peakCompany = d3.rollups(brushFiltered, v => d3.sum(v, d => d.Laid_Off), d => d.Company)
  .sort((a, b) => b[1] - a[1])[0];

const formatDate = d3.timeFormat("%b %Y");
const dateRange = (start && end) ? `${formatDate(start)} – ${formatDate(end)}` : "All Time";

const statsDiv = html`<div class="stats-row">
  <div class="stat-card"><div class="stat-value">${dateRange}</div><div class="stat-label">Selected Range</div></div>
  <div class="stat-card"><div class="stat-value">${d3.format(",")(totalLaidOff)}</div><div class="stat-label">Total Amount Laid Off</div></div>
  <div class="stat-card"><div class="stat-value">${numCompanies}</div><div class="stat-label">Companies Affected</div></div>
  <div class="stat-card"><div class="stat-value">${peakCompany ? peakCompany[0] : "-"}</div><div class="stat-label">Peak Company</div></div>
</div>`;
display(statsDiv);
```

---

<h2 class="section-title">Layoffs by State</h2>
<p class="section-sub">Brush the timeline above to update the map.</p>

```js
// Filter to US only for the map and bar chart, using the selected timeline range from the top chart
const usData = brushFiltered.filter(d =>
  d.Country === "USA" &&
  d.USState &&
  String(d.USState).trim() !== "" &&
  d.Laid_Off > 0
);
```

```js
const stateNameToAbbr = {
  "Alabama":"AL","Alaska":"AK","Arizona":"AZ","Arkansas":"AR","California":"CA",
  "Colorado":"CO","Connecticut":"CT","Delaware":"DE","Florida":"FL","Georgia":"GA",
  "Hawaii":"HI","Idaho":"ID","Illinois":"IL","Indiana":"IN","Iowa":"IA",
  "Kansas":"KS","Kentucky":"KY","Louisiana":"LA","Maine":"ME","Maryland":"MD",
  "Massachusetts":"MA","Michigan":"MI","Minnesota":"MN","Mississippi":"MS",
  "Missouri":"MO","Montana":"MT","Nebraska":"NE","Nevada":"NV","New Hampshire":"NH",
  "New Jersey":"NJ","New Mexico":"NM","New York":"NY","North Carolina":"NC",
  "North Dakota":"ND","Ohio":"OH","Oklahoma":"OK","Oregon":"OR","Pennsylvania":"PA",
  "Rhode Island":"RI","South Carolina":"SC","South Dakota":"SD","Tennessee":"TN",
  "Texas":"TX","Utah":"UT","Vermont":"VT","Virginia":"VA","Washington":"WA",
  "West Virginia":"WV","Wisconsin":"WI","Wyoming":"WY","District of Columbia":"DC"
};

const yearFiltered = usData;
const usTotalLaidOff = d3.sum(yearFiltered, d => d.Laid_Off);

const byState = d3.rollups(
  yearFiltered,
  v => {
    const total = d3.sum(v, d => d.Laid_Off);
    const companiesAffected = new Set(v.map(d => d.Company)).size;

    const topCompany = d3.rollups(
      v,
      rows => d3.sum(rows, d => d.Laid_Off),
      d => d.Company
    ).sort((a, b) => b[1] - a[1])[0];

    return {
      total,
      companiesAffected,
      topCompany: topCompany ? topCompany[0] : null,
      topCompanyTotal: topCompany ? topCompany[1] : 0
    };
  },
  d => d.USState
);

const stateMap = new Map(byState.map(([state, val]) => [state, val]));
```

```js
const us = await fetch("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json").then(r => r.json());
const { feature } = await import("npm:topojson-client");
const states = feature(us, us.objects.states);
```

```js
const validStates = new Set(["Alabama","Alaska","Arizona","Arkansas","California","Colorado","Connecticut","Delaware","Florida","Georgia","Hawaii","Idaho","Illinois","Indiana","Iowa","Kansas","Kentucky","Louisiana","Maine","Maryland","Massachusetts","Michigan","Minnesota","Mississippi","Missouri","Montana","Nebraska","Nevada","New Hampshire","New Jersey","New Mexico","New York","North Carolina","North Dakota","Ohio","Oklahoma","Oregon","Pennsylvania","Rhode Island","South Carolina","South Dakota","Tennessee","Texas","Utah","Vermont","Virginia","Washington","West Virginia","Wisconsin","Wyoming","District of Columbia"]);

const stateList = ["All States", ...Array.from(validStates).sort()];

const stateSelector = Inputs.select(stateList, {
  label: "State",
  value: "All States"
});

const selectedState = view(stateSelector);

function setPickedStateFromMap(state) {
  stateSelector.value = state;
  stateSelector.dispatchEvent(new InputEvent("input", { bubbles: true }));
}
```

```js
const mapH = 520;
const mapInnerH = 480;
const maxVal = d3.max(byState, ([, v]) => v.total) || 1;
const color = d3.scaleSequentialPow(t => d3.interpolateBlues(0.08 + 0.92 * t))
  .exponent(0.35)
  .domain([0, maxVal]);
const projection = d3.geoAlbersUsa().fitSize([width, mapInnerH], states);
const path = d3.geoPath().projection(projection);

const mapSvg = d3.create("svg").attr("width", width).attr("height", mapH);

const mapContainer = htl.html`
  <style>
    #map-wrapper {
      position: relative;
    }

    #tooltip-map {
      font: 13px sans-serif;
      background: rgba(255, 255, 255, 0.96);
      border: 1px solid #d6d6d6;
      border-radius: 6px;
      padding: 8px 10px;
      box-shadow: 0 3px 10px rgba(0, 0, 0, 0.18);
      line-height: 1.25;
      color: #222;
      min-width: 145px;
      z-index: 10;
      visibility: hidden;
      display: none;
      position: absolute;
      pointer-events: none;
    }

    #tooltip-map .tooltip-title {
      font-weight: 700;
      margin-bottom: 5px;
    }

    #tooltip-map .tooltip-row {
      display: flex;
      justify-content: space-between;
      gap: 14px;
    }
  </style>

  <div id="map-wrapper">
    <div id="tooltip-map"></div>
    ${mapSvg.node()}
  </div>
`;

const mapTooltip = d3.select(mapContainer).select("#tooltip-map");
const mapWrapper = d3.select(mapContainer).select("#map-wrapper").node();

function attachMapTooltip(selection, tooltip, htmlFn) {
  function positionTooltip(event) {
    const rect = mapWrapper.getBoundingClientRect();

    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;

    tooltip
      .style("left", `${x + 10}px`)
      .style("top", `${y + 10}px`);
  }

  selection
    .on("mouseover", function(event, d) {
      d3.select(this).attr("stroke", "#333").attr("stroke-width", 1.5);

      tooltip
        .html(htmlFn(d));

      positionTooltip(event);

      tooltip
        .style("visibility", "visible")
        .style("display", "block");
    })
    .on("mousemove", function(event) {
      positionTooltip(event);
    })
    .on("mouseout", function(event,d) {
      d3.select(this)
      .attr("stroke", d.properties.name === selectedState ? "#111" : "#54555c")
      .attr("stroke-width", d.properties.name === selectedState ? 2 : 0.6);

      tooltip
        .style("visibility", "hidden")
        .style("display", "none");
    });
}

mapSvg.selectAll("path.state")
  .data(states.features)
  .join(
    enter => enter.append("path")
      .attr("class", "state")
      .attr("d", path)
      .attr("fill", d => {
        const abbr = stateNameToAbbr[d.properties.name];
        const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
        return color(val ? val.total : 0);
      })
      .attr("stroke", d => d.properties.name === selectedState ? "#111" : "#54555c")
      .attr("stroke-width", d => d.properties.name === selectedState ? 2 : 0.6)
      .call(sel => attachMapTooltip(sel, mapTooltip, d => {
        const abbr = stateNameToAbbr[d.properties.name];
        const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
        return `
        <div class="tooltip-title">${d.properties.name}</div>

        <div class="tooltip-row">
          <span>Total laid off</span>
          <strong>${val ? d3.format(",")(val.total) : 0}</strong>
        </div>

        <div class="tooltip-row">
          <span>Share of U.S.</span>
          <strong>${val && usTotalLaidOff ? d3.format(".1%")(val.total / usTotalLaidOff) : "0.0%"}</strong>
        </div>

        <div class="tooltip-row">
          <span>Companies affected</span>
          <strong>${val ? d3.format(",")(val.companiesAffected) : 0}</strong>
        </div>

        <div class="tooltip-row">
          <span>Top company</span>
          <strong>${val && val.topCompany ? val.topCompany : "None"}</strong>
        </div>
      `;
      }))
    .on("click", function(event, d) {
      const state = d.properties.name;

      mapTooltip
      .style("visibility", "hidden")
      .style("display", "none");

      setPickedStateFromMap(selectedState === state ? "All States" : state);
    }),
    update => update
    .attr("fill", d => {
      const abbr = stateNameToAbbr[d.properties.name];
      const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
      return color(val ? val.total : 0);
    })
    .attr("stroke", d => d.properties.name === selectedState ? "#111" : "#54555c")
    .attr("stroke-width", d => d.properties.name === selectedState ? 2 : 0.6),
    exit => exit.remove()
  );

const legendW = 240;
const legendH = 10;
const legendX = width - legendW - 80;
const legendY = mapInnerH + 9;

const defs = mapSvg.append("defs");

const grad = defs.append("linearGradient")
  .attr("id", "legend-grad")
  .attr("x1", "0%")
  .attr("x2", "100%")
  .attr("y1", "0%")
  .attr("y2", "0%");

grad.selectAll("stop")
  .data(d3.range(0, 1.01, 0.1))
  .join("stop")
  .attr("offset", d => `${d * 100}%`)
  .attr("stop-color", d => color(d * maxVal));

const lg = mapSvg.append("g")
  .attr("class", "map-legend")
  .attr("transform", `translate(${legendX}, ${legendY})`);

lg.append("text")
  .attr("x", 0)
  .attr("y", -8)
  .style("font-size", "12px")
  .style("font-weight", "600")
  .style("fill", "#333")
  .text("Total layoffs by state");

lg.append("rect")
  .attr("width", legendW)
  .attr("height", legendH)
  .attr("rx", 2)
  .style("fill", "url(#legend-grad)");

const legendScale = d3.scaleLinear()
  .domain([0, maxVal])
  .range([0, legendW]);

const legendAxis = d3.axisBottom(legendScale)
  .tickValues([0, maxVal / 2, maxVal])
  .tickFormat(d => d3.format(",.0f")(d))
  .tickSize(4);

lg.append("g")
  .attr("transform", `translate(0, ${legendH})`)
  .call(legendAxis)
  .call(g => g.select(".domain").remove())
  .call(g => g.selectAll("text")
    .style("font-size", "10px")
    .style("fill", "#555"))
  .call(g => g.selectAll("line")
    .style("stroke", "#777"));
display(mapContainer);
```

---

<h2 class="section-title">Top Companies by State</h2>
<p class="section-sub">Click a state on the map, or use the dropdown, to see which companies laid off the most workers there.</p>

```js
display(stateSelector);
```

```js
const companySource = selectedState === "All States"
  ? usData
  : usData.filter(d => d.USState === selectedState);

const companyData = d3.rollups(
  companySource,
  v => d3.sum(v, d => d.Laid_Off),
  d => d.Company
)
.map(([company, total]) => ({ company, total }))
.sort((a, b) => b.total - a.total)
.slice(0, 15);
```

```js
const barMargin = { top: 10, right: 30, bottom: 30, left: 160 };
const barW = width - barMargin.left - barMargin.right;
const barH = companyData.length * 28;

const barSvg = d3.create("svg").attr("width", width).attr("height", barH + barMargin.top + barMargin.bottom);

const barContainer = htl.html`
  <style>
    #bar-wrapper {
      position: relative;
    }

    #tooltip-bar {
      font: 13px sans-serif;
      background: rgba(255, 255, 255, 0.96);
      border: 1px solid #d6d6d6;
      border-radius: 6px;
      padding: 8px 10px;
      box-shadow: 0 3px 10px rgba(0, 0, 0, 0.18);
      line-height: 1.25;
      color: #222;
      min-width: 165px;
      z-index: 10;
      visibility: hidden;
      display: none;
      position: absolute;
      pointer-events: none;
    }

    #tooltip-bar .tooltip-title {
      font-weight: 700;
      margin-bottom: 5px;
    }

    #tooltip-bar .tooltip-row {
      display: flex;
      justify-content: space-between;
      gap: 14px;
    }
  </style>

  <div id="bar-wrapper">
    <div id="tooltip-bar"></div>
    ${barSvg.node()}
  </div>
`;

const barTooltip = d3.select(barContainer).select("#tooltip-bar");
const barWrapper = d3.select(barContainer).select("#bar-wrapper").node();

const selectedStateTotal = d3.sum(companyData, d => d.total);

function attachBarTooltip(selection, tooltip, htmlFn) {
  function positionTooltip(event) {
    const rect = barWrapper.getBoundingClientRect();

    let x = event.clientX - rect.left + 10;
    let y = event.clientY - rect.top + 10;

    const tipWidth = 230;
    const tipHeight = 120;

    if (x + tipWidth > rect.width) x = event.clientX - rect.left - tipWidth - 10;
    if (y + tipHeight > rect.height) y = event.clientY - rect.top - tipHeight - 10;

    tooltip
      .style("left", `${Math.max(6, x)}px`)
      .style("top", `${Math.max(6, y)}px`);
  }

  selection
    .on("mouseover", function(event, d) {
      d3.select(this).attr("fill", "#2a5f8f");

      tooltip.html(htmlFn(d));
      positionTooltip(event);

      tooltip
        .style("visibility", "visible")
        .style("display", "block");
    })
    .on("mousemove", function(event) {
      positionTooltip(event);
    })
    .on("mouseout", function() {
      d3.select(this).attr("fill", "steelblue");

      tooltip
        .style("visibility", "hidden")
        .style("display", "none");
    });
}

const bg = barSvg.append("g").attr("transform", `translate(${barMargin.left},${barMargin.top})`);

const xBar = d3.scaleLinear().domain([0, d3.max(companyData, d => d.total) || 1]).range([0, barW]);
const yBar = d3.scaleBand().domain(companyData.map(d => d.company)).range([0, barH]).padding(0.2);

bg.append("g").call(d3.axisLeft(yBar).tickSize(0)).select(".domain").remove();
bg.append("g").attr("transform", `translate(0,${barH})`).call(d3.axisBottom(xBar).ticks(5).tickFormat(d3.format(",")));

bg.selectAll("rect.bar")
  .data(companyData)
  .join(
    enter => enter.append("rect")
      .attr("class", "bar")
      .attr("y", d => yBar(d.company))
      .attr("width", d => xBar(d.total))
      .attr("height", yBar.bandwidth())
      .attr("fill", "steelblue")
      .attr("rx", 3)
      .call(sel => attachBarTooltip(sel, barTooltip, d => {
  const rank = companyData.findIndex(x => x.company === d.company) + 1;
  const share = selectedStateTotal ? d.total / selectedStateTotal : 0;

  return `
      <div class="tooltip-title">${d.company}</div>

      <div class="tooltip-row">
        <span>State</span>
        <strong>${selectedState === "All States" ? "All U.S." : selectedState}</strong>
      </div>

      <div class="tooltip-row">
        <span>Rank</span>
        <strong>#${rank}</strong>
      </div>

      <div class="tooltip-row">
        <span>Laid off</span>
        <strong>${d3.format(",")(d.total)}</strong>
      </div>

      <div class="tooltip-row">
        <span>Share</span>
        <strong>${d3.format(".1%")(share)}</strong>
      </div>
    `;
    })),
    update => update
      .attr("y", d => yBar(d.company))
      .attr("width", d => xBar(d.total))
      .attr("height", yBar.bandwidth()),
    exit => exit.remove()
  );

display(barContainer);
```

---

<h2 class="section-title">Layoffs by Stage</h2>
<p class="section-sub">Select a year and metric to explore how layoffs shifted across funding stages over time."</p>

```js
const rawHeat = await FileAttachment("data/Cleaned_tech_layoffs.csv").csv({ typed: true });
const heatData = rawHeat.filter(d => d.Stage && d.Stage.trim() !== "" && d.Laid_Off > 0 && d.Year >= 2020 && d.Year <= 2025);
```
```js
const heatMetric = Inputs.select(
  new Map([
    ["Total laid off", "total"],
    ["% of workforce cut", "pct"],
    ["Number of events", "events"]
  ]),
  { label: "Metric", value: "total" }
);
const metric = Generators.input(heatMetric);

const heatYear = Inputs.select(
  ["All", "2020", "2021", "2022", "2023", "2024", "2025"],
  { label: "Year", value: "All" }
);
const heatYearVal = Generators.input(heatYear);

display(htl.html`<div style="display:flex;gap:2rem;align-items:center;margin-bottom:1rem;">
  ${heatMetric}
  ${heatYear}
</div>`);
```

```js
// stage ordering
const stageOrder = [
  "Pre-seed", "Seed", "Series A", "Series B", "Series C",
  "Series D", "Series E", "Series F", "Series G", "Series H",
  "Private", "Public", "Acquired", "Unknown"
];

// normalize stage values into cleaner (more conventional naming) buckets
function normalizeStage(raw) {
  if (!raw) return "Unknown";
  const s = raw.trim();
  if (/^pre.?seed$/i.test(s)) return "Pre-seed";
  if (/^seed$/i.test(s)) return "Seed";
  if (/^series\s*a$/i.test(s)) return "Series A";
  if (/^series\s*b$/i.test(s)) return "Series B";
  if (/^series\s*c$/i.test(s)) return "Series C";
  if (/^series\s*d$/i.test(s)) return "Series D";
  if (/^series\s*e$/i.test(s)) return "Series E";
  if (/^series\s*f$/i.test(s)) return "Series F";
  if (/^series\s*[gh]$/i.test(s)) return "Series G";
  if (/public|ipo|listed/i.test(s)) return "Public";
  if (/private/i.test(s)) return "Private";
  if (/acqui/i.test(s)) return "Acquired";
  return "Unknown";
}

const quarters = ["Q1", "Q2", "Q3", "Q4"];

function getQuarter(d) {
  const date = typeof d.Date_layoffs === "string" ? new Date(d.Date_layoffs) : d.Date_layoffs;
  if (!date || isNaN(date)) return null;
  return `Q${Math.floor(date.getMonth() / 3) + 1}`;
}

const heatYearFiltered = heatYearVal === "All"
  ? heatData
  : heatData.filter(d => String(d.Year) === heatYearVal);

const colKeys = heatYearVal === "All"
  ? [2020, 2021, 2022, 2023, 2024, 2025].flatMap(y => quarters.map(q => `${y}-${q}`))
  : quarters;

const rollup = new Map();

for (const d of heatYearFiltered) {
  const stage = normalizeStage(d.Stage);
  const q = getQuarter(d);
  if (!q) continue;
  const col = heatYearVal === "All" ? `${d.Year}-${q}` : q;
  const key = `${stage}|${col}`;
  if (!rollup.has(key)) rollup.set(key, { total: 0, pctSum: 0, pctCount: 0, events: 0, companies: new Map() });
  const r = rollup.get(key);
  r.total += d.Laid_Off || 0;
  if (d.Percentage > 0) { r.pctSum += d.Percentage; r.pctCount++; }
  r.events += 1;
  r.companies.set(d.Company, (r.companies.get(d.Company) || 0) + (d.Laid_Off || 0));
}

function getCellValue(stage, col) {
  const r = rollup.get(`${stage}|${col}`);
  if (!r) return 0;
  if (metric === "total") return r.total;
  if (metric === "pct") return r.pctCount > 0 ? r.pctSum / r.pctCount : 0;
  return r.events;
}

// only show stages that appear in the filtered data
const activeStages = stageOrder.filter(st =>
  colKeys.some(col => getCellValue(st, col) > 0)
);

const allValues = activeStages.flatMap(st => colKeys.map(col => getCellValue(st, col)));
const heatMaxVal = d3.max(allValues) || 1;

// match existing color pallette 
const heatColor = d3.scaleSequential(d3.interpolateBlues).domain([0, heatMaxVal]);

function cellTextColor(val) {
  return val / heatMaxVal > 0.6 ? "#fff" : "#1a1a2e";
}

function formatCellVal(val) {
  if (metric === "total") return val === 0 ? "" : val >= 1000 ? `${Math.round(val / 1000)}k` : Math.round(val);
  if (metric === "pct") return val === 0 ? "" : `${val.toFixed(1)}%`;
  return val === 0 ? "" : val;
}

function formatTooltipVal(val) {
  if (metric === "total") return d3.format(",")(Math.round(val));
  if (metric === "pct") return `${val.toFixed(1)}%`;
  return val;
}

const metricLabel = { total: "Total laid off", pct: "Avg % of workforce", events: "Layoff events" }[metric];
```

```js
const cellH = 36;
const cellW = heatYearVal === "All" ? Math.max(28, Math.floor((width - 140) / colKeys.length)) : Math.floor((width - 140) / 4);
const labelW = 140;
const headerH = heatYearVal === "All" ? 44 : 28;
const totalH = headerH + activeStages.length * cellH + 40;

const heatSvg = d3.create("svg")
  .attr("width", width)
  .attr("height", totalH)
  .attr("role", "img")
  .attr("aria-label", `Heatmap of tech layoffs by funding stage and quarter, metric: ${metricLabel}`);

const heatContainer = htl.html`
  <div style="position: relative;">
    #tooltip-heat {
      font: 10pt sans-serif;
      background-color: white;
      border: 1pt solid grey;
      padding: 5px;
      box-shadow: 3px 3px 3px darkgrey;
      max-width: 40ch;
      z-index: 1;
      visibility: hidden;
      position: absolute;
      pointer-events: none;
    }
  </style>
  <div id="tooltip-heat"></div>
  ${heatSvg.node()}
`;

const heatTooltip = d3.select(heatContainer).select("#tooltip-heat");

if (heatYearVal === "All") {
  [2020, 2021, 2022, 2023, 2024, 2025].forEach((yr, yi) => {
    const x = labelW + yi * 4 * cellW;
    heatSvg.append("text")
      .attr("x", x + 2 * cellW)
      .attr("y", 14)
      .attr("text-anchor", "middle")
      .style("font-size", "12px")
      .style("font-weight", "600")
      .style("fill", "#333")
      .text(yr);

    quarters.forEach((q, qi) => {
      heatSvg.append("text")
        .attr("x", x + qi * cellW + cellW / 2)
        .attr("y", 30)
        .attr("text-anchor", "middle")
        .style("font-size", "10px")
        .style("fill", "#888")
        .text(q);
    });

    if (yi > 0) {
      heatSvg.append("line")
        .attr("x1", x).attr("x2", x)
        .attr("y1", 0).attr("y2", totalH - 40)
        .attr("stroke", "#ccc").attr("stroke-width", 1);
    }
  });
} else {
  quarters.forEach((q, qi) => {
    heatSvg.append("text")
      .attr("x", labelW + qi * cellW + cellW / 2)
      .attr("y", 18)
      .attr("text-anchor", "middle")
      .style("font-size", "12px")
      .style("fill", "#555")
      .text(q);
  });
}

activeStages.forEach((stage, si) => {
  const y = headerH + si * cellH;

  heatSvg.append("text")
    .attr("x", labelW - 8)
    .attr("y", y + cellH / 2 + 4)
    .attr("text-anchor", "end")
    .style("font-size", "12px")
    .style("fill", "#444")
    .text(stage);

  colKeys.forEach((col, ci) => {
    const val = getCellValue(stage, col);
    const cx = labelW + ci * cellW;

    const cell = heatSvg.append("rect")
      .attr("x", cx + 1).attr("y", y + 1)
      .attr("width", cellW - 2).attr("height", cellH - 2)
      .attr("rx", 3)
      .attr("fill", val === 0 ? "#f0f0f0" : heatColor(val))
      .style("cursor", "pointer");

    if (cellW >= 32) {
      heatSvg.append("text")
        .attr("x", cx + cellW / 2)
        .attr("y", y + cellH / 2 + 4)
        .attr("text-anchor", "middle")
        .style("font-size", "10px")
        .style("fill", val === 0 ? "#bbb" : cellTextColor(val))
        .style("pointer-events", "none")
        .text(formatCellVal(val));
    }

    const colLabel = heatYearVal === "All"
      ? `${col.split("-")[0]} ${col.split("-")[1]}`
      : `${col} ${heatYearVal}`;

    cell
      .on("mouseover", function(event) {
        d3.select(this).attr("stroke", "#333").attr("stroke-width", 1.5);
        heatTooltip.style("visibility", "visible")
          .html(() => {
            const r = rollup.get(`${stage}|${col}`);
            const topCo = r ? [...r.companies.entries()].sort((a,b) => b[1]-a[1])[0] : null;
            const topLine = topCo ? `<br/>Top company: <strong>${topCo[0]}</strong> (${d3.format(",")(topCo[1])})` : "";
            return `<strong>${stage} — ${colLabel}</strong><br/>${metricLabel}: ${formatTooltipVal(val)}${topLine}`;
          });
      })
      .on("mousemove", function(event) {
        const tipWidth = 220;
        const flipped = event.clientX + tipWidth + 12 > window.innerWidth;
        heatTooltip
          .style("left", flipped ? (event.clientX - tipWidth - 12) + "px" : (event.clientX + 12) + "px")
          .style("top", (event.clientY - 28) + "px");
      })
      .on("mouseout", function() {
        d3.select(this).attr("stroke", null);
        heatTooltip.style("visibility", "hidden");
      });
  });
});

const heatLegendW = 180, heatLegendH = 10;
const heatLegendX = labelW;
const heatLegendY = totalH - 30;

const heatDefs = heatSvg.append("defs");
const heatGrad = heatDefs.append("linearGradient").attr("id", "heat-legend-grad");
d3.range(0, 1.01, 0.05).forEach(t => {
  heatGrad.append("stop").attr("offset", `${t * 100}%`).attr("stop-color", heatColor(t * heatMaxVal));
});

heatSvg.append("rect")
  .attr("x", heatLegendX).attr("y", heatLegendY)
  .attr("width", heatLegendW).attr("height", heatLegendH)
  .attr("rx", 2)
  .style("fill", "url(#heat-legend-grad)");

heatSvg.append("text")
  .attr("x", heatLegendX).attr("y", heatLegendY + heatLegendH + 12)
  .style("font-size", "10px").style("fill", "#888")
  .text(metric === "total" ? "0" : metric === "pct" ? "0%" : "0");

heatSvg.append("text")
  .attr("x", heatLegendX + heatLegendW).attr("y", heatLegendY + heatLegendH + 12)
  .attr("text-anchor", "end")
  .style("font-size", "10px").style("fill", "#888")
  .text(metric === "total"
    ? d3.format(",")(Math.round(heatMaxVal))
    : metric === "pct"
    ? `${heatMaxVal.toFixed(1)}%`
    : Math.round(heatMaxVal));

display(heatContainer);
```

<h2 class="section-title">Stages of Survival: Layoffs by Funding Stage</h2>
<p class="section-sub">Each dot is a layoff event, sized by number laid off. Drag the timeline to filter by date, click a dot to highlight a company.</p>

```js
const rawScatter = await FileAttachment("data/Cleaned_tech_layoffs.csv").csv({typed: true});

const processed = rawScatter
  .filter(d => d.Country === "USA" && +d.Money_Raised_in__mil > 0 && d.Percentage !== "" && d.Laid_Off !== "" && d.Stage && d.Stage !== "Unknown")
  .map(d => ({
    company:     d.Company,
    industry:    d.Industry || "Unknown",
    stage:       d.Stage    || "Unknown",
    moneyRaised: +d.Money_Raised_in__mil,
    layoffPct:   +d.Percentage,
    laidOff:     +d.Laid_Off,
    date:        new Date(d.Date_layoffs),
  }));

//mutable is how frameworks hold state for rerendering 
const scatterDateRange = Mutable([
  d3.min(processed, d => d.date),
  d3.max(processed, d => d.date)
]);
const setDateRange = r => { scatterDateRange.value = r; };

const selectedCompany = Mutable(null);
const setSelectedCompany = c => { selectedCompany.value = c; };

const allIndustries = ["All Industries",
  ...Array.from(new Set(processed.map(d => d.industry))).sort()
];

//dropdown for industry 
const selectedIndustry = view(Inputs.select(allIndustries, {label: "Industry"}));
```

```js
const filtered = processed.filter(d =>
  d.date >= scatterDateRange[0] && d.date <= scatterDateRange[1] &&
  (selectedIndustry === "All Industries" || d.industry === selectedIndustry)
);
```

Showing **${filtered.length}** US events: drag the timeline below to filter by date.

```js
(() => {
  const W  = width; //reactive page width 
  const H  = 500;
  const M  = {top: 30, right: 200, bottom: 90, left: 70};
  const IW = W - M.left - M.right;
  const IH = H - M.top  - M.bottom;

  const stageOrder = [
    "Seed", "Series A", "Series B", "Series C", "Series D", "Series E",
    "Series F", "Series G", "Series H", "Series I", "Series J",
    "Private Equity", "Post-IPO", "Acquired", "Subsidiary"
  ];

  const xSc = d3.scaleBand() //to divide evenly 
    .domain(stageOrder)
    .range([0, IW])
    .padding(0.1); //for 1% spacing between bands

  // Deterministic jitter so dots don't rearrange on every re-render
  //using djb2 hash 
  function jitter(str, bw) {
    let h = 5381;
    for (let i = 0; i < str.length; i++) h = ((h << 5) + h) ^ str.charCodeAt(i);
    return ((h >>> 0) / 4294967295 - 0.5) * bw * 0.75;
  }

  const ySc = d3.scaleLinear()
    .domain([0, 105])
    .range([IH, 0]);

  const rSc = d3.scaleSqrt()
    .domain([0, d3.max(processed, d => d.laidOff)]) // processed to keeps scale stable during brush
    .range([2, 22]);

  // Color by quadrant by stage and laif off %
  const established = new Set(["Post-IPO", "Private Equity", "Acquired", "Subsidiary"]);
  function quadColor(stage, pct) {
    if (pct >= 50 && !established.has(stage)) return "#ef4444";  // startup high lay off is red
    if (pct >= 50 &&  established.has(stage)) return "#f59e0b";  // established high layoff amber
    if (pct <  50 && !established.has(stage)) return "#3b82f6";  // startup low layoff  blue
    return "#22c55e";                                             // established low layoff green
  }

  const container = d3.create("div").style("position", "relative");

  //the tooltip 
  const tip = container.append("div")
    .style("position", "absolute") //freely at any x y pixel in container
    .style("pointer-events", "none") //nothing if you click on it with mouse
    .style("background", "rgba(15,23,42,0.92)")
    .style("color", "#f1f5f9")
    .style("padding", "10px 14px")
    .style("border-radius", "7px") // rounded corners
    .style("font-size", "13px")
    .style("line-height", "1.65") // added line spacking because it was cramped
    .style("display", "none") // onlye popus up on mouse 
    .style("max-width", "220px") // capped width for long company names
    .style("z-index", "10")
    .style("box-shadow", "0 4px 12px rgba(0,0,0,0.25)");

  const svg = container.append("svg").attr("width", W).attr("height", H).style("overflow", "visible");
  const g   = svg.append("g").attr("transform", `translate(${M.left},${M.top})`); //margin and svg adjustment

  // Chart title
  svg.append("text")
   .attr("x", W / 2).attr("y", 20)
   .attr("text-anchor", "middle")
   .style("font-size", "15px").style("font-weight", "700")
   .style("fill", "#1e293b")
   .text("Stages of Survival: US Tech Layoffs across Industries by Funding Stage");

  // Divider for x and y axis to make qudrants
  const midX = xSc("Private Equity");
  const midY = ySc(50);

  // Quadrant background shading 
  g.append("rect").attr("x", 0).attr("y", 0)
   .attr("width", midX).attr("height", midY)
   .attr("fill", "#fef2f2").attr("opacity", 0.55);   // red   startup high 
  g.append("rect").attr("x", midX).attr("y", 0)
   .attr("width", IW - midX).attr("height", midY)
   .attr("fill", "#fffbeb").attr("opacity", 0.55);   // amber establsihed high
  g.append("rect").attr("x", 0).attr("y", midY)
   .attr("width", midX).attr("height", IH - midY)
   .attr("fill", "#f0fdfa").attr("opacity", 0.55);   // teal  startup low
  g.append("rect").attr("x", midX).attr("y", midY)
   .attr("width", IW - midX).attr("height", IH - midY)
   .attr("fill", "#f0fdf4").attr("opacity", 0.55);   // green established low

  // Dashed divider lines
  g.append("line")
   .attr("x1", 0).attr("x2", IW).attr("y1", midY).attr("y2", midY)
   .attr("stroke", "#cbd5e1").attr("stroke-dasharray", "5,4").attr("stroke-width", 1.5);
  g.append("line")
   .attr("x1", midX).attr("x2", midX).attr("y1", 0).attr("y2", IH)
   .attr("stroke", "#cbd5e1").attr("stroke-dasharray", "5,4").attr("stroke-width", 1.5);


  // X axis 
  g.append("g")
   .attr("transform", `translate(0,${IH})`)
   .call(d3.axisBottom(xSc))
   .selectAll("text")
   .attr("transform", "rotate(-40)")
   .style("text-anchor", "end")
   .attr("dx", "-0.5em")
   .attr("dy", "0.15em");

  // Y axis
  g.append("g")
   .call(d3.axisLeft(ySc).tickFormat(d => `${d}%`).ticks(6));

  // Axis labels
  g.append("text")
   .attr("x", IW / 2).attr("y", IH + 82)
   .attr("text-anchor", "middle")
   .style("font-size", "13px")
   .text("Funding Stage");

  g.append("text")
   .attr("transform", "rotate(-90)")
   .attr("x", -(IH / 2)).attr("y", -55)
   .attr("text-anchor", "middle")
   .style("font-size", "13px")
   .text("% of Workforce Laid Off");

  // Dots
  g.append("g")
   .selectAll("circle")
   .data(filtered.slice().sort((a, b) => b.laidOff - a.laidOff)) // sorted to order bigger circle before smaller
   .join("circle")
   .attr("cx", d => xSc(d.stage) + xSc.bandwidth() / 2 + jitter(d.company + d.date, xSc.bandwidth()))
   .attr("cy", d => ySc(d.layoffPct))
   .attr("r", d => rSc(d.laidOff))
   .attr("fill", d => quadColor(d.stage, d.layoffPct))
   .attr("opacity", d => selectedCompany ? (d.company === selectedCompany ? 1 : 0.08) : 0.60)
   .attr("stroke", d => selectedCompany && d.company === selectedCompany ? "#0f172a" : "white")
   .attr("stroke-width", d => selectedCompany && d.company === selectedCompany ? 1.5 : 0.4)
   .style("cursor", "pointer")
   .on("click", function(event, d) {
     event.stopPropagation();
     setSelectedCompany(selectedCompany === d.company ? null : d.company);
   })
   .on("mouseover", function(event, d) {
     if (!selectedCompany || d.company === selectedCompany)
       d3.select(this).attr("opacity", 1).attr("stroke", "#0f172a").attr("stroke-width", 1.5);
     const [mx, my] = d3.pointer(event, container.node()); // to get mouse position for tooltip
     const fmt = v => v >= 1000 ? `$${(v / 1000).toFixed(1)}B` : `$${v}M`;
     tip.style("display", "block")
        .html(`<strong style="font-size:14px">${d.company}</strong><br>
               <span style="color:#94a3b8">${d.industry} · ${d.stage}</span><br><br>
               Laid off: <strong>${d.laidOff.toLocaleString()}</strong> people (${d.layoffPct.toFixed(0)}%)<br>
               Funding raised: <strong>${fmt(d.moneyRaised)}</strong><br>
               ${d.date.toLocaleDateString("en-US", {month: "long", year: "numeric"})}`)
        .style("left", `${mx + 14}px`)
        .style("top",  `${my - 10}px`);
   })
   .on("mousemove", function(event) {
     const [mx, my] = d3.pointer(event, container.node());
     tip.style("left", `${mx + 14}px`).style("top", `${my - 10}px`); //to be able to select other companies
   })
   .on("mouseout", function(event, d) {
     const base = selectedCompany ? (d.company === selectedCompany ? 1 : 0.08) : 0.60;
     const str  = selectedCompany && d.company === selectedCompany ? "#0f172a" : "white";
     d3.select(this).attr("opacity", base).attr("stroke", str);
     tip.style("display", "none");
   }); 

  // Raise selected company dots to top so they intercept clicks correctly
  if (selectedCompany) {
    g.selectAll("circle").filter(d => d.company === selectedCompany).raise();
  }


  // Color legend on right margin
  const legendEntries = [
    { color: "#ef4444", title: ["Underfunded &", "Exposed"],        sub: "Startup, High Layoffs (≥50%)" },
    { color: "#3b82f6", title: ["Capital-Efficient", "Survivors"],  sub: "Startup, Resilient (<50%)" },
    { color: "#f59e0b", title: ["Funded but", "Struggling"],        sub: "Established, High layoffs (≥50%)" },
    { color: "#22c55e", title: ["Funded and", "Resilient"],         sub: "Established,  Resilient (<50%)" },
  ];
  const legG = svg.append("g")
    .attr("transform", `translate(${M.left + IW + 24}, ${M.top + 10})`);

  legG.append("text")
    .attr("x", 0).attr("y", 0)
    .style("font-size", "11px").style("font-weight", "700")
    .style("fill", "#475569").style("letter-spacing", "0.05em")
    .text("QUADRANT");

  legendEntries.forEach(({ color, title, sub }, i) => {
    const y = 20 + i * 52;
    legG.append("circle")
      .attr("cx", 7).attr("cy", y + 10)
      .attr("r", 7).attr("fill", color).attr("opacity", 0.85);
    const titleEl = legG.append("text")
      .attr("x", 20).attr("y", y + 4)
      .style("font-size", "11px").style("font-weight", "700")
      .style("fill", color);
    titleEl.append("tspan").attr("x", 20).attr("dy", "0").text(title[0]);
    titleEl.append("tspan").attr("x", 20).attr("dy", "14").text(title[1]);
    legG.append("text")
      .attr("x", 20).attr("y", y + 34)
      .style("font-size", "10px").style("fill", "#64748b")
      .text(sub);
  });

  // Click anywhere on the chart background to deselect
  svg.on("click", () => setSelectedCompany(null));

  return container.node();
})()
```

```js
(() => {
  const W  = width;
  const H  = 155;
  const M  = {top: 10, right: 30, bottom: 70, left: 90};
  const IW = W - M.left - M.right;
  const IH = H - M.top  - M.bottom;

  // Always show all data so the context bar doesn't change shape while brushing
  const monthly = Array.from(
    d3.rollup(processed, v => d3.sum(v, d => d.laidOff),
              d => d3.timeMonth.floor(d.date)),
    ([date, total]) => ({date, total})
  ).sort((a, b) => a.date - b.date);

  const xSc = d3.scaleTime()
    .domain([
      d3.timeMonth.floor(d3.min(processed, d => d.date)),
      d3.timeMonth.ceil(d3.max(processed, d => d.date))
    ])
    .range([0, IW]);

  const ySc = d3.scaleLinear()
    .domain([0, d3.max(monthly, d => d.total)])
    .range([IH, 0]);

  const svg = d3.create("svg").attr("width", W).attr("height", H).style("overflow", "visible");
  const g   = svg.append("g").attr("transform", `translate(${M.left},${M.top})`);

  // Monthly bars
  const bw = Math.max(2, IW / monthly.length - 1);
  g.append("g").selectAll("rect").data(monthly).join("rect")
   .attr("x", d => xSc(d.date))
   .attr("y", d => ySc(d.total))
   .attr("width", bw)
   .attr("height", d => IH - ySc(d.total))
   .attr("fill", "#6366f1").attr("opacity", 0.5);

  //  month lines
  const allMonths = d3.timeMonth.range(
    d3.timeMonth.floor(d3.min(processed, d => d.date)),
    d3.timeMonth.ceil(d3.max(processed, d => d.date))
  );
  g.append("g").selectAll("line.month-grid").data(allMonths).join("line")
   .attr("class", "month-grid")
   .attr("x1", d => xSc(d)).attr("x2", d => xSc(d))
   .attr("y1", 0).attr("y2", IH)
   .attr("stroke", "#e2e8f0").attr("stroke-width", 0.5);

  // X axis 
  const xAxisG = g.append("g").attr("transform", `translate(0,${IH})`);

  // quarterly ticks with month names
  xAxisG.call(d3.axisBottom(xSc)
    .ticks(d3.timeMonth.every(3))
    .tickFormat(d3.timeFormat("%b"))
    .tickSize(4));

  // years
  const yearTicks = d3.timeYear.range(
    d3.timeYear.floor(d3.min(processed, d => d.date)),
    d3.timeYear.ceil(d3.max(processed, d => d.date))
  );
  yearTicks.forEach(year => {
    const x1  = Math.max(0, xSc(year));
    const x2  = Math.min(IW, xSc(d3.timeYear.offset(year, 1)));
    const mid = (x1 + x2) / 2;
    const gap = 2; // small gap so brakcets can be distinguished

    xAxisG.append("line")
      .attr("x1", x1 + gap).attr("x2", x2 - gap)
      .attr("y1", 24).attr("y2", 24)
      .attr("stroke", "#64748b").attr("stroke-width", 1.5);

    // Left end cap
    xAxisG.append("line")
      .attr("x1", x1 + gap).attr("x2", x1 + gap)
      .attr("y1", 20).attr("y2", 24)
      .attr("stroke", "#64748b").attr("stroke-width", 1.5);

    // Right end cap
    xAxisG.append("line")
      .attr("x1", x2 - gap).attr("x2", x2 - gap)
      .attr("y1", 20).attr("y2", 24)
      .attr("stroke", "#64748b").attr("stroke-width", 1.5);

    // Centered year label below bracket
    xAxisG.append("text")
      .attr("x", mid).attr("y", 38)
      .attr("text-anchor", "middle")
      .style("font-size", "11px").style("fill", "#475569").style("font-weight", "600")
      .text(d3.timeFormat("%Y")(year));
  });

  // X axis label 
  xAxisG.append("text")
    .attr("x", IW / 2).attr("y", 58)
    .attr("text-anchor", "middle")
    .style("font-size", "11px").style("fill", "#787276").style("font-style", "italic")
    .text("← drag to filter the chart by date range →");

  // Y axis
  g.append("g")
   .call(d3.axisLeft(ySc).ticks(2).tickFormat(d => d >= 1000 ? `${d/1000}k` : d));

  // Y axis label
  ["Total Number of Layoffs", "Across All Industries"].forEach((line, i) =>
    g.append("text")
     .attr("transform", "rotate(-90)")
     .attr("x", -(IH / 2)).attr("y", -68 + i * 12)
     .attr("text-anchor", "middle")
     .style("font-size", "10px").style("fill", "#64748b")
     .text(line)
  );

  // Brush
  const brush = d3.brushX()
    .extent([[0, 0], [IW, IH]])
    .on("brush end", ({selection}) => {
      if (selection) {
        setDateRange(selection.map(v => xSc.invert(v)));
      } else {
        setDateRange([d3.min(processed, d => d.date), d3.max(processed, d => d.date)]);
      }
    });

  const brushG = g.append("g").call(brush);
  brushG.call(brush.move, [0, IW]); // start fully selected

  brushG.select(".selection")
    .style("fill", "#6366f1").style("fill-opacity", 0.15)
    .style("stroke", "#6366f1").style("stroke-width", 1.5);
  brushG.selectAll(".handle")
    .style("fill", "#6366f1").style("opacity", 0.9);

  return svg.node();
})()
```

<style>
h1 { font-size: 2rem; margin-bottom: 0.25rem; pointer-events: none; }
h1 a, h1 a:hover { color: inherit; text-decoration: none; pointer-events: none; }
.subtitle { font-size: 1.1rem; color: #666; margin-top: 0; margin-bottom: 1.5rem; }
.section-title { pointer-events: none; }
.section-title a, .section-title a:hover { color: inherit; text-decoration: none; pointer-events: none; }
.section-sub { color: #555; margin-top: -0.5rem; margin-bottom: 1rem; }
hr { margin: 2rem 0; }
.brush .selection { fill: steelblue; fill-opacity: 0.2; stroke: steelblue; }
.stats-row { display: flex; gap: 1.5rem; margin: 1rem 0 2rem; }
.stat-card { background: #f5f5f5; border-radius: 8px; padding: 1rem 1.5rem; min-width: 140px; text-align: center; }
.stat-value { font-size: 1.8rem; font-weight: 700; color: steelblue; }
.stat-label { font-size: 0.85rem; color: #666; margin-top: 0.25rem; }
</style>