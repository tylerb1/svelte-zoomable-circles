<script>
  import { interpolateZoom, interpolateHcl } from 'd3-interpolate'
  import { hierarchy, pack } from 'd3-hierarchy'
  import { transition } from 'd3-transition'
  import { scaleLinear } from 'd3-scale'
	import { onMount } from 'svelte';
  import { sampleTree } from './sampleTree.js';

  export let tree = sampleTree
  export let svgWidth = 300
  export let svgHeight = 300
  export let startColor = '#555'
  export let endColor = '#ccc'
  export let textFillColor = '#000'
  export let circleHoverStrokeColor = '#000'
  export let circleHoverStrokeWidth = 3
  export let textFontSize = 25
  export let padding = 100
  export let zoomDurationMs = 500
  let currentNode = { name: tree.name, path: [tree.name], value: null }
  let innerWidth
  let innerHeight
  let root
  let view
  let activeFocus
  let activeZoomA
  let activeZoomB
  let activeZoomK
  let packFunc
  let longestPathLength
  let color

  $: {
    if (innerWidth) {
      if (innerWidth < innerHeight) {
        innerHeight = innerWidth;
      } else {
        innerWidth = innerHeight;
      }
      svgWidth = innerWidth;
      svgHeight = innerHeight;
      setPackFunc()
      root = packFunc(tree)
      clickedCategory(currentNode.name)
    }
  }

  $: if (longestPathLength) {
    color = scaleLinear()
      .domain([0, longestPathLength])
      .range([startColor, endColor])
      .interpolate(interpolateHcl);
  }

  onMount(async () => {
    longestPathLength = findLongestPath(tree);
    setPackFunc()
    root = packFunc(tree)
    resetActiveFocus()
  })

  const getPath = (data) => {
    let path = [data.data.name];
    let parent = data.parent;
    while (parent) {
      path = [parent.data.name, ...path];
      parent = parent.parent;
    }
    return path;
  }

  const findLongestPath = (node, currentPathLength = 0) => {
    if (node.value !== undefined) {
      return currentPathLength;
    }
    let longestPath = 0;
    if (node.children) {
      for (const child of node.children) {
        const pathLength = findLongestPath(child, currentPathLength + 1);
        longestPath = Math.max(longestPath, pathLength);
      }
    }
    return longestPath;
  }

  const setPackFunc = () => {
    packFunc = (data) => pack()
      .size([svgWidth, svgHeight])
      .radius((d) => d.value)
      .padding(padding)
      (hierarchy(data)
        .sum(d => d.value)
        .sort((a, b) => b.value - a.value));
  }

  const inactiveZoomTo = (v) => {
    activeZoomK = svgWidth / v[2];
    activeZoomA = v[0];
    activeZoomB = v[1];
    view = v;
  };

  const resetActiveFocus = () => {
    activeFocus = root;
    activeZoomA = root.x;
    activeZoomB = root.y;
    activeZoomK = svgWidth / root.r * 2;
    inactiveZoomTo([root.x, root.y, root.r * 2]);
  }

  const zoom = (d, e) => {
    if (e) {
      e.stopPropagation();
    }
    activeFocus = d;
    transition()
      .duration(zoomDurationMs)
      .tween('zoom', () => {
        var i = interpolateZoom(
          view, 
          [
            activeFocus.x, 
            activeFocus.y, 
            activeFocus.r * 2
          ]
        );
        return function(t) { 
          inactiveZoomTo(i(t)); 
        };
      });
  };

  const clickedCircle = (e, rootData) => {
    if (activeFocus !== rootData) {
      clickedCategory(rootData.data.name, e)
    } else {
      currentNode = { name: tree.name, path: [tree.name], value: null }
    }
  }

  const clickedCategory = (name, e) => {
    const rootData = root.descendants().find((d) => d.data.name === name);
    const path = getPath(rootData)
    currentNode = {
      name,
      path,
      value: rootData.data.value || null
    }
    zoom(rootData, e);
  }
</script>

<svelte:window bind:innerHeight bind:innerWidth on:keydown={(event) => {
  if (event.key === 'Escape') {
    const previousIndex = currentNode.path.length === 1 ? 0 : currentNode.path.length - 2;
    clickedCategory(currentNode.path[previousIndex]);
  }
}}/>

{#if root}
  <svg 
    height={svgHeight}
    width={svgWidth}
    style="background: {endColor};" 
    on:click={(e) => {
      zoom(root, e)
      currentNode = { name: tree.name, path: [tree.name], value: null }
    }}
    role="button"
    tabindex="-2"
    on:keydown={() => {}}
  >
    <g transform="translate({svgWidth / 2},{svgHeight / 2})">
      {#each root.descendants().slice(1) as rootData}
        <circle
          style="cursor: pointer;"
          fill={rootData.data.children ? color(rootData.depth) : startColor} 
          transform="translate({(rootData.x - activeZoomA) * activeZoomK},{(rootData.y - activeZoomB) * activeZoomK})"
          r={rootData.r * activeZoomK}
          onmouseover="evt.target.setAttribute('stroke-width', '{circleHoverStrokeWidth}'); evt.target.setAttribute('stroke', '{circleHoverStrokeColor}');"
          onmouseout="evt.target.setAttribute('stroke-width', '0'); evt.target.setAttribute('stroke', 'transparent');"
          on:click={(e) => clickedCircle(e, rootData)}
          role="button"
          tabindex="-1"
          on:keydown={() => {}}
        ></circle>
      {/each}
      {#each root.descendants() as rootDes}
        <text 
          font-size="{textFontSize}px"
          transform="translate({(rootDes.x - activeZoomA) * activeZoomK},{(rootDes.y - activeZoomB) * activeZoomK})"
          style="
            pointer-events: none; 
            text-anchor: middle;
            dominant-baseline: central;
            fill: {textFillColor};
            display: {
              rootDes.parent === activeFocus || (
                currentNode && 
                currentNode.name === rootDes.data.name &&
                !!currentNode.value
              ) ? "inline" : "none"
            };
          "
        >
          {rootDes.data.name}
        </text>
      {/each}
    </g>
  </svg>
{/if}