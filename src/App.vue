<script setup lang="ts">
import Cytoscape from './components/Cytoscape.vue';

const edges = [
    {
      data: {
        id: "network",
        source: "service one",
        target: "service two"
      }
    } ,
    {
      data: {
        id: "depends-on",
        arrow: "triangle",
        source: "service one",
        target: "service three",
      }
      
    }
  ];

  const nodes = [  {
    data: { 
      id: 'service one', 
    },
    renderedPosition: {
      x: 0,
      y: 0
    }
  },
  {
    data: { 
      id: 'service two', 
    },
    renderedPosition: {
      x: 0,
      y: 100
    }
  },
  {
    data: { 
      id: 'service three',
      "depends-on": "service two"
    },
    renderedPosition: {
      x: 100,
      y: 0
    }
  }];

/* 
  need to be able to make a docker compose so i need to add:
   - ability to add nodes and edges
   - delete nodes and deges
   - connect nodes and determine network vs depends on connections
   - edit docker specific data -> will go into node.data or maybe node.scratch

  network connections are line
  depends on connections are a line with arrow pointing to node that DEPENDS ON other node

  might want to make networks their own nodes?

  might want to make networks "parent nodes"?

  need to take docker compose yaml and parse data so I can find nodes that are connected via networks and nodes that depend on each other
*/
</script>

<template>

<header id="app-header">
  <button>new</button>
  <button>upload</button>
</header> 
<main id="app-main">
  <Cytoscape v-bind="{nodes, edges}"  />
</main>

</template>

<style scoped>
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
</style>