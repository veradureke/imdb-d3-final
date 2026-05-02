# Observable Implementation Reference

**IMDB Top 1000 Interactive Dashboard**
N328 Visualizing Information · Vera Dureke · Spring 2026

This document accompanies the GitHub Pages submission and describes an
alternate implementation of the same visualization in
[Observable](https://observablehq.com), Mike Bostock's reactive notebook
environment. Observable was used during development to prototype individual
charts and verify the reactive filter model before porting the production
version to a static D3 site. The two implementations cover the same dataset
and answer the same four driving questions; this document is intended as a
companion read for anyone interested in how the dashboard would be expressed
in a notebook environment.

## Why two implementations

The final submission uses plain HTML, CSS, and D3.js v7 hosted on GitHub Pages, which gives full control over layout, theming, and the cross-chart
linked-filter behavior. Observable was used in parallel to:

- Prototype each chart in isolation with live data feedback.
- Verify that a reactive dataflow model (one shared filter state, charts as derived views) was the right architecture before committing to it in the static implementation.
- Confirm the data-cleaning logic by inspecting intermediate values cell by cell.

## Architecture: a reactive dataflow graph

Observable runs every cell as a node in a dependency graph. When any input changes, the runtime re-executes only the downstream cells that depend on it. 
The dashboard's reactive structure is:

```
films  ──►  filtered  ──►  scatter, histogram, decadeChart, directorBar, table
                ▲
            (depends on selectedGenre, yearStart, yearEnd, minVotes,
             searchTerm, selectedDirector)
```

Filter inputs feed `filtered`, every chart reads `filtered`, and clicking
a director's bar mutates `selectedDirector`, which re-renders the dependent
charts. There is no manual re-render plumbing — the runtime tracks
dependencies automatically.

## Data pipeline

A single asynchronous cell loads the CSV via Observable's `FileAttachment`
helper and produces a cleaned, typed array of film records:

```js
films = {
  const raw = await FileAttachment("imdb_top_1000.csv").csv();
  return raw
    .map(r => {
      const year = parseInt(r.Released_Year, 10);
      const runtime = parseInt((r.Runtime || "").replace(/\D/g, ""), 10);
      const gross = r.Gross ? +r.Gross.replace(/\D/g, "") : null;
      return {
        title: r.Series_Title,
        year: isNaN(year) ? null : year,
        decade: isNaN(year) ? null : Math.floor(year / 10) * 10,
        runtime: isNaN(runtime) ? null : runtime,
        certificate: r.Certificate || "",
        genres: (r.Genre || "").split(",").map(s => s.trim()).filter(Boolean),
        primaryGenre: ((r.Genre || "").split(",")[0] || "").trim(),
        rating: +r.IMDB_Rating,
        meta: r.Meta_score ? +r.Meta_score : null,
        director: r.Director,
        stars: [r.Star1, r.Star2, r.Star3, r.Star4].filter(Boolean),
        votes: +r.No_of_Votes || 0,
        gross: gross
      };
    })
    .filter(d => d.year && d.rating);
}
```

The same three cleaning rules used in the production version apply: drop
the row with `"PG"` in `Released_Year`, parse `Runtime` strings like
`"142 min"` to integer minutes, and parse `Gross` strings with comma
separators to numbers.

A genre color scale is derived once from the data, ordered so the most
common primary genres get the most visually distinct hues:

```js
color = {
  const counts = d3.rollup(films, v => v.length, d => d.primaryGenre);
  const ordered = [...counts]
    .sort((a, b) => d3.descending(a[1], b[1]))
    .map(d => d[0]);
  return d3.scaleOrdinal(ordered, d3.schemeTableau10.concat(d3.schemeSet3));
}
```

## Filter controls

Each control is an Observable `viewof` cell — the value updates reactively
as the user interacts:

```js
viewof selectedGenre = Inputs.select(
  ["ALL", ...new Set(films.flatMap(d => d.genres))].sort(),
  { label: "Genre", value: "ALL" }
)

viewof yearStart = Inputs.range([1920, 2020], {
  label: "From year", step: 1, value: 1920
})

viewof yearEnd = Inputs.range([1920, 2020], {
  label: "To year", step: 1, value: 2020
})

viewof minVotes = Inputs.select(
  [0, 50000, 200000, 500000, 1000000],
  { label: "Min votes", value: 0, format: d => d.toLocaleString() }
)

viewof searchTerm = Inputs.text({
  label: "Search title / director / star",
  placeholder: "e.g. Nolan"
})

mutable selectedDirector = null
```

The `mutable` keyword lets the director-bar click handler reassign
`selectedDirector`, which propagates through the dataflow graph to every
chart that references it.

## The reactive filter

A single derived cell expresses the filter logic. Any control change
re-runs this cell, which re-runs every chart that depends on it:

```js
filtered = {
  const term = (searchTerm || "").trim().toLowerCase();
  const lo = Math.min(yearStart, yearEnd);
  const hi = Math.max(yearStart, yearEnd);
  return films.filter(d => {
    if (selectedGenre !== "ALL" && !d.genres.includes(selectedGenre)) return false;
    if (d.votes < minVotes) return false;
    if (d.year < lo || d.year > hi) return false;
    if (term) {
      const hay = (d.title + " " + d.director + " " + d.stars.join(" ")).toLowerCase();
      if (!hay.includes(term)) return false;
    }
    return true;
  });
}
```

A summary line under the controls is one line of code that surfaces the
current filter's count, mean rating, and combined gross:

```js
md`**${filtered.length}** films · avg rating **${d3.mean(filtered, d => d.rating)?.toFixed(2) ?? "—"}** · combined gross **$${d3.format(".2s")(d3.sum(filtered.filter(d=>d.gross), d => d.gross)).replace("G","B")}**`
```

## The four charts

Each chart is its own cell. None of them know about filtering — they each
read `filtered` and render the result. Re-rendering on filter change is
handled by Observable's dataflow.

### Q1 — Scatterplot: rating vs. box-office gross

```js
scatter = {
  const width = 760, height = 420;
  const margin = { top: 20, right: 20, bottom: 50, left: 60 };
  const data = filtered.filter(d => d.gross);

  const x = d3.scaleLog()
    .domain(d3.extent(films.filter(d => d.gross), d => d.gross))
    .range([margin.left, width - margin.right]).nice();
  const y = d3.scaleLinear()
    .domain([7.5, d3.max(films, d => d.rating)])
    .range([height - margin.bottom, margin.top]).nice();
  const r = d3.scaleSqrt()
    .domain([0, d3.max(films, d => d.votes)])
    .range([2, 12]);

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("max-width", "100%").style("height", "auto");

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).ticks(6, "$~s"));
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y));

  svg.append("g").selectAll("circle")
    .data(data).join("circle")
    .attr("cx", d => x(d.gross))
    .attr("cy", d => y(d.rating))
    .attr("r",  d => r(d.votes))
    .attr("fill", d => color(d.primaryGenre))
    .attr("fill-opacity", d =>
      selectedDirector
        ? (d.director === selectedDirector ? 0.95 : 0.08)
        : 0.7)
    .attr("stroke", d => d.director === selectedDirector ? "black" : "none")
    .attr("stroke-width", 1.2)
    .append("title")
    .text(d => `${d.title} (${d.year})
${d.primaryGenre} · ${d.director}
IMDB ${d.rating}${d.meta ? " · Meta " + d.meta : ""}
$${d3.format(",")(d.gross)} · ${d3.format(",")(d.votes)} votes`);

  return svg.node();
}
```

The directly-derived `fill-opacity` expression handles the highlight
cascade — when a director is selected, only that director's films stay
opaque while everything else dims.

### Q2 — Histogram: rating distribution

```js
histogram = {
  const width = 460, height = 320;
  const margin = { top: 20, right: 16, bottom: 40, left: 50 };

  const x = d3.scaleLinear().domain([7.5, 9.4])
    .range([margin.left, width - margin.right]);
  const bins = d3.bin().domain(x.domain())
    .thresholds(d3.range(7.5, 9.4, 0.1))
    (filtered.map(d => d.rating));
  const y = d3.scaleLinear().domain([0, d3.max(bins, b => b.length) || 1])
    .range([height - margin.bottom, margin.top]).nice();

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("max-width","100%").style("height","auto");

  svg.append("g").selectAll("rect")
    .data(bins).join("rect")
    .attr("x", d => x(d.x0) + 1)
    .attr("y", d => y(d.length))
    .attr("width", d => Math.max(0, x(d.x1) - x(d.x0) - 2))
    .attr("height", d => y(0) - y(d.length))
    .attr("fill", "#7c3aed")
    .append("title")
    .text(d => `${d.x0.toFixed(1)}–${d.x1.toFixed(1)}: ${d.length} films`);

  // axes omitted for brevity
  return svg.node();
}
```

### Q3 — Decade chart: line + bars

A dual-encoding chart that shows the average IMDB rating per decade as a
yellow line and the count of top-1000 films per decade as translucent
purple bars sharing the same X-axis:

```js
decadeChart = {
  const width = 760, height = 360;
  const margin = { top: 20, right: 60, bottom: 50, left: 50 };

  const decades = d3.range(1920, 2030, 10);
  const grouped = d3.rollup(filtered,
    v => ({ avg: d3.mean(v, d => d.rating), count: v.length }),
    d => d.decade);
  const series = decades.map(d => ({
    decade: d,
    ...(grouped.get(d) || { avg: null, count: 0 })
  }));

  const x = d3.scaleLinear().domain([1920, 2020])
    .range([margin.left, width - margin.right]);
  const yAvg = d3.scaleLinear().domain([7.5, 9.0])
    .range([height - margin.bottom, margin.top]).nice();

  const allDecadeMax = d3.max(d3.rollup(films, v => v.length, d => d.decade).values());
  const yCnt = d3.scaleLinear().domain([0, allDecadeMax])
    .range([height - margin.bottom, margin.top + (height - margin.top - margin.bottom) * 0.45]);

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("max-width","100%").style("height","auto");

  // bars: count per decade
  svg.append("g").selectAll("rect")
    .data(series).join("rect")
    .attr("x", d => x(d.decade) - 18)
    .attr("y", d => yCnt(d.count))
    .attr("width", 36)
    .attr("height", d => yCnt(0) - yCnt(d.count))
    .attr("fill", "#7c3aed").attr("opacity", 0.45);

  // line: avg rating per decade
  const valid = series.filter(d => d.avg != null);
  const line = d3.line()
    .x(d => x(d.decade))
    .y(d => yAvg(d.avg))
    .curve(d3.curveMonotoneX);
  svg.append("path").datum(valid)
    .attr("fill","none").attr("stroke","#f5c518").attr("stroke-width", 2.5)
    .attr("d", line);

  // axes omitted for brevity
  return svg.node();
}
```

### Q4 — Director bar chart

The bar chart's click handler is the only place in the entire notebook
that mutates state, and it does so by reassigning the `mutable` cell
declared in the controls section. Every other chart reacts:

```js
directorBar = {
  const width = 460, height = 360;
  const margin = { top: 10, right: 16, bottom: 34, left: 160 };

  const top = d3.rollups(filtered,
      v => ({ avg: d3.mean(v, d => d.rating), count: v.length }),
      d => d.director)
    .filter(([k, v]) => k && v.count >= 3)
    .sort((a, b) => d3.descending(a[1].avg, b[1].avg))
    .slice(0, 12);

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("max-width","100%").style("height","auto");

  if (!top.length) {
    svg.append("text").attr("x", width/2).attr("y", height/2)
       .attr("text-anchor","middle").attr("fill","currentColor")
       .text("No director has ≥3 films in current filter.");
    return svg.node();
  }

  const y = d3.scaleBand().domain(top.map(d => d[0]))
    .range([margin.top, height - margin.bottom]).padding(0.18);
  const x = d3.scaleLinear().domain([7.5, d3.max(top, d => d[1].avg)])
    .range([margin.left, width - margin.right]).nice();

  svg.append("g").selectAll("rect")
    .data(top).join("rect")
    .attr("x", margin.left).attr("y", d => y(d[0]))
    .attr("width", d => x(d[1].avg) - margin.left)
    .attr("height", y.bandwidth())
    .attr("fill", d => d[0] === selectedDirector ? "#f5c518" : "#7c3aed")
    .style("cursor", "pointer")
    .on("click", (event, d) => {
      mutable selectedDirector = (selectedDirector === d[0]) ? null : d[0];
    });

  // axes and bar value labels omitted for brevity
  return svg.node();
}
```

## Detail-on-demand: the film table

`Inputs.table` provides a sortable, filterable table in a single
declaration. It mirrors `filtered` by default, and narrows to a single
director when one is selected:

```js
tableData = selectedDirector
  ? filtered.filter(d => d.director === selectedDirector)
  : filtered

Inputs.table(tableData, {
  columns: ["title", "year", "primaryGenre", "director", "rating", "meta", "runtime", "votes", "gross"],
  header: {
    title: "Title", year: "Year", primaryGenre: "Genre", director: "Director",
    rating: "IMDB", meta: "Meta", runtime: "Runtime", votes: "Votes", gross: "Gross ($)"
  },
  format: {
    runtime: d => d == null ? "—" : d + " min",
    votes:   d => d3.format(",")(d),
    gross:   d => d == null ? "—" : "$" + d3.format(",")(d),
    meta:    d => d == null ? "—" : d
  },
  sort: "rating",
  reverse: true,
  rows: 18
})
```

## Layout

Charts compose into the final two-column dashboard via `htl.html`:

```js
htl.html`<div style="display:flex; gap:16px; flex-wrap:wrap;">
  <div style="flex:2 1 600px;">${scatter}</div>
  <div style="flex:1 1 380px;">${histogram}</div>
</div>`

htl.html`<div style="display:flex; gap:16px; flex-wrap:wrap;">
  <div style="flex:2 1 600px;">${decadeChart}</div>
  <div style="flex:1 1 380px;">${directorBar}</div>
</div>`
```

## Findings

The same five findings surfaced in both implementations:

1. **Rating ≠ box office.** The scatter shows only a weak positive
   correlation. The highest-grossing films cluster around 8.0–8.4, not at
   the very top of the rating scale.
2. **Most "great" films sit in the 7.6–8.0 band.** The histogram is
   right-skewed even in this curated top-1000 set; ratings of 9.0+ are
   vanishingly rare.
3. **Production volume soared after 1990.** Decade bars climb steeply from
   the 1990s onward, while the average rating line stays flat at roughly
   8.0 across a hundred years.
4. **A small set of directors dominates the rankings.** Christopher Nolan,
   Stanley Kubrick, Francis Ford Coppola, Akira Kurosawa, and Hayao
   Miyazaki appear at the top of the average-rating bar chart with at
   least three films each.
5. **Animation is a 1990s-and-onward phenomenon.** Filtering to "Animation"
   collapses the bar chart and decade chart to the most recent three
   decades, reflecting the mainstream rise of Pixar, DreamWorks, and
   Studio Ghibli.

## Comparison: Observable vs. the static GitHub Pages submission

| Concern | Observable | GitHub Pages submission |
| --- | --- | --- |
| Reactivity | Built in via the dataflow graph | Hand-rolled `applyFilters()` / `renderAll()` |
| UI controls | `Inputs.select` / `Inputs.range` / `Inputs.text` give polished defaults | Native HTML controls styled with CSS |
| Hosting | `observablehq.com/@user/notebook` | `<user>.github.io/<repo>/` |
| Theming control | Limited to Observable's notebook chrome | Full custom theme (the lavender palette in the final submission) |
| Iteration speed during development | Very fast — each cell is independently re-runnable | Slower — full page reload per change |
| Final deliverable form | Notebook + static export | Two HTML files plus the CSV |

The Observable version proved the architecture quickly and is preserved
here as a reference implementation. The submitted version is the static
GitHub Pages site at
[veradureke.github.io/imdb-d3-final](https://veradureke.github.io/imdb-d3-final/).
