# Protein Intake Patterns Across Indian States

**Name:** Rahul Bharadwaj Ramesh
**Student ID:** 801458858

---

## Theme

This dashboard explores **dietary protein consumption patterns across Indian states**, examining how factors like urbanization, vegetarianism, and food source types influence protein intake. The visualizations investigate regional disparities in nutrition and the relationship between lifestyle factors and dietary choices.

---

## About the Dataset

This dataset is sourced from the **Indian Council of Medical Research (ICMR)** ([www.icmr.gov.in](https://www.icmr.gov.in)). It contains nutritional and demographic data for various Indian states, covering metrics such as average daily protein intake, urban/rural population distribution, vegetarian population percentages, and protein source information. Key fields include `State`, `Urban (%)`, `Rural (%)`, `Average Daily Protein Intake (grams)`, `% Vegetarian Population`, `Protein_Density (g/100g)`, `Dietary_Share (%)`, `Source_Type`, and `Protein_Quality`. Data cleaning involved filtering out records with missing values and excluding states with 100% urban population to ensure meaningful comparisons.

---

## Visualizations

```js
import * as Plot from "@observablehq/plot";
import * as d3 from "d3";

const data = await FileAttachment("data/dataset.csv").csv({typed: true});

// Filter
const filtered = data.filter(d =>
  d["Urban (%)"] != null &&
  d["Average Daily Protein Intake (grams)"] != null &&
  d["Urban (%)"] !== 100
);
```

### Interactive Controls

```js
// State Selector - Interactive Dropdown
const stateList = ["All States", ...filtered.map(d => d.State).sort()];
const selectedState = view(Inputs.select(stateList, {
  label: "Select State",
  value: "All States"
}));
```

```js
// Protein Range Slider - Interactive Filter
const proteinExtent = d3.extent(filtered, d => d["Average Daily Protein Intake (grams)"]);
const proteinRange = view(Inputs.range(
  [Math.floor(proteinExtent[0]), Math.ceil(proteinExtent[1])],
  {
    label: "Protein Intake Range (g)",
    step: 1,
    value: Math.floor(proteinExtent[0])
  }
));
```

```js
// Apply filters based on interactive controls
const interactiveFiltered = filtered.filter(d => {
  const matchesState = selectedState === "All States" || d.State === selectedState;
  const matchesProtein = d["Average Daily Protein Intake (grams)"] >= proteinRange;
  return matchesState && matchesProtein;
});
```

### Chart 1: Urbanization vs. Protein Intake

```js

// Calculate mean for the filtered data
const meanProtein = d3.mean(interactiveFiltered, d => d["Average Daily Protein Intake (grams)"]);

display(Plot.plot({
  // Style: Dark mode optimized
  style: "color: white; background-color: transparent; font-family: sans-serif;",

  title: "Urbanization vs. Protein Intake",
  subtitle: `Showing ${interactiveFiltered.length} state(s) | Filter: ${selectedState}, Min Protein: ${proteinRange}g`,
  width: 1000,
  height: 500,
  grid: true,
  inset: 10,
  
  x: { label: "Urban Population (%)" },
  y: { label: "Daily Protein Intake (g)" },

  marks: [
    // 1. The Reference Line (Orange Dashed)
    Plot.ruleY([meanProtein], {
      stroke: "orange",
      strokeDasharray: "4",
      strokeWidth: 2
    }),
    
// Add text to the reference line
    Plot.text(interactiveFiltered, Plot.groupZ({y: "mean"}, {
      y: "Average Daily Protein Intake (grams)",
      text: d => `Avg: ${meanProtein ? meanProtein.toFixed(1) : 0}g`,
      frameAnchor: "right",
      dy: -10,
      fill: "orange"
    })),

    // 3. The Dots (With State Tooltip)
    Plot.dot(interactiveFiltered, {
      x: "Urban (%)",
      y: "Average Daily Protein Intake (grams)",
      r: 6, // Constant size is better than redundant size
      fill: "teal",
      stroke: "white",
      strokeWidth: 1.5,
      opacity: 0.8,
      
      // --- HERE IS THE TOOLTIP LOGIC ---
      channels: {
        State: "State", // This adds "State" to the tooltip
        Rural: "Rural (%)"
      },
      tip: {
        format: {
          x: (d) => `${d}%`, // Format Urban %
          y: (d) => `${d}g`, // Format Protein
          Rural: (d) => `${d}%` // Format Rural %
        }
      }
    }),

    // 4. Label the Top Performer
    Plot.text(interactiveFiltered, Plot.selectMaxY({
      x: "Urban (%)",
      y: "Average Daily Protein Intake (grams)",
      text: "State",
      dy: -15,
      fill: "white",
      fontWeight: "bold"
    }))
  ]
}));
```

### Chart 2: Dietary Comparison

```js
// 1. Sort Order: High -> Low Protein
const sortedStates = interactiveFiltered
  .slice()
  .sort((a, b) => b["Average Daily Protein Intake (grams)"] - a["Average Daily Protein Intake (grams)"])
  .map(d => d.State);

// 2. Prepare Data (Long Format)
const longData = interactiveFiltered.flatMap(d => [
  {State: d["State"], Metric: "Protein Intake (g)", Value: d["Average Daily Protein Intake (grams)"]},
  {State: d["State"], Metric: "% Vegetarian Pop.", Value: d["% Vegetarian Population"]}
]);

display(Plot.plot({
  title: "Dietary Comparison (Sorted by Protein)",
  subtitle: "Top: Vegetarian Population (%) | Bottom: Daily Protein (g)",
  width: 1000,
  height: 600,
  grid: true,
  marginLeft: 60,
  // INCREASED THIS to fix cut-off text on the right
  marginRight: 120, 
  marginBottom: 100,
  
  style: "color: white; background-color: transparent;",

  x: { 
    axis: "bottom", 
    tickRotate: -45, 
    domain: sortedStates, 
    label: null
  },
  
  y: { label: null }, 

  fy: { 
    label: null,
    domain: ["% Vegetarian Pop.", "Protein Intake (g)"] 
  },

  color: {
    domain: ["% Vegetarian Pop.", "Protein Intake (g)"],
    range: ["lightgreen", "teal"], 
    legend: true
  },

  marks: [
    Plot.barY(longData, {
      x: "State", 
      y: "Value", 
      fy: "Metric", 
      fill: "Metric",
      
      // --- CUSTOM TOOLTIP SETTINGS ---
      channels: {
        State: "State", // Explicitly adds State to the tooltip
        // This adds a custom line with the correct unit
        Reading: d => `${d.Value} ${d.Metric.includes("Protein") ? "g" : "%"}`
      },
      tip: true
      // -------------------------------
    }),
    Plot.ruleY([0])
  ]
}));


```

### Chart 3: Protein Efficiency Bubble Chart

```js
// 1. Load the new dataset (Assuming it is in data/ folder like the previous one)
const proteinData = await FileAttachment("data/dataset2.csv").csv({typed: true});

// 2. Filter the data to remove missing values AND apply interactive filter
const filteredProtein = proteinData.filter(d =>
  d["Protein_Density (g/100g)"] != null &&
  d["Dietary_Share (%)"] != null &&
  (selectedState === "All States" || d["State"] === selectedState)
);

display(Plot.plot({
  // Style for Dark Mode
  style: "color: white; background-color: transparent;",

  title: "Protein Efficiency: Density vs. Dietary Share",
  subtitle: `Bubble size = Dietary contribution | Filter: ${selectedState}`,
  width: 1000,
  height: 500,
  grid: true,
  inset: 20,
  marginLeft: 50,
  
  x: { label: "Protein Density (g/100g) →", domain: [0, 40] },
  y: { label: "Dietary Share (%) ↑", domain: [0, 70] },
  
  // Note: Mapping 'r' to the same variable as 'y' emphasizes the vertical position
  r: { range: [3, 25], label: "Dietary Share %" },
  
  color: { 
    legend: true, 
    domain: ["Animal", "Plant"], 
    range: ["tomato", "seagreen"] 
  },

  marks: [
    Plot.dot(filteredProtein, {
      x: "Protein_Density (g/100g)",
      y: "Dietary_Share (%)",
      r: "Dietary_Share (%)", 
      fill: "Source_Type",
      fillOpacity: 0.6,
      stroke: "white", 
      strokeWidth: 1.5,
      
      // Define the extra data for the tooltip
      channels: {
        Food: "Protein_Source",State: "State",
        "Protein Quality": "Protein_Quality"
      },
      
      // Enable Tooltip with formatting
      tip: {
        format: {
          x: (d) => `${d} g/100g`,
          y: (d) => `${d}%`,
          r: (d) => `${d}%`
        }
      }
    })
  ]
}));
```

---

## Key Insights & Design Decisions

- **Interactive Filtering:** The dashboard includes a state selector dropdown and a protein intake range slider, allowing users to dynamically filter all linked visualizations. This enables focused analysis on specific states or protein intake thresholds without cluttering the charts.

- **Visual Encoding Choices:** The scatter plot uses position encoding for the primary relationship (urbanization vs. protein), while the bubble chart employs size and color to represent dietary share and source type (animal vs. plant). This multi-dimensional approach reveals patterns that single-variable charts would miss.

- **Sorted Bar Charts:** The dietary comparison chart sorts states by protein intake (high to low), making it immediately apparent which states lead in protein consumption and how this correlates with vegetarian population percentages.

- **Key Finding - Urbanization Paradox:** Contrary to expectations, higher urbanization does not consistently correlate with higher protein intake. States like Haryana show high protein consumption despite moderate urbanization, suggesting cultural and dietary traditions play a stronger role than urban development.

- **Plant vs. Animal Protein Tradeoff:** The bubble chart reveals that plant-based proteins (cereals, pulses) dominate dietary share but have lower protein density, while animal sources offer higher density but smaller dietary contribution. This highlights India's predominantly plant-based protein landscape.
