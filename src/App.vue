<script setup>
import { ref } from 'vue'
import { VueFlow, useVueFlow, MarkerType, Position } from '@vue-flow/core'
import { Background } from '@vue-flow/background'
import { ControlButton, Controls } from '@vue-flow/controls'
import { MiniMap } from '@vue-flow/minimap'
import Icon from '@/components/Icon.vue'
import { sepolia } from 'viem/chains'
import { createPublicClient, http } from 'viem'
import { WORKFLOW_ABI } from '@/assets/abi/WORKFLOW_ABI'

/**
 * `useVueFlow` provides:
 * 1. a set of methods to interact with the VueFlow instance (like `fitView`, `setViewport`, `addEdges`, etc)
 * 2. a set of event-hooks to listen to VueFlow events (like `onInit`, `onNodeDragStop`, `onConnect`, etc)
 * 3. the internal state of the VueFlow instance (like `nodes`, `edges`, `viewport`, etc)
 */
const { onInit, onNodeDragStop, onConnect, addEdges, setViewport, toObject, updateNode } =
  useVueFlow()

const nodes = ref([])

const edges = ref([])

// our dark mode toggle flag
const dark = ref(false)

/**
 * This is a Vue Flow event-hook which can be listened to from anywhere you call the composable, instead of only on the main component
 * Any event that is available as `@event-name` on the VueFlow component is also available as `onEventName` on the composable and vice versa
 *
 * onInit is called when the VueFlow viewport is initialized
 */
onInit((vueFlowInstance) => {
  // instance is the same as the return of `useVueFlow`
  readOnchainData().then(() => {
    vueFlowInstance.fitView()
  })
})

/**
 * onNodeDragStop is called when a node is done being dragged
 *
 * Node drag events provide you with:
 * 1. the event object
 * 2. the nodes array (if multiple nodes are dragged)
 * 3. the node that initiated the drag
 * 4. any intersections with other nodes
 */
onNodeDragStop(({ event, nodes, node }) => {
  console.log('Node Drag Stop', { event, nodes, node })
})

/**
 * onConnect is called when a new connection is created.
 *
 * You can add additional properties to your new edge (like a type or label) or block the creation altogether by not calling `addEdges`
 */
onConnect((connection) => {
  addEdges(connection)
})

/**
 * To update a node or multiple nodes, you can
 * 1. Mutate the node objects *if* you're using `v-model`
 * 2. Use the `updateNode` method (from `useVueFlow`) to update the node(s)
 * 3. Create a new array of nodes and pass it to the `nodes` ref
 */
function updatePos() {
  nodes.value = nodes.value.map((node) => {
    return {
      ...node,
      position: {
        x: Math.random() * 400,
        y: Math.random() * 400,
      },
    }
  })
}

function readOnchainData() {
  const client = createPublicClient({
    chain: sepolia,
    transport: http('https://sepolia.infura.io/v3/3d3b5b68cc944f0494f62ef80ea683cb'),
  })

  return client
    .readContract({
      address: '0x957a5f77297a3B0F4f714e7cf9f58C9EeA3342a1',
      abi: WORKFLOW_ABI,
      functionName: 'stateIndex',
    })
    .then((states) => {
      client
        .readContract({
          address: '0x957a5f77297a3B0F4f714e7cf9f58C9EeA3342a1',
          abi: WORKFLOW_ABI,
          functionName: 'transitionIndex',
        })
        .then(async (transitions) => {
          let yPos = 50,
            xPos = 250
          for (let stateIndex = 1; stateIndex < Number(states); stateIndex++) {
            let state = await client.readContract({
              address: '0x957a5f77297a3B0F4f714e7cf9f58C9EeA3342a1',
              abi: WORKFLOW_ABI,
              functionName: 'states',
              args: [stateIndex],
            })
            yPos = yPos + 100
            nodes.value.push({
              id: stateIndex.toString(),
              data: { label: state[0].toString() },
              position: { x: xPos, y: yPos },
              class: 'light',
            })
          }
          for (let transitionIndex = 1; transitionIndex < Number(transitions); transitionIndex++) {
            let transition = await client.readContract({
              address: '0x957a5f77297a3B0F4f714e7cf9f58C9EeA3342a1',
              abi: WORKFLOW_ABI,
              functionName: 'transitions',
              args: [transitionIndex],
            })
            edges.value.push({
              id: transitionIndex,
              label: transition[0],
              source: transition[1].toString(),
              target: transition[2].toString(),
              animated: Boolean(transition[3]),
              markerEnd: MarkerType.ArrowClosed,
            })
          }
          adjustView()
        })
    })
}

function adjustView() {
  updateNode('1', { type: 'input' })
  nodes.value.forEach((node) => {
    if (!edges.value.find((edge) => edge.source === node.id)) {
      //no edges 'from' this node
      updateNode(node.id, { type: 'output' })
    }
  })
}

/**
 * toObject transforms your current graph data to an easily persist-able object
 */
function logToObject() {
  console.log(toObject())
}

/**
 * Resets the current viewport transformation (zoom & pan)
 */
function resetTransform() {
  setViewport({ x: 0, y: 0, zoom: 1 })
}

function toggleDarkMode() {
  dark.value = !dark.value
}
</script>

<template>
  <VueFlow
    :nodes="nodes"
    :edges="edges"
    :class="{ dark }"
    class="basic-flow"
    :min-zoom="0.2"
    :max-zoom="4"
  >
    <Background pattern-color="#aaa" :gap="16" />

    <MiniMap />

    <Controls position="top-left">
      <ControlButton title="Reset Transform" @click="resetTransform">
        <Icon name="reset" />
      </ControlButton>

      <ControlButton title="Shuffle Node Positions" @click="updatePos">
        <Icon name="update" />
      </ControlButton>

      <ControlButton title="Toggle Dark Mode" @click="toggleDarkMode">
        <Icon v-if="dark" name="sun" />
        <Icon v-else name="moon" />
      </ControlButton>

      <ControlButton title="Log `toObject`" @click="logToObject">
        <Icon name="log" />
      </ControlButton>
    </Controls>
  </VueFlow>
</template>
