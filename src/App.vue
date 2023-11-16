<template>
  <div class="h-screen flex gap-x-4 bg-red-200 p-2">
    <div class="flex flex-col gap-y-2 w-1/3">
      <input class="block" type="file" @change="change" accept="text/csv" />

      <textarea
        v-model="csvText"
        class="h-full"
        wrap="off"
        @drop.prevent="drop"
        @dragover.prevent="dragover"
      />
    </div>
    <div class="flex flex-col gap-y-2 w-2/3">
      <div class="flex gap-x-2">
        <button @click="tab = 'dot'">dot</button>
        <button @click="tab = 'svg'">svg</button>
        <button @click="download">download</button>
      </div>

      <textarea v-if="tab === 'dot'" readonly :value="dotText" class="h-full" wrap="off" />

      <iframe v-else-if="tab === 'svg'" :srcdoc="svgText" class="h-full w-full" />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import snakeCase from 'lodash/snakeCase'
import groupBy from 'lodash/groupBy'
import { parse } from 'csv-parse/browser/esm/sync'
import graphviz from 'graphviz-wasm'

const wasmLoaded = ref(false)
const csvText = ref('')
const tab = ref('svg')

const dotText = computed(() => {
  const records = parse(csvText.value, {
    columns: true,
    group_columns_by_name: true
  })

  const issues = records.map((record) => {
    let blockedBy = record['Inward issue link (Blocks)']
    if (!Array.isArray(blockedBy)) blockedBy = [blockedBy]
    let blocks = record['Outward issue link (Blocks)']
    if (!Array.isArray(blocks)) blocks = [blocks]

    return {
      key: record['Issue key'],
      summary: record['Summary'],
      type: record['Issue Type'],
      epic: record['Parent summary'],
      blockedBy: blockedBy.filter((id) => id.length > 0),
      blocks: blocks.filter((id) => id.length > 0)
    }
  })

  let dot = ''
  const println = (text) => (dot += text + '\n')

  println('digraph G {')
  println('rankdir=LR;')
  println('node[shape=record, style=filled, color=white];')

  // Nodes
  for (const issue of issues) {
    const node = snakeCase(issue.key)
    const label = `${issue.key}\\n${issue.summary}`
    println(`${node} [label="${label}"];`)
  }

  // Edges
  for (const issue of issues) {
    if (issue.blocks.length > 0) {
      const src = snakeCase(issue.key)
      for (const blocked of issue.blocks) {
        const dest = snakeCase(blocked)
        println(`${src} -> ${dest};`)
      }
    }
  }

  // Clusters
  for (const [epic, is] of Object.entries(groupBy(issues, 'epic'))) {
    println(`subgraph "${epic}" {`)
    println('cluster=true;')
    println('style=filled;')
    println(`label="${epic}";`)
    for (const issue of is) {
      println(`${snakeCase(issue.key)};`)
    }
    println('}')
  }

  println('}')
  return dot
})

const svgText = computed(() => {
  if (!wasmLoaded.value) return ''
  return graphviz.layout(dotText.value)
})

function download() {
  const parser = new DOMParser()
  const svgDoc = parser.parseFromString(svgText.value, 'image/svg+xml')
  const svgNode = svgDoc.querySelector('svg')
  const svgString = new XMLSerializer().serializeToString(svgNode)

  const svgBlob = new Blob([svgString], {
    type: 'image/svg+xml;charset=utf-8'
  })

  const url = window.URL.createObjectURL(svgBlob)

  const image = new Image()
  image.width = svgNode.width.baseVal.value
  image.height = svgNode.height.baseVal.value
  image.src = url
  image.addEventListener('load', () => {
    const canvas = document.createElement('canvas')
    canvas.width = image.width
    canvas.height = image.height

    const ctx = canvas.getContext('2d')
    ctx.drawImage(image, 0, 0)
    window.URL.revokeObjectURL(url)

    const imgURI = canvas.toDataURL('image/png').replace('image/png', 'image/octet-stream')

    const a = document.createElement('a')
    a.download = 'jira_graph.png'
    a.target = '_blank'
    a.href = imgURI

    a.dispatchEvent(
      new MouseEvent('click', {
        view: window,
        bubbles: false,
        cancelable: true
      })
    )
  })
}

function processFile(file) {
  const reader = new FileReader()
  reader.addEventListener('load', () => {
    csvText.value = reader.result
  })
  reader.readAsText(file)
}

function drop(event) {
  if (event.dataTransfer.items) {
    // Use DataTransferItemList interface to access the file(s)
    for (const item of [...event.dataTransfer.items]) {
      if (item.kind === 'file') {
        const file = item.getAsFile()
        processFile(file)
        break
      }
    }
  } else {
    for (const file of [...event.dataTransfer.files]) {
      processFile(file)
      break
    }
  }
}

function change(event) {
  const files = [...event.target.files]
  for (const file of files) {
    processFile(file)
    break
  }
  event.target.value = ''
}

graphviz.loadWASM().then(() => {
  wasmLoaded.value = true
})
</script>

<style>
@tailwind base;
@tailwind components;
@tailwind utilities;
</style>
