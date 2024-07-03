<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'
import Versions from './components/Versions.vue'
import * as Diff2html from 'diff2html'
import * as Diff from 'diff'
import 'diff2html/bundles/css/diff2html.min.css'

const repoPath = ref('')
const leftBranch = ref('')
const rightBranch = ref('')
const diffFiles = ref([])
const diffByFile = ref({})
const selectedDiffFileName = ref('')
const diffRefs = ref({})

watch(repoPath, (value) => {
  localStorage.setItem('repoPath', value)
})

watch(leftBranch, (value) => {
  localStorage.setItem('leftBranch', value)
})

watch(rightBranch, (value) => {
  localStorage.setItem('rightBranch', value)
})

async function compareBranches() {
  const response = await window.electron.ipcRenderer.invoke('compareBranches', {
    repoPath: repoPath.value,
    leftBranch: leftBranch.value,
    rightBranch: rightBranch.value
  })

  const diffByFileData = {}

  diffFiles.value = response.diff.map((diffItem) => {
    const oldFileName = diffItem.oldFileName.replace(/^a\//, '')
    const newFileName = diffItem.newFileName.replace(/^b\//, '')

    const fileName = oldFileName === newFileName ? oldFileName : `${oldFileName} -> ${newFileName}`

    diffByFileData[fileName] = getDiff(diffItem)

    return {
      fileName,
    }
  })

  diffByFile.value = diffByFileData
}

function getDiff(diff) {
  const unifiedDiff = Diff.formatPatch(diff)
  const diffJson = Diff2html.parse(unifiedDiff)
  return Diff2html.html(diffJson, {
    colorScheme: 'dark',
    highlight: true,
  })
}

function focusDiff(fileName) {
  selectedDiffFileName.value = fileName

  diffRefs.value[fileName].scrollIntoView({
    behavior: 'smooth',
    block: 'start',
  })
}

onMounted(() => {
  repoPath.value = localStorage.getItem('repoPath') || ''
  leftBranch.value = localStorage.getItem('leftBranch') || ''
  rightBranch.value = localStorage.getItem('rightBranch') || ''
})
</script>

<template>
  <div style="display: grid; grid-template-rows: auto auto 1fr; padding: 1.5rem; overflow: auto; height: 100%;">
    <div>
      <label>
        <div>Repo Path</div>
        <input type="text" v-model="repoPath" class="full-width input" spellcheck="false">
      </label>
    </div>
    <div style="display: flex; align-items: flex-end; gap: 1rem; margin-top: 0.5rem;">
      <label>
        <div>Left Branch</div>
        <input type="text" v-model="leftBranch" class="full-width input" spellcheck="false">
      </label>
      <label>
        <div>Right Branch</div>
        <input type="text" v-model="rightBranch" class="full-width input" spellcheck="false">
      </label>
      <button @click="compareBranches">Compare</button>
    </div>
    <div style="overflow: auto; margin-top: 1rem; display: grid; grid-template-columns: 200px 1fr;">
      <div style="overflow: auto;">
          <div v-for="diffFile in diffFiles" @click="focusDiff(diffFile.fileName)" class="sidebar-item" :class="{ active: selectedDiffFileName === diffFile.fileName }">
            {{ diffFile.fileName }}
          </div>
      </div>
      <div style="overflow: auto;">
        <div :ref="element => { diffRefs[fileName] = element }" v-html="diffByFile[fileName]" v-for="fileName of Object.keys(diffByFile)"></div>
      </div>
    </div>
  </div>
  <!-- <Versions /> -->
</template>
