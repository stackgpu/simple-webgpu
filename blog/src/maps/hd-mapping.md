deck.gl is a “GPU-powered framework for visual exploratory data analysis of large datasets.” You can import deck.gl’s standalone bundle like so:

import deck from "npm:deck.gl";

const {DeckGL, AmbientLight, GeoJsonLayer, HexagonLayer, LightingEffect, PointLight} = deck;
Data: Department for Transport

const data = FileAttachment("../data/dft-road-collisions.csv").csv({array: true, typed: true}).then((data) => data.slice(1));
const topo = import.meta.resolve("npm:visionscarto-world-atlas/world/50m.json");
const world = fetch(topo).then((response) => response.json());

const countries = world.then((world) => topojson.feature(world, world.objects.countries));

2. The layout

Using nested divs, we position a large area for the chart, and a card floating on top that will receive the title, the color legend, and interactive controls:


<div class="card" style="margin: 0 -1rem;">

## Personal injury road collisions, 2022
### ${data.length.toLocaleString("en-US")} reported collisions on public roads

<figure style="max-width: none; position: relative;">
  <div id="container" style="border-radius: 8px; overflow: hidden; background: rgb(18, 35, 48); height: 800px; margin: 1rem 0; "></div>
  <div style="position: absolute; top: 1rem; right: 1rem; filter: drop-shadow(0 0 4px rgba(0,0,0,.5));">${colorLegend}</div>
  <figcaption>Data: <a href="https://www.data.gov.uk/dataset/cb7ae6f0-4be6-4935-9277-47e5ce24a11f/road-safety-data">Department for Transport</a></figcaption>
</figure>

</div>
The colors are represented as (red, green, blue) triplets, as expected by deck.gl. The legend is made using Observable Plot:


const colorRange = [
  [1, 152, 189],
  [73, 227, 206],
  [216, 254, 181],
  [254, 237, 177],
  [254, 173, 84],
  [209, 55, 78]
];

const colorLegend = Plot.plot({
  margin: 0,
  marginTop: 20,
  width: 180,
  height: 35,
  style: "color: white;",
  x: {padding: 0, axis: null},
  marks: [
    Plot.cellX(colorRange, {fill: ([r, g, b]) => `rgb(${r},${g},${b})`, inset: 0.5}),
    Plot.text(["Fewer"], {frameAnchor: "top-left", dy: -12}),
    Plot.text(["More"], {frameAnchor: "top-right", dy: -12})
  ]
});
3. The DeckGL instance
We create a DeckGL instance targetting the container defined in the layout. During development & preview, this code can run several times, so we take care to clean it up each time the code block runs:


const deckInstance = new DeckGL({
  container,
  initialViewState,
  getTooltip,
  effects,
  controller: true
});

// clean up if this code re-runs
invalidation.then(() => {
  deckInstance.finalize();
  container.innerHTML = "";
});
initialViewState describes the initial position of the camera:


const initialViewState = {
  longitude: -2,
  latitude: 53.5,
  zoom: 5.7,
  minZoom: 5,
  maxZoom: 15,
  pitch: 40.5,
  bearing: -5
};
getTooltip generates the contents displayed when you mouse over a hexagon:


function getTooltip({object}) {
  if (!object) return null;
  const [lng, lat] = object.position;
  const count = object.points.length;
  return `latitude: ${lat.toFixed(2)}
    longitude: ${lng.toFixed(2)}
    ${count} collisions`;
}
effects defines the lighting:


const effects = [
  new LightingEffect({
    ambientLight: new AmbientLight({color: [255, 255, 255], intensity: 1.0}),
    pointLight: new PointLight({color: [255, 255, 255], intensity: 0.8, position: [-0.144528, 49.739968, 80000]}),
    pointLight2: new PointLight({color: [255, 255, 255], intensity: 0.8, position: [-3.807751, 54.104682, 8000]})
  })
];
4. The props
Since some parameters are interactive, we use the setProps method to update the layers when their value changes:


deckInstance.setProps({
  layers: [
    new GeoJsonLayer({
      id: "base-map",
      data: countries,
      lineWidthMinPixels: 1,
      getLineColor: [60, 60, 60],
      getFillColor: [9, 16, 29]
    }),
    new HexagonLayer({
      id: "heatmap",
      data,
      coverage,
      radius,
      upperPercentile,
      colorRange,
      elevationScale: 50,
      elevationRange: [0, 5000 * t],
      extruded: true,
      getPosition: (d) => d,
      pickable: true,
      material: {
        ambient: 0.64,
        diffuse: 0.6,
        shininess: 32,
        specularColor: [51, 51, 51]
      }
    })
  ]
});

Lastly, the t variable controls the height of the extruded hexagons with a generator (that can be reset with a button input):


const t = (function* () {
  const duration = 1000;
  const start = performance.now();
  const end = start + duration;
  let now;
  while ((now = performance.now()) < end) yield d3.easeCubicInOut(Math.max(0, (now - start) / duration));
  yield 1;
})();
