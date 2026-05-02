# IMDB Top 1000 Interactive Dashboard

**N328 Visualizing Information · Vera Dureke · Spring 2026**

A coordinated multi-view dashboard for exploring the top 1,000 films on
IMDB from 1920 to 2020. Each chart shares a single filter state, so
selecting a genre, brushing a year range, or clicking a director's bar
re-filters every other panel and the film list at the bottom of the page.

---

## Setup

Attach the dataset using the paperclip icon in the right sidebar:
[`imdb_top_1000.csv`](https://www.kaggle.com/datasets/harshitshankhdhar/imdb-dataset-of-top-1000-movies-and-tv-shows).

The first cell loads the CSV, parses the typed fields, and produces a
cleaned array of film records. Three small data quirks are handled at
parse time: one row contains `"PG"` in the `Released_Year` field and is
dropped, runtime strings like `"142 min"` are stripped down to integer
minutes, and gross strings like `"28,341,469"` lose their commas before
being parsed.

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

A categorical color scale is derived once from the data, ordered so the
most common primary genres get the most visually distinct hues from the
Tableau-10 palette, with Set3 filling in any remaining categories.

```js
color = {
  const counts = d3.rollup(films, v => v.length, d => d.primaryGenre);
  const ordered = [...counts]
    .sort((a, b) => d3.descending(a[1], b[1]))
    .map(d => d[0]);
  return d3.scaleOrdinal(ordered, d3.schemeTableau10.concat(d3.schemeSet3));
}
```

---

## Filter controls

Five controls drive every chart in the notebook. Each one is a `viewof`
cell, which means the value updates reactively as the user interacts.

```js
viewof selectedGenre = Inputs.select(
  ["ALL", ...new Set(films.flatMap(d => d.genres))].sort(),
  { label: "Genre", value: "ALL" }
)
```

```js
viewof yearStart = Inputs.range([1920, 2020], {
  label: "From year", step: 1, value: 1920
})
```

```js
viewof yearEnd = Inputs.range([1920, 2020], {
  label: "To year", step: 1, value: 2020
})
```

```js
viewof minVotes = Inputs.select(
  [0, 50000, 200000, 500000, 1000000],
  { label: "Min votes", value: 0, format: d => d.toLocaleString() }
)
```

```js
viewof searchTerm = Inputs.text({
  label: "Search title / director / star",
  placeholder: "e.g. Nolan"
})
```

The director bar chart can also push state back into the dataflow.
A `mutable` cell holds the currently-selected director and gets
reassigned by the bar chart's click handler:

```js
mutable selectedDirector = null
```

---

## The reactive filter

Every chart reads from a single derived cell called `filtered`. Whenever
any control changes, the runtime re-evaluates this cell, and the charts
that depend on it re-render automatically.

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

A summary line just below the controls surfaces the current selection's
size, mean rating, and combined gross:

```js
md`**${filtered.length}** films · avg rating **${d3.mean(filtered, d => d.rating)?.toFixed(2) ?? "—"}** · combined gross **$${d3.format(".2s")(d3.sum(filtered.filter(d=>d.gross), d => d.gross)).replace("G","B")}**`
```

---

## Q1: rating versus box-office gross

Each film is a dot in a scatterplot. Color encodes the primary genre,
size encodes the number of votes on a square-root scale, and the X-axis
runs on a log scale so revenue across five orders of magnitude all
fits in one chart. When a director is selected, that director's films
stay opaque while everything else dims to 8% opacity, producing a
highlight cascade across the rest of the dashboard.

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

---

## Q2: the rating distribution

A histogram of IMDB ratings in 0.1-wide bins from 7.5 to 9.4. The
distribution is right-skewed: most films cluster between 7.6 and 8.0,
and ratings above 9.0 are extremely rare even in this curated top-1000
set.

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

  return svg.node();
}
```

The Q1 scatter and Q2 histogram are laid out side by side using
`htl.html` and CSS flexbox.

```js
htl.html`<div style="display:flex; gap:16px; flex-wrap:wrap;">
  <div style="flex:2 1 600px;">${scatter}</div>
  <div style="flex:1 1 380px;">${histogram}</div>
</div>`
```

---

## Q3: trends by decade

Two encodings share the same X-axis. The yellow line traces the average
IMDB rating per decade, which stays remarkably flat near 8.0 across a
century. Behind it, translucent purple bars count the number of top-1000
films released each decade, revealing a steep climb from the 1990s
onward.

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

  svg.append("g").selectAll("rect")
    .data(series).join("rect")
    .attr("x", d => x(d.decade) - 18)
    .attr("y", d => yCnt(d.count))
    .attr("width", 36)
    .attr("height", d => yCnt(0) - yCnt(d.count))
    .attr("fill", "#7c3aed").attr("opacity", 0.45);

  const valid = series.filter(d => d.avg != null);
  const line = d3.line()
    .x(d => x(d.decade))
    .y(d => yAvg(d.avg))
    .curve(d3.curveMonotoneX);
  svg.append("path").datum(valid)
    .attr("fill","none").attr("stroke","#f5c518").attr("stroke-width", 2.5)
    .attr("d", line);

  return svg.node();
}
```

---

## Q4: top directors

Directors are ranked by the mean IMDB rating across their films in the
dataset, restricted to those with at least three films so the ranking
isn't dominated by a single high-rated outlier. Clicking a bar
reassigns `selectedDirector`, which propagates back to the scatterplot
to drive the highlight cascade and to the film table to narrow the
visible rows.

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

  return svg.node();
}
```

The Q3 decade chart and Q4 director bar share a row using the same
flexbox composition.

```js
htl.html`<div style="display:flex; gap:16px; flex-wrap:wrap;">
  <div style="flex:2 1 600px;">${decadeChart}</div>
  <div style="flex:1 1 380px;">${directorBar}</div>
</div>`
```

---

## Detail-on-demand: the film table

A sortable, paginated table that mirrors the current filtered selection
and narrows further to a single director when one is chosen. The whole
control is a single declaration thanks to `Inputs.table`, with column
formatters for runtime, votes, and gross.

```js
tableData = selectedDirector
  ? filtered.filter(d => d.director === selectedDirector)
  : filtered
```

```js
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

---

## Findings

The dashboard surfaces five patterns in the data:

1. **Rating and box office only weakly correlate.** The scatter shows a
   gentle positive trend at most. Highest-grossing films cluster around
   8.0–8.4, not at the very top of the rating scale.
2. **Most "great" films sit in the 7.6–8.0 band.** The histogram is
   right-skewed even for a curated top-1000 set; ratings of 9.0+ are
   vanishingly rare.
3. **Production volume soared after 1990.** The decade bars climb
   steeply from the 1990s onward, while the average rating stays close
   to 8.0 across the entire century.
4. **A small set of directors dominates the rankings.** Christopher
   Nolan, Stanley Kubrick, Francis Ford Coppola, Akira Kurosawa, and
   Hayao Miyazaki appear at the top of the average-rating bar chart
   with at least three films each.
5. **Animation is a 1990s-and-onward phenomenon.** Filtering to
   "Animation" collapses the bar chart and decade chart to the most
   recent three decades, reflecting the mainstream rise of Pixar,
   DreamWorks, and Studio Ghibli.
