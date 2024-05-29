<script>
import data from "../data/data.json";
import { timeParse } from "d3-time-format";
import { scaleLinear,scaleSqrt } from "d3-scale";
import { extent, range } from "d3-array";
import { format } from "d3-format";
import { interpolateHclLong } from "d3-interpolate";
import { timeDay,timeYear } from "d3-time";
import { timeFormat } from "d3-time-format";
import { fade } from "svelte/transition";

const parseDate = timeParse('%Y-%m-%d');

let metricAccessor = (d) => d['killed'];
let timeAccessor = (d) => d['Date'];

const progressInYearAccessor = (d) => {
  const date = parseDate(timeAccessor(d));
  return timeDay.count(timeYear.floor(date), date); //0 to 364
};

let width = 100;
$: height = width

$: metricDomain = extent(data,d=>d.killed) 
$: timeDomain = extent(data, d=>parseDate(d.Date))

$: colorScale = scaleLinear()
  .domain([0, metricDomain[1] / 3, (metricDomain[1] / 3) * 2, metricDomain[1]])
  .range(['#F7D6B2', '#F2845C', '#D93E30', '#400101'])
  .interpolate(interpolateHclLong);

const maxLength = 9; // for legend
$: scaleScale = scaleSqrt().domain(metricDomain).range([0, maxLength]);
$: colorScaleTicks = colorScale.ticks(6).map((d) => { // to draw legend
  const scale = scaleScale(d);
  const tickHeight = (height / 100) * scale;
  return {
    tickHeight,
    value: d,
    percent: (d * 100) / metricDomain[1],
    color: colorScale(d)
  };
});

$: radiusScale = scaleLinear().domain(timeDomain).range([15, 40]); //inner + outer radius of shape
$: yearRadiusScale = scaleLinear()
  .domain(timeDomain)
  .range([height * 0.21, height * 0.41]);
$: angleScale = scaleLinear().domain([0, 365]).range([0, 360]); //map days to angle, but idk if it should be 365 or 364 because of 2/29

//function to return x/y positions given a distance and angle
const getPositionFromDistanceAndAngle = (distance, angle) => {
  //Archimedean spiral
  //r= a + bθ, r is distance from origin (radius), θ is the angle in radians, a and b are constants that control the starting radius and the distance between the turns of the spiral
  const x = distance * Math.cos((angle * Math.PI) / 180); //if add a constant b can make it bigger, or flatter or more vertical if it differs
  const y = distance * Math.sin((angle * Math.PI) / 180);
  return { x, y };
};

//function to take the x/y from the above fn and translate it
const getTransformFromDistanceAndAngle = (distance, angle) => { //distance from 15-40, angle -90 to 270
  const { x, y } = getPositionFromDistanceAndAngle(distance, angle);
  return `translate(${x}, ${y})`;
};

//function to generate the line path based on the value
const getPathForValue = (value) => {
  //value ranges between 0 and 22
  const height = scaleScale(value);
  return ['M', 0, -height, 'L', 0, 0].join(' ');
  //['M', 0, 0, 'L', 0, height].join(' '); this option sets the bottom to same starting pt
  //['M', 0, -height/2, 'L', 0, height/2].join(' ') original
};


const monthNames = range(0,12).map(i => timeFormat('%b')(new Date(2000, i, 1)));
$: months = monthNames.map((month, i) => {
  const angle = (360 / 12) * i - 360 / 4;
  const { x, y } = getPositionFromDistanceAndAngle(width * (i ? 0.38 + i * 0.009 : 0.49), angle);
  return {
    i,
    name: month,
    angle,
    x: x + width / 2,
    y: y - height / 2
  };
});


$: years = timeYear.range(timeYear.floor(timeDomain[0]), timeDomain[1])
    .map((year) => {
      const r = yearRadiusScale(year);
      const { x, y } = getPositionFromDistanceAndAngle(r, -360 / 4);
      return {
        name: year.getFullYear(),
        x: x + width / 2,
        y: y - height / 2
      };
    });
  
</script>


<div class="wrapper">
<div bind:clientWidth={width}>
  <svg width="100%" viewBox="-50 -50 100 100">
      {#each data as d,i }
        <path
          class="transition"
          in:fade={{delay: i * 3}}
          d={getPathForValue(metricAccessor(d))}
          r={scaleScale(metricAccessor(d))}
          stroke={colorScale(metricAccessor(d))}
          stroke-width={0.55}
          stroke-linecap="round"
          transform={getTransformFromDistanceAndAngle(
            radiusScale(parseDate(timeAccessor(d))),
            angleScale(progressInYearAccessor(d)) - 360 / 4
          ) + `rotate(${angleScale(progressInYearAccessor(d))})`}
        >
        <!-- if I take out the 360/4 above the lines become flat -->
        <!-- if I remove the rotate the each line is vertical -->
        </path>
      {/each}
  </svg>

  

  {#each months as { name, x, y }}
    <div
      class="label"
      style="transform: translate({x}px, {y}px)"
    >
      {name}
    </div>
  {/each}
  {#each years as { name, x, y }}
    <div
      class="label"
      style="transform: translate({x}px, {y}px)"
    >
      <div style="-webkit-text-stroke: 6px white">
        {name}
      </div>
      <div style="position: absolute">
        {name}
      </div>
    </div>
  {/each}
</div>

<div class="legend">
    <div class="legend__title">
      <slot name="legend-title" />
    </div>
    <div class="legend__ticks">
      {#each colorScaleTicks as { value, tickHeight, color } (value)}
        <div class="legend__tick">
          <div
            class="legend__bar-wrapper"
            style="height: {(height / 100) * maxLength}px"
          >
            <div
              class="legend__bar"
              style="height: {tickHeight}px; background-color: {color}"
            />
          </div>

          <div class="legend__value">
            {value}
          </div>
        </div>
      {/each}
    </div>
  </div>
</div>

<style>
.wrapper {
  position: relative;
  width: 100%;
  max-width: 80vh;
  padding: 2%;
  margin-bottom: 1em;
}
.label {
  position: absolute;
  width: 0;
  font-size: 0.8em;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  display: flex;
  align-items: center;
  justify-content: center;
}
.legend {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.legend__title {
  text-align: center;
  font-size: 0.8em;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  line-height: 1.2em;
  color: #999;
  max-width: 24em;
}
.legend__ticks {
  display: flex;
}
.legend__tick {
  display: flex;
  flex-direction: column;
  margin: 0 0.2em;
  transition: all 0.6s ease-out;
}
.legend__bar-wrapper {
  display: flex;
  justify-content: center;
  align-items: flex-end; 
  /* center */
}
.legend__bar {
  width: 0.3em;
  border-radius: 1em;
}
.legend__value {
  font-size: 0.8em;
  color: #999;
  text-align: center;
}
.transition {
  transition: all 0.6s ease-out;
}
</style>
