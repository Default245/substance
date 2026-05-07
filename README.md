# Substance

A multi-lens reading of the periodic table. Every element rendered through five dimensions at once: rarity, price, toxicity, supply concentration, and whether it's in the device you're holding right now.

**Live:** https://default245.github.io/substance/

## What it does

- **Heatmap** — color cells by Price, Crustal Rarity, Toxicity, or Supply Concentration. Log scales where they belong.
- **The Rarity Plane** — log–log scatter of crustal abundance × price. Reveals why rhodium costs more than gold.
- **In Your Phone** — highlights the ~50 elements inside a typical smartphone. Conflict-mineral flag (3TG + cobalt) where applicable.
- **Supply Concentration** — colors cells by single-country share of mine production.

Click any element in any view to see all five dimensions at once.

## Data

Single static HTML file, no build step, no dependencies beyond Google Fonts.

- Prices: indicative, late 2024, USD per kilogram
- Abundances: continental crust, parts per million
- Supply concentration: top single-country share of global mine production, USGS / industry reports
- Conflict mineral flag: 3TG (Tin · Tantalum · Tungsten · Gold) plus Cobalt
- Toxicity: qualitative 0–5 scale

## License

MIT.
