---
title: The Survival Matrix
toc: false
---

# Stages of Survival: US Tech Layoffs by Funding Stage

```js
import * as d3 from "npm:d3";
```

```js
const raw = await FileAttachment("data/Cleaned_tech_layoffs.csv").csv({typed: true});

const processed = raw
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
```

```js
//mutable is how frameworks hold state for rerendering 
const dateRange = Mutable([
  d3.min(processed, d => d.date),
  d3.max(processed, d => d.date)
]);
const setDateRange = r => { dateRange.value = r; };
```

```js
const selectedCompany = Mutable(null);
const setSelectedCompany = c => { selectedCompany.value = c; };
```

```js
const allIndustries = ["All Industries",
  ...Array.from(new Set(processed.map(d => d.industry))).sort()
];
```

```js
//dropdown for industry 
const selectedIndustry = view(Inputs.select(allIndustries, {label: "Industry"}));
```

```js
const filtered = processed.filter(d =>
  d.date >= dateRange[0] && d.date <= dateRange[1] &&
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
    { color: "#3b82f6", title: ["Capital-Efficient", "Survivors"],  sub: "Startup, Low Layoffs (<50%)" },
    { color: "#f59e0b", title: ["Funded but", "Struggling"],        sub: "Established, High layoffs (≥50%)" },
    { color: "#22c55e", title: ["Funded and", "Resilient"],         sub: "Established,  Low Layoffs (<50%)" },
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


---

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
