<template>
  <div class="page-contents">
    <!-- Following hidden input is hacky way to update monaco editor theme when user changes theme manually -->
    <input type="hidden" inert :value="onThemeChange" />
    <Navbar :show-back-button="true" />
    <main class="outline-none" tabindex="0">
      <DiffActionBar :diff-navigator="diffNavigator" />
      <section
        class="flex flex-wrap items-stretch w-full gap-4 font-mono text-gray-800  dark:text-gray-50"
      >
        <div class="flex w-full gap-4 space-around">
          <p
            class="flex-grow-0 flex-shrink-0 w-1/2 text-lg font-bold text-center capitalize break-all "
          >
            <span class="inline-block w-4/5">{{ lhsLabel }}</span>
          </p>
          <p
            class="flex-grow-0 flex-shrink-0 w-1/2 text-lg font-bold text-center capitalize break-all "
          >
            <span class="inline-block w-4/5">{{ rhsLabel }}</span>
          </p>
        </div>
        <div
          id="monaco-diff-viewer"
          class="w-full h-screen p-2 border border-gray-600 rounded-md editor"
        ></div>
      </section>
    </main>
  </div>
</template>

<script lang="ts">
import loader from '@monaco-editor/loader'
import pako from 'pako'
import Vue from 'vue'
import {
  getMonacoEditorDefaultOptions,
  undoUrlSafeBase64,
} from '../../helpers/utils'
import DiffActionBar from '~/components/v2/diffActionBar.vue'
import Navbar from '~/components/v2/navbar.vue'
import { v2DiffData } from '~/helpers/types'
export default Vue.extend({
  components: { DiffActionBar, Navbar },
  layout: 'main',
  data(): v2DiffData {
    return {
      lhs: '',
      rhs: '',
      rhsLabel: '',
      lhsLabel: '',
      monacoDiffEditor: {},
      diffNavigator: {},
    }
  },
  head() {
    return {
      title: 'Diff Viewer - Diff view',
    }
  },
  computed: {
    onThemeChange() {
      const theme = this.$store.state.theme.darkMode ? 'vs-dark' : 'light'
      this.monacoDiffEditor?.updateOptions?.({ theme })
      return this.$store.state.theme.darkMode
    },
  },
  beforeMount() {
    const _diff = this.$route.hash
    if (_diff) {
      const gunzip = pako.ungzip(
        Buffer.from(undoUrlSafeBase64(_diff), 'base64')
      )
      const diffData = JSON.parse(Buffer.from(gunzip).toString('utf8'))
      const { lhs, rhs, lhsLabel, rhsLabel } = diffData
      this.lhsLabel = lhsLabel
      this.rhsLabel = rhsLabel
      this.lhs = lhs
      this.rhs = rhs
    }
  },
  mounted() {
    const monacoDiffViewerEl = document.getElementById('monaco-diff-viewer')
    const theme = this.$cookies.isDarkMode ? 'vs-dark' : 'light'
    const monacoEditorOptions = getMonacoEditorDefaultOptions(theme)
    loader.init().then((monaco) => {
      if (monacoDiffViewerEl) {
        this.monacoDiffEditor = monaco.editor.createDiffEditor(
          monacoDiffViewerEl,
          {
            ...monacoEditorOptions,
            readOnly: true,
            wordWrap: 'on',
            diffAlgorithm: 'advanced',
          }
        ) as any
        if (this.monacoDiffEditor) {
          this.monacoDiffEditor.setModel({
            original: monaco.editor.createModel(this.lhs, 'javascript'),
            modified: monaco.editor.createModel(this.rhs, 'javascript'),
          })
          this.diffNavigator = monaco.editor.createDiffNavigator(
            this.monacoDiffEditor,
            {
              followsCaret: true,
              ignoreCharChanges: true,
              alwaysRevealFirst: true,
            }
          )
        }
        this.$store.commit('data/set', {
          lhs: this.lhs,
          rhs: this.rhs,
          lhsLabel: this.lhsLabel,
          rhsLabel: this.rhsLabel,
        })
      }
    })
  },
})
</script>

<style>
.editor {
  max-height: calc(100vh - 15rem);
}
</style>
