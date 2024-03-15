<script setup lang="ts">
import { ref, watch, nextTick } from "vue";
import yaml from "js-yaml";
import Cytoscape from './components/Cytoscape.vue';
import { EdgeDefinition, ElementsDefinition, NodeDefinition } from "cytoscape";
/* 
  need to be able to make a docker compose so i need to add:
    - ability to add nodes and edges
    - delete nodes and edges
    - connect nodes and determine network vs depends on connections
    - edit docker specific data -> will go into node.data or maybe node.scratch

  network connections are line
  depends on connections are a line with arrow pointing to node that DEPENDS ON other node

  might want to make networks their own nodes?

  might want to make networks "parent nodes"?

  need to take docker compose yaml and parse data so I can find nodes that are connected via networks and nodes that depend on each other
*/
const nodes = ref<NodeDefinition[] | null>(null);
const edges = ref<EdgeDefinition[] | null>(null);

const fetchDevData = async (path: string | URL) => await fetch(path).then((res) => {
  return res.json();
}).catch((e) => {
  throw new Error(e);
});

const fileData = ref<ElementsDefinition | null>(null);
const fileInputEle = ref();

const readFile = async (file: Blob) => {
  const file_data = await file.text();
  const dockerCompose = yaml.load(file_data);
  fileData.value =  composeToCytosape(dockerCompose.services);
}

const composeToCytosape = (compose: any): ElementsDefinition => {
  const parsed = {nodes: [], edges: []} as ElementsDefinition;

  if (Object.hasOwn(compose, "services")) {
    /* 
      Need to create dependsOnToEdges 
      Find all services that have "depends on" values and create edge with arrow pointing towards the node that "depends on" the other

      - If when calling servicesToNodes we find a service with a depends on value we can easily have a side effect in that .map!
    */
    parsed.nodes = servicesToNodes(compose.services);
  }

  if (Object.hasOwn(compose, "networks")) {
    parsed.edges = networksToEdges(compose.networks, compose.services);
  }

  return parsed;
}
const networksToEdges = (networks: {[key: string]: any}, services: {[key: string]: any}) => { 
  /* 
    Need to loop over networks Object
    Find all services that use network
    Create Edge for each connection
  */
  // return Object.entries(obj).map(([id, data], idx) => {});
  return [];
}
const servicesToNodes = (services: {[key: string]: any}) => { 
  /* 
    Need to x coordinates based off of the services "depends on" value
    - Call side effect function based on service that has "depends on" and that an easily make the edge from the .map()!
  */
  return Object.entries(services).map(([id, data], idx) => {
    return {
      data: {
        id,
        ...data
      },
      renderPosition: {
        x: 0,
        y: idx * 100  
      }
    }
  });
}

const resetCytoscape = () => {
  nodes.value = [
    {
      "data": {
        "id": "service one"
      },
      "renderedPosition": {
        "x": 0,
        "y": 0
      }
    }];
    edges.value = [];
};

watch(fileData, async (nv) => {
  if (!nv) return;
  if (Object.hasOwn(nv, "nodes")) {
    await nextTick();
    nodes.value = nv.nodes;
    edges.value = nv.edges;
  }
});
</script>

<template>
<header id="app-header">
  <button @click="resetCytoscape" class="new">new</button>
  <button class="upload" @click="fileInputEle.click()">upload
    <input
      ref="fileInputEle"
      type="file"
      @change="(e: any) => readFile(e.target.files.item(0))" />
  </button>
</header>
<main id="app-main">
  <Cytoscape v-if="Array.isArray(nodes) && Array.isArray(edges)" v-bind="{ nodes, edges }" />
</main>
<section class="info-container"
      v-if="!nodes">
  <span class="info">
    <p>Please upload a docker-compose.yml file</p>
  </span>
</section>
</template>

<style>
#app-header {
  border-bottom: 1px solid white;
  flex: 0 0 2.50%;
  display: flex;
  column-gap: 2rem;
  padding: .5ch;
}

#app-main {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: transparent;
  flex: 1 1;
}

.info-container {
  position: absolute;
  text-align: center;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 100;
}

.backdrop {
  width: 100dvw;
  height: 100dvh;
  backdrop-filter: blur(5px);
  display: flex;
}

.info {
  border: 2px dashed aliceblue;
  border-radius: .5rem;
  padding: 1rem;
  width: 40dvw;
  height: 20dvh;
  margin: auto;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #242424;
}

input[type="file"] {
  display: none;
}

button {
  padding: 2.5px;
  width: 4rem;
  cursor: pointer;
}
</style>