<script lang="ts" setup>
  const props = withDefaults(defineProps<{
    nodes?: NodeDefinition[];
    edges?: EdgeDefinition[];
  }>(), {
    nodes: () => [],
    edges: () => []
  })
  import { onMounted, ref } from "vue";
  import cytoscape from "cytoscape";
  import type { EdgeDefinition, NodeDefinition } from "cytoscape"
  const cy = ref();
  onMounted(() => {
    cytoscape({
      container: cy.value,
      elements: {
        nodes: props.nodes,
        edges: props.edges
      },
      layout: {
        name: 'preset'
      },
      style: fetch("./public/arrows.json").then((r) => r.json())
    })
  });
</script>
<template>
  <div ref="cy"
       id="cytoscape"></div>
</template>

<style>
#cytoscape {
  background-color: transparent;
  flex: 1;

}
</style>