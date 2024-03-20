<script lang="ts" setup>
import { watchEffect, ref } from "vue";
import cytoscape from "cytoscape";
import type { EdgeDefinition, NodeDefinition } from "cytoscape";

const props = withDefaults(
  defineProps<{
    nodes?: NodeDefinition[];
    edges?: EdgeDefinition[];
  }>(),
  {
    nodes: () => [],
    edges: () => [],
  },
);

const cy = ref();
const isLoading = ref(true);
const initCytoscape = () => {
  isLoading.value = true;
  cytoscape({
    container: cy.value,
    elements: {
      nodes: props.nodes,
      edges: props.edges,
    },
    layout: {
      name: "preset",
    },
    style: fetch("/arrows.json").then((r) => r.json()),
  });
  setTimeout(() => {
    isLoading.value = false;
  }, 2000);
};
watchEffect(initCytoscape);
</script>
<template>
  <div ref="cy" id="cytoscape">
    <Teleport to="#app">
      <section class="info-container backdrop" v-if="isLoading">
        <span class="info">
          <p>loading...</p>
        </span>
      </section>
    </Teleport>
  </div>
</template>

<style scoped>
#cytoscape {
  background-color: transparent;
  flex: 1;
  position: relative;
}
</style>
