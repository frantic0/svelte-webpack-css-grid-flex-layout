<script>
  import Grid from "svelte-grid";
  import gridHelp from "svelte-grid/build/helper";
  import map from "lodash.map";
  import Editor from '../Editor.svelte';

  export let value = '';

  const id = () =>
    "_" +
    Math.random()
      .toString(36)
      .substr(2, 9);

  const random = (min, max) => Math.random() * (max - min) + min;

  const randomHexColorCode = () => {
    let n = (Math.random() * 0xfffff * 1000000).toString(16);
    return "#" + n.slice(0, 6);
  };
  let items = [];

  function generateLayout(col) {
    return map(new Array(20), function(item, i) {
      const x = Math.ceil(Math.random() * 3) + 2;
      const y = Math.ceil(Math.random() * 4) + 1;
      return {
        ...gridHelp.item({
          x: (i * 2) % col,
          y: Math.floor(i / 6) * y,
          w: x,
          h: y,
          id: id(),
        }),
        ...{ data: randomHexColorCode() },
      };
    });
  }

  var cols = 15;

  items = generateLayout(cols);
  items = gridHelp.resizeItems(items, cols);

  let breakpoints = [[1000, 10], [700, 5], [500, 3], [400, 1]];
</script>


<style>
.layout-template-container {
  height: 100vh;
}

.content {
  width: 100%;
  height: 100%;
  border-radius: 6px;
  border-bottom-right-radius: 3px;
}

:global(*) {
  user-select: none;
}

:global(body) {
  overflow: scroll;
  margin: 0;
}

:global(.svlt-grid-resizer) {
  z-index: 1500;
}

:global(.svlt-grid-resizer::after) {
  border-color: white !important;
}

:global(.svlt-grid-transition > .svlt-grid-item) {
  transition: transform 0.2s;
}

:global(.svlt-grid-shadow) {
  background: pink;
  border-radius: 6px;
  border-bottom-right-radius: 3px;
   /*transition: top 0.2s, left 0.2s;*/
  transition: transform 0.2s;
}

</style>

<div class="layout-template-container">
  <Grid useTransform {breakpoints} gap={1} {items} bind:items {cols} rowHeight={100} let:item>
    <div class="content" style="background: {item.static ? '#ccccee' : item.data}" >
      <Editor bind:value={value}/>
    </div>
  </Grid>
</div>
