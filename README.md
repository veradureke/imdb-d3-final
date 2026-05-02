# IMDB Top 1000 Interactive Dashboard

**Final project · N328 Visualizing Information · Spring 2026**
**Author:** Vera Dureke

An interactive, multi-view dashboard built with [D3.js v7](https://d3js.org)
that lets a viewer explore the
[IMDB Top 1000 Movies dataset](https://www.kaggle.com/datasets/harshitshankhdhar/imdb-dataset-of-top-1000-movies-and-tv-shows)
— 1,000 films from 1920 to 2020 with attributes for release year, genre,
certificate, runtime, IMDB rating, Meta score, director, top-billed cast,
vote count, and worldwide gross.

## Live links

- **Interactive visualization:** https://veradureke.github.io/imdb-d3-final/
- **Documentation page** (design process, rationale, findings, demo video):
  https://veradureke.github.io/imdb-d3-final/about.html
- **Demo video** (2–4 min screen-capture walkthrough): embedded on the documentation page

## Driving questions

The dashboard is organized around four related questions a film fan might ask:

1. Do critically loved films also make the most money? (rating vs. gross)
2. What does the rating distribution look like, and where does any given film land?
3. How have ratings, gross, and the volume of "great" films changed by decade?
4. Which directors most consistently produce critically acclaimed films?

## Features

- **Four linked charts** — a scatterplot (rating vs. gross), a histogram
  (rating distribution), a line and bar combo (trends by decade), and a
  horizontal bar chart (top directors by average rating).
- **Coordinated views** — every chart shares one filter state. A genre
  dropdown, a search box, a minimum-votes filter, a year-range brush on the
  decade chart, a click-to-filter histogram, and a click-to-highlight
  director bar all update the rest of the dashboard simultaneously.
- **Detail-on-demand film table** — sortable, filterable, with formatted
  values for rating, runtime, votes, and box-office gross.
- **Hover tooltips** on every dot, bar, and bin display the underlying film
  or aggregate values.

## Visual encodings

| Variable | Encoding | Why |
| --- | --- | --- |
| IMDB rating | Y-position (scatter, line); X-position (histogram, bar) | Position is the most accurate visual channel — rating is the primary quantitative variable. |
| Box office gross | X-position on a log scale | Gross spans `$10K`–`$2.8B`; a linear scale would crush 90% of films into one corner. |
| Number of votes | Dot area (sqrt scale) | Encodes a third dimension without adding a chart. |
| Primary genre | Categorical color (Tableau-10) | Genres are unordered; this palette is colorblind-friendly. |
| Decade | X-axis on the line chart | Naturally ordinal — position is the right channel. |

A more detailed design rationale, including initial sketches and the
iteration history, lives on the
[documentation page](https://veradureke.github.io/imdb-d3-final/about.html).

## Repository contents

| File | Purpose |
| --- | --- |
| `index.html` | Interactive D3 dashboard |
| `about.html` | Documentation page |
| `imdb_top_1000.csv` | Source dataset (1,000 films, 16 columns) |
| `20260501_201615.jpg` | Initial pencil sketch (single-chart design) |
| `20260501_201627.jpg` | Refined pencil sketch (coordinated-dashboard layout) |
| `video2378517474.mp4` | 2–4 minute screen-capture demo with narration |
| `favicon.svg` | Site favicon |
| `og-image.svg` | Open Graph preview image |
| `404.html` | Custom not-found page |
| `README.md` | This file |

## Tech stack

- **D3.js v7** — all rendering, scales, axes, and the year-range brush
- **Plain HTML, CSS, JavaScript** — no build step, no framework
- **GitHub Pages** — static hosting

## Data cleaning

The CSV is cleaned client-side in `index.html` at parse time:

- One row had `"PG"` in the `Released_Year` field, dropped from the dataset.
- `Runtime` strings like `"142 min"` are parsed to integer minutes.
- `Gross` strings like `"28,341,469"` (with commas) are parsed to integers.
- Films missing `Gross` are kept in every chart except the scatterplot
  (which requires both axes).

## Acknowledgements

Dataset: [IMDB Top 1000 Movies](https://www.kaggle.com/datasets/harshitshankhdhar/imdb-dataset-of-top-1000-movies-and-tv-shows)
by Harshit Shankhdhar (Kaggle, CC0).
Built with [D3.js](https://d3js.org) by Mike Bostock.
