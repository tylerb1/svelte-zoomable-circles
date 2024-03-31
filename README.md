# svelte-zoomable-circles

A zoomable circle component for displaying and browsing hierarchical data

Built with Svelte and D3.js

# demo

![demo](https://i.imgur.com/Yi82D5L.gif)

Live demo: [openopensource.com](https://openopensource.com) 

# usage

```
<script>
	import ZoomableCircles from "svelte-zoomable-circles";
</script>

<ZoomableCircles tree={treeObject} />
```

The `tree` object must be a JSON object in the following format:

```
export const sampleTree = {
  name: "Root",
  children: [
    {
      name: "Child 1",
      children: [
        {
          name: "Child 1.1",
          value: 1000
        },
        {
          name: "Child 1.2",
          value: 500
        },
        {
          name: "Child 1.3",
          value: 250
        },
      ]
    },
    {
      name: "Child2",
      value: 800
    },
    {
      name: "Child3",
      value: 600
    },
  ]
}

```

# props

| Prop                      | Type                  | Default         | Description                                   | 
|---------------------------|-----------------------|-----------------|-----------------------------------------------|
| `tree`                    | `object`              | `sampleTree`    | Component data                                |
| `svgHeight`               | `number`              | `300`           | Container height                              |
| `svgWidth`                | `number`              | `300`           | Container width                               |
| `startColor`              | `string (hex code)`   | `#555`          | Start color for node color gradient           |
| `endColor`                | `string (hex code)`   | `#ccc`          | End color for node gradient                   |
| `textFillColor`           | `string (hex code)`   | `#000`          | Color of node labels                          |
| `circleHoverStrokeColor`  | `string (hex code)`   | `#000`          | Color of circle outline on hover              |
| `circleHoverStrokeWidth`  | `number`              | `3`             | Width of circle outline on hover in px        |
| `textFontSize`            | `number`              | `25`            | Size of label font in px                      |
| `padding`                 | `number`              | `100`           | Transition duration (ms)                      |
| `zoomDurationMs`          | `number`              | `500`           | Enables autoplay of pages                     |

# authors

- built by [tylerb1](https://github.com/tylerb1) 
- adapted from [this demo](https://svend3r.dev/charts/circlePack) by Svend3r
