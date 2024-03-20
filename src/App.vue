<script setup lang="ts">
import { ref, watch, nextTick, reactive } from "vue";
import yaml from "js-yaml";
import Cytoscape from "./components/Cytoscape.vue";
import type { EdgeDefinition, ElementsDefinition, NodeDefinition } from "cytoscape";
/* 
  need to be able to make a docker compose so i need to add:
    - ability to add nodes and edges
    - delete nodes and edges
    - connect nodes and determine network vs depends on connections
    - edit docker specific data -> will go into node.data or maybe node.scratch


  Networks need to be nodes - using different line styles is confusing

  might want to make networks "parent nodes"?
  might want to make volumes children nodes of services?

*/
type UserOptions = {
  networksAsNodes: boolean;
}
const userOptions = reactive<UserOptions>({
  networksAsNodes: true
})
const nodes = ref<NodeDefinition[] | null>(null);
const edges = ref<EdgeDefinition[] | null>(null);
const fileData = ref<ElementsDefinition | null>(null);
const fileInputEle = ref();

const readFile = async (file: Blob) => {
  const file_data = await file.text();
  const dockerCompose = yaml.load(file_data);
  fileData.value = composeToCytosape(dockerCompose, userOptions);
};

const composeToCytosape = (compose: any, opts: UserOptions): ElementsDefinition => {
  const parsed = { nodes: [], edges: [] } as ElementsDefinition;

  if (Object.hasOwn(compose, "networks")) {
    if (opts.networksAsNodes) {
      const networksNodes = networksToNodes(compose);
      parsed.nodes.push(...networksNodes.nodes);
      parsed.edges.push(...networksNodes.edges);
    } else {
      parsed.edges.push(...networksToEdges(compose.networks, compose.services))
    }
  }

  if (Object.hasOwn(compose, "services")) {
    /* 
      Need to create dependsOnToEdges 
      Find all services that have "depends on" values and create edge with arrow pointing towards the node that "depends on" the other

      - If when calling servicesToNodes we find a service with a depends on value we can easily have a side effect in that .map!
    */
    const services = servicesToNodes(compose.services);
    parsed.nodes.push(...services.nodes);
    parsed.edges.push(...services.edges)
  }
  return parsed;
};
const networksToEdges = (
  networks: { [key: string]: any },
  services: { [key: string]: any },
) => {
  /* 
    Need to loop over networks Object
    Find all services that use network
    Create Edge for each connection
  */
  const names: {[key: string]: string[]} = {};
  Object.keys(networks).forEach((k) => names[k] = []);
  Object.entries(services).forEach(([id, data]) => {
    const hasNetwork = Object.hasOwn(data, "networks");
    if (!hasNetwork) return;
    for (const nw of data.networks) {
      names[nw].push(id);
    }
  });

  const edgeArr = [] as EdgeDefinition[];
  Object.entries(names).forEach(([id, arr]) => {
    for (let i = 0; i < arr.length - 1; i++) {
      edgeArr.push({
        data: {
          id,
          source: arr[i],
          target: arr[i + 1]
        }
      })
    }
  })
  return edgeArr;
};
const networksToNodes = ({networks, services}: {networks: any, services: any}) =>{ 
  const { nodes, edges } = servicesToNodes(networks);
  Object.entries(services).forEach(([id, data]: any) => {
    const hasNetwork = Object.hasOwn(data, "networks");
    if (!hasNetwork) return;
    for (const nw of data.networks) {
      // {
      //     data: {
      //       id: "depends_on",
      //       arrow: "triangle",
      //       source,
      //       target: id
      //     }
      //   }
      edges.push({
        data: {
          id: id + "_" + nw,
          source: nw,
          target: id
        }
      })
    }
  });
  return {nodes: nodes.map(({data, ...n}) => {
    return {
      ...n,
      data: {
        ...data,
        shape: "triangle",
        backgroundColor: "red"
      }
    }
  }), edges}

};
const servicesToNodes = (services: { [key: string]: any }) => {
  /* 
    Need to x coordinates based off of the services "depends on" value
    - Call side effect function based on service that has "depends on" and that an easily make the edge from the .map()!
  */
  const dependsOnEdges = [] as EdgeDefinition[];
  const nodes = Object.entries(services).map(([id, data], idx) => {
    const isDependent = data && Object.hasOwn(data, "depends_on");
    if (isDependent) {
      data.depends_on.forEach((source: string) => {
        dependsOnEdges.push({
          data: {
            id: "depends_on",
            arrow: "triangle",
            source,
            target: id
          }
        })
      })
    }
    return {
      data: {
        id,
        ...data,
      },
      position: {
        x: isDependent ? data.depends_on.length * 100 : 0,
        y: isDependent ? 0 : idx * 100,
      },
    };
  });
  return {
    nodes: nodes.map(({data, ...n}): any => {
      return {
        ...n,
        data: {
          ...data,
          backgroundColor: "dodgerblue",
          shape: "circle"
        }
      }
    }),
   edges: dependsOnEdges}
};


const resetCytoscape = () => {
  nodes.value = [
    {
      data: {
        id: "service one",
      },
      renderedPosition: {
        x: 0,
        y: 0,
      },
    },
  ];
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
    <button class="upload" @click="fileInputEle.click()">
      upload
      <input
        ref="fileInputEle"
        type="file"
        @change="(e: any) => readFile(e.target.files.item(0))"
      />
    </button>
  </header>
  <main id="app-main">
    <Cytoscape
      v-if="Array.isArray(nodes) && Array.isArray(edges)"
      v-bind="{ nodes, edges }"
    />
  </main>
  <section class="info-container" v-if="!nodes">
    <span class="info">
      <p>Please upload a docker-compose.yml file</p>
    </span>
  </section>
</template>

<style>
#app-header {
  border-bottom: 1px solid white;
  flex: 0 0 2.5%;
  display: flex;
  column-gap: 2rem;
  padding: 0.5ch;
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
  border-radius: 0.5rem;
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
