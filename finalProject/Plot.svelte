<script>
  import { onMount } from "svelte";
  import { csv } from "d3-fetch";
  import { scaleLinear } from "d3-scale";
  import { line } from "d3-shape";
  import { extent } from "d3-array";
  import { fade } from "svelte/transition";

  let { scrollIndex = $bindable() } = $props();

  let data = $state();

  onMount(async () => {
    data = await csv(
      "https://raw.githubusercontent.com/nguyencomputing/cs333_finalProject/refs/heads/main/enrollmentData.csv"
    );
  });

  let spacing = 40;
  let column = 10;
  let rows = 10;

  // bump right margin so the plane doesn't clip
  let margin = { top: 20, left: 70, bottom: 40, right: 50 };
  let width = column * spacing + margin.left + margin.right;
  let height = rows * spacing + 80;

  let padding = 20;

  let dataset = $derived.by(() => {
    if (!data) return [];
    return data.map((d) => ({
      year: +d.year,
      students: parseFloat(String(d.students).replace(/,/g, "")),
    }));
  });

  const innerW = $derived(width - margin.left - margin.right);
  const innerH = $derived(height - margin.top - margin.bottom);

  const xScale = $derived.by(() => {
    if (!dataset.length) return null;
    return scaleLinear().domain(extent(dataset, (d) => d.year)).range([0, innerW]);
  });

  const yScale = $derived.by(() => {
    if (!dataset.length) return null;
    return scaleLinear()
      .domain([0, Math.max(...dataset.map((d) => d.students))])
      .nice()
      .range([innerH, 0]);
  });

  const lineGen = $derived.by(() => {
    if (!xScale || !yScale) return null;
    return line()
      .x((d) => xScale(d.year))
      .y((d) => yScale(d.students));
  });

  const ticks = $derived.by(() => {
    if (!dataset.length) return [];
    const startYear = dataset[0].year;
    const endYear = dataset[dataset.length - 1].year;
    const step = 15;
    let t = [startYear];
    for (let y = startYear + step; y < endYear; y += step) t.push(y);
    t.push(endYear);
    return t;
  });

  const plane = $derived.by(() => {
    if (!xScale || !yScale || dataset.length < 2) return null;
    const p1 = dataset[dataset.length - 2];
    const p2 = dataset[dataset.length - 1];
    const x1 = xScale(p1.year), y1 = yScale(p1.students);
    const x2 = xScale(p2.year), y2 = yScale(p2.students);

    return { x: x2, y: y2 };
  });
</script>

<svg width={width} height={height}>
  <g transform="translate({margin.left}, {margin.top})">
    {#if scrollIndex == 0 && dataset.length}
      <g transition:fade>
        <!-- X axis line -->
        <line x1={0} x2={innerW} y1={innerH} y2={innerH} stroke="#ccc" stroke-width="2" />

        <!-- X axis ticks -->
        {#each ticks as y}
          <line x1={xScale(y)} x2={xScale(y)} y1={innerH - 5} y2={innerH + 5} stroke="#888" stroke-width="2" />
          <text x={xScale(y)} y={innerH + 20} text-anchor="middle" font-size="13" fill="#555" font-weight="bold">{y}</text>
        {/each}

        <!-- Axis Label -->
        <text x={innerW / 2} y={innerH + 40} text-anchor="middle" font-size="16" fill="#333" font-weight="bold">Year</text>

        <!-- Y axis line -->
        <line x1={0} x2={0} y1={0} y2={innerH} stroke="#ccc" stroke-width="2" />

        <!-- Y axis ticks -->
        {#if yScale}
          {#each yScale.ticks(5) as s}
            <line x1={-5} x2={5} y1={yScale(s)} y2={yScale(s)} stroke="#888" stroke-width="2" />
            <text x={-10} y={yScale(s) + 4} text-anchor="end" font-size="13" font-weight="bold" fill="#555">
              {Math.round(s / 1000)}k
            </text>
          {/each}
        {/if}

        <!-- Y axis label -->
        <text transform="rotate(-90)" x={-innerH / 2} y={-45} text-anchor="middle" font-size="16" fill="#333" font-weight="bold">
          Students
        </text>

        <!-- Line plot -->
        {#if lineGen}
          <path d={lineGen(dataset)} fill="none" stroke="steelblue" stroke-width="2" />
        {/if}

        {#if plane}
          <text
            x={plane.x} y={plane.y} transform="rotate(-40, {plane.x}, {plane.y})" text-anchor="middle" dominant-baseline="middle" font-size="22" style="pointer-events:none"
          >
          <!-- the pointer event thing is a trick that stops the cursor from changing to a text cursor when hovering over the emoji lolz -->
            ✈️
          </text>
        {/if}
      </g>
    {/if}
  </g>
</svg>

