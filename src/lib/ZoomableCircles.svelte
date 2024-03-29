<script>
  import { interpolateZoom, interpolateHcl } from 'd3-interpolate'
  import { hierarchy, pack } from 'd3-hierarchy'
  import { transition } from 'd3-transition'
  import { scaleLinear } from 'd3-scale'
	import { onMount } from 'svelte';

  // Tree & UI state
  export let tree = {
    name: "root",
    children: [
      {
        name: "child1",
        children: [
          {
            name: "child1.1",
            value: 1000
          },
          {
            name: "child1.2",
            value: 500
          },
          {
            name: "child1.3",
            value: 250
          },
        ]
      },
      {
        name: "child2",
        value: 500
      },
      {
        name: "child3",
        value: 250
      },
    ]
  }
  export let svgWidth = 300
  export let svgHeight = 300
  export let startColor = '#555'
  export let endColor = '#ccc'
  export let textFillColor = '#000'
  export let circleHoverStrokeColor = '#000'
  export let circleHoverStrokeWidth = 3
  export let textFontSize = 20
  let innerWidth
  let innerHeight
  let root
  let view
  let activeFocus
  let activeZoomA
  let activeZoomB
  let activeZoomK
  let packFunc
  let clickedObjectType = "branch"
  let currentNode

  $: {
    // Rerender SVG on screen width change
    if (innerWidth) {
      if (innerWidth < innerHeight) {
        innerHeight = innerWidth;
      } else {
        innerWidth = innerHeight;
      }
      svgWidth = innerWidth;
      svgHeight = innerHeight;
      setPackFunc()
      resetRoot()
      returnToCurrentNode()
    }
  }

  onMount(async () => {
    currentNode = { name: tree.name, path: [tree.name], value: null }
    setPackFunc()
    resetRoot()
    resetActiveFocus()
  })

  /* UTILITY FUNCTIONS */

  const resetRoot = () => {
    if (tree) {
      root = packFunc(tree)
    }
  }

  const getPath = (data) => {
    let path = [data.data.name];
    let parent = data.parent;
    while (parent) {
      path = [parent.data.name, ...path];
      parent = parent.parent;
    }
    return path;
  }

  const setPackFunc = () => {
    packFunc = (data) => pack()
      .size([svgWidth, svgHeight])
      .radius((d) => !d.value ? 5 : d.value / 200)
      .padding(4)
      (hierarchy(data)
        .sum(d => d.value)
        .sort((a, b) => b.value - a.value));
  }

  const returnToCurrentNode = () => {
    clickedCategory(currentNode?.name)
  }

  const color = scaleLinear()
    .domain([0, 5])
    .range([startColor, endColor])
    .interpolate(interpolateHcl);

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
      .duration(750)
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
    const rootData = root?.descendants()?.find((d) => d.data.name === name);
    if (rootData) {
      const path = getPath(rootData)
      zoom(rootData, e);
      clickedObjectType = 'value' in rootData.data ? 'leaf' : 'branch'
      currentNode = {
        name,
        path,
        value: rootData.data.value || null
      }
    }
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
    role="button"
    tabindex="0"
    on:click={(e) => {
      zoom(root, e)
      clickedObjectType = "branch"
      currentNode = { name: tree.name, path: [tree.name], value: null }
    }}
    on:keydown={() => {}}
  >
    <g transform="translate({svgWidth / 2},{svgHeight / 2})">
      {#each root.descendants().slice(1) as rootData}
        <circle
          style="
            cursor: pointer;
            {rootData.parent 
              ? rootData.data.children
                ? '' 
                : `fill: ${startColor};` 
              : 'pointer-events: none;'
            }
          "
          role="button"
          tabindex="-1"
          fill={rootData.data.children ? color(rootData.depth) : 'null'} 
          transform="translate({(rootData.x - activeZoomA) * activeZoomK},{(rootData.y - activeZoomB) * activeZoomK})"
          r={rootData.r * activeZoomK}
          onmouseover="evt.target.setAttribute('stroke-width', '{circleHoverStrokeWidth}'); evt.target.setAttribute('stroke', '{circleHoverStrokeColor}');"
          onmouseout="evt.target.setAttribute('stroke-width', '0'); evt.target.setAttribute('stroke', 'transparent');"
          on:click={(e) => clickedCircle(e, rootData)}
          on:keydown={() => {}}
        ></circle>
      {/each}
      {#each root.descendants() as rootDes}
        <text 
          font-size="{textFontSize}px"
          style="
            pointer-events: none; 
            text-anchor: middle;
            fill: {textFillColor};
            fill-opacity: {
              rootDes.parent === activeFocus || (
                currentNode && 
                currentNode.name === rootDes?.data?.name &&
                !!currentNode.value
              ) ? 1 : 0}; 
            display: {
              rootDes.parent === activeFocus || (
                currentNode && 
                currentNode.name === rootDes?.data?.name &&
                !!currentNode.value
              ) ? "inline" : "none"
            };
          "
          transform="translate({(rootDes.x - activeZoomA) * activeZoomK},{(rootDes.y - activeZoomB) * activeZoomK})"
        >
          {rootDes?.data?.name}
        </text>
      {/each}
    </g>
  </svg>
{/if}