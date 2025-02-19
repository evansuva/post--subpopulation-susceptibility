<script>
  import { onMount } from "svelte";
  import { range, sqrdist, getModelShape } from "../util.js";
  import * as svgPaths from "../svg-paths.js";

  export let initSpIndex;
  export let data;
  export let fID;

  let canvas;
  let svg;
  let slider;
  let playButton;
  let stepForwardButton, stepBackButton;

  let spIndex = initSpIndex;
  let hoverIndex = -1;
  let poisonIndex = 0;
  let poisons = [];
  let nPoisons = data.attacks[spIndex].poisons.length;
  let framerate = Math.max(Math.min(15, parseInt((nPoisons + 1) / 10)), 1);
  let isPlaying = true;
  let waiting = false;

  const width = 704;
  const height = 600;

  const render = () => {
    let dset = data.dset;
    const xValue = (p) => p.x[0];
    const yValue = (p) => p.x[1];
    const getClass = (p) => {
      if (p.subpops == undefined) return p.y == 1 ? "blue-poison" : "red-poison";
      if (p.subpops.includes(spIndex)) return "target-point";
      else if (p.subpops.includes(hoverIndex)) return "selected-point";
      else if (p.y == 1) return "blue-point";
      else return "red-point";
    };
    const getRadius = (p) => {
      if (p.subpops === undefined) return 4;
      else if (p.subpops.includes(hoverIndex) || p.subpops.includes(spIndex)) return 5;
      else return 4;
    };

    const margin = { top: 60, right: 40, bottom: 60, left: 40 };
    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    let extentX = d3.extent(dset, xValue);
    extentX = [extentX[0] - 0.1, extentX[1] + 0.1];
    const xScale = d3.scaleLinear().domain(extentX).range([0, innerWidth]).nice();

    let extentY = d3.extent(dset, yValue);
    extentY = [extentY[0] - 0.1, extentY[1] + 0.1];
    const yScale = d3.scaleLinear().domain(extentY).range([innerHeight, 0]).nice();

    const shadingG = d3
      .select(svg)
      .attr("pointer-events", "none")
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const dsetG = d3
      .select(svg)
      .attr("pointer-events", "none")
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const poisonG = d3
      .select(svg)
      .attr("pointer-events", "none")
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const modelG = d3
      .select(svg)
      .attr("pointer-events", "none")
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const xAxis = d3.axisBottom(xScale).tickSize(-innerHeight).tickPadding(15);
    xAxis.tickValues(range(11).map((x, i) => i / 10));
    const yAxis = d3.axisLeft(yScale).tickSize(-innerWidth).tickPadding(10);
    yAxis.tickValues(range(11).map((x, i) => i / 10));

    const yAxisG = dsetG.append("g").call(yAxis);

    const xAxisG = dsetG.append("g").call(xAxis).attr("transform", `translate(0,${innerHeight})`);

    let line = d3
      .line()
      .x((d) => xScale(d[0]))
      .y((d) => yScale(d[1]));

    modelG
      .append("clipPath")
      .attr("id", `rect-clip${fID}`)
      .append("rect")
      .attr("x", 0)
      .attr("y", 0)
      .attr("width", innerWidth)
      .attr("height", innerHeight);

    const model_c = modelG
      .append("line")
      .style("stroke", "darkgray")
      .style("stroke-width", 5)
      .attr("clip-path", `url(#rect-clip${fID})`);

    const model_t = modelG
      .append("line")
      .style("stroke", "black")
      .style("stroke-width", 5)
      .attr("clip-path", `url(#rect-clip${fID})`);

    const belowArea = shadingG.append("path").attr("clip-path", `url(#rect-clip${fID})`);

    const aboveArea = shadingG.append("path").attr("clip-path", `url(#rect-clip${fID})`);

    let dsetScatter = dsetG
      .selectAll("circle")
      .data(dset)
      .enter()
      .append("circle")
      .attr("class", getClass)
      .attr("cx", (d) => xScale(xValue(d)))
      .attr("cy", (d) => yScale(yValue(d)))
      .attr("r", (d) => getRadius(d));

    let poisonScatter = poisonG.selectAll("path");

    const delaunay = d3.Delaunay.from(data.cluster_centers);

    //--- figure interaction control ---//
    const updateClasses = () => {
      poisonScatter.attr("class", getClass).attr("r", getRadius);
      dsetScatter.attr("class", getClass).attr("r", getRadius);
    };

    const resetSlider = () => {
      slider.value = poisonIndex = 0;
      d3.select(slider).attr("max", nPoisons);
    };

    const pause = (paused) => {
      isPlaying = paused == undefined ? !isPlaying : paused;
      d3.select(playButton)
        .select("svg")
        .select("path")
        .attr("d", isPlaying ? svgPaths.pausePath : svgPaths.playPath);
    };

    const animStepFrame = () => {
      if (isPlaying && !waiting) {
        slider.value = (1 + +slider.value) % (nPoisons + 1);
        sliderHandler(false);
        if (poisonIndex == nPoisons) {
          waiting = true;
          setTimeout(() => (waiting = false), 1000);
        }
      }
      setTimeout(animStepFrame, 1000 / framerate);
    };

    const updateModels = () => {
      let modelShape;
      let theta_c = data.attacks[spIndex].im_models[0];
      let theta_t = data.attacks[spIndex].im_models[poisonIndex];

      modelShape = getModelShape(theta_c, extentX, extentY);
      model_c
        .attr("x1", xScale(modelShape.boundary[0][0]))
        .attr("x2", xScale(modelShape.boundary[1][0]))
        .attr("y1", yScale(modelShape.boundary[0][1]))
        .attr("y2", yScale(modelShape.boundary[1][1]));

      modelShape = getModelShape(theta_t, extentX, extentY);
      model_t
        .attr("x1", xScale(modelShape.boundary[0][0]))
        .attr("x2", xScale(modelShape.boundary[1][0]))
        .attr("y1", yScale(modelShape.boundary[0][1]))
        .attr("y2", yScale(modelShape.boundary[1][1]));
      belowArea.attr("d", line(modelShape.below)).attr("class", theta_t[1] < 0 ? "area-blue" : "area-red");
      aboveArea.attr("d", line(modelShape.above)).attr("class", theta_t[1] < 0 ? "area-red" : "area-blue");
    };

    const updateData = () => {
      poisons.forEach((e, i) => (e.id = i.toString()));
      poisonScatter = poisonG.selectAll("path").data(poisons, (d) => d.id);

      poisonScatter
        .enter()
        .append("path")
        .attr("class", (d) => getClass(d))
        .attr("d", d3.symbol().type(d3.symbolCross).size(600))
        .attr("transform", (d) => `translate(${xScale(xValue(d))},${yScale(yValue(d))})`)
        // .attr("cx", (d) => xScale(xValue(d)))
        // .attr("cy", (d) => yScale(yValue(d)));
        .transition()
        .duration(isPlaying ? 200 : 200)
        .attr("d", d3.symbol().type(d3.symbolCross).size(200));
      // .transition()
      // .duration(500)
      // .attr(
      //   "transform",
      //   (d) =>
      //     `translate(${xScale(xValue(d))},${yScale(yValue(d))})rotate(-45)`
      // );

      poisonScatter.exit().remove();
    };

    const mousemoveHandler = (event) => {
      let [x, y] = d3.pointer(event);
      (x -= margin.left), (y -= margin.top);
      [x, y] = [xScale.invert(x), yScale.invert(y)];
      hoverIndex = delaunay.find(x, y, spIndex);
      if (sqrdist(data.cluster_centers[hoverIndex], [x, y]) > 0.05) hoverIndex = -1;
      updateClasses();
    };

    const clickHandler = (event) => {
      if (hoverIndex != -1 && hoverIndex != spIndex) {
        spIndex = hoverIndex;
        nPoisons = data.attacks[spIndex].poisons.length;
        framerate = Math.max(Math.min(15, parseInt((nPoisons + 1) / 10)), 1);
        resetSlider();
        sliderHandler();
        updateClasses();
      }
    };

    const mouseoutHandler = (event) => {
      hoverIndex = -1;
      updateClasses();
      updateModels();
    };

    const sliderHandler = (isUser) => {
      if (isUser) pause(false);
      poisonIndex = +slider.value;
      poisons = data.attacks[spIndex].poisons.slice(0, poisonIndex);
      updateData();
      updateModels();
    };

    d3.select(canvas).on("mousemove", mousemoveHandler).on("click", clickHandler).on("mouseout", mouseoutHandler);

    d3.select(slider)
      .attr("max", nPoisons)
      .on("input", () => sliderHandler(true));

    d3.select(playButton).on("click", () => pause());
    d3.select(stepForwardButton).on("click", () => {
      slider.value = Math.min(+slider.value + 1, nPoisons);
      sliderHandler(true);
    });
    d3.select(stepBackButton).on("click", () => {
      slider.value = Math.max(+slider.value - 1, 0);
      sliderHandler(true);
    });

    updateData();
    updateData(); // probably a bug somewhere, but need to do this twice for highlighting subpops to work for some reason
    updateModels();

    setTimeout(animStepFrame, 1000 / framerate);
  };

  onMount(() => {
    render();
  });
</script>

<svg bind:this={svg} {width} {height} class="overlay">
  <text text-anchor="middle" x="50%" y="99%">{poisonIndex} / {nPoisons} Poisons</text></svg
>
<canvas bind:this={canvas} {width} {height} />
<button bind:this={playButton} class="button play-button" style="cursor: pointer">
  <svg width="10" height="10" viewBox="0 0 10 10">
    <path d={svgPaths.pausePath} fill="#888" />
  </svg>
</button>
<button bind:this={stepForwardButton} class="button step-forward-button" style="cursor: pointer">
  <svg width="10" height="10" viewBox="0 0 10 10">
    <path d={svgPaths.stepForwardPath} fill="#888" />
  </svg>
</button>
<button bind:this={stepBackButton} class="button step-back-button" style="cursor: pointer">
  <svg width="10" height="10" viewBox="0 0 10 10">
    <path d={svgPaths.stepBackPath} fill="#888" />
  </svg>
</button>
<input bind:this={slider} type="range" class="slider attack-slider" min="0" max="1" value="0" />
