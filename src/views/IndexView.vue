<template>
  <v-container class="bg-surface-variant h-screen" fluid>
    <v-overlay :model-value="isAwaitRunning" class="align-center justify-center" persistent>
      <v-progress-circular color="white" indeterminate size="64"></v-progress-circular>
    </v-overlay>

    <div class="h-100" style="display: flex; flex-direction: column">
      <v-row style="flex: 0 1 auto">
        <v-col cols="6"><h2>PiTOGo</h2></v-col>
        <v-col cols="6">
          <div>
            <v-tooltip text="Reset to initial example" location="left" open-delay="500">
              <template v-slot:activator="{ props }">
                <v-btn @click="resetCode" class="mr-2" v-bind="props">Reset</v-btn>
              </template>
            </v-tooltip>

            <v-tooltip text="Run the transpiled code" location="bottom" open-delay="500">
              <template v-slot:activator="{ props }">
                <v-btn @click="runCode" class="mr-2" v-bind="props" :disabled="isAwaitRunning"
                  >Run</v-btn
                >
              </template>
            </v-tooltip>

            <v-tooltip
              text="Copy the transpiled code in your clipboard"
              location="right"
              open-delay="500"
            >
              <template v-slot:activator="{ props }">
                <v-btn @click="copyCode" v-bind="props">Copy</v-btn>
              </template>
            </v-tooltip>
          </div>
        </v-col>
      </v-row>
      <v-row style="flex: 1 1 auto">
        <v-col cols="6">
          <v-ace-editor
            id="pi"
            @init="() => {}"
            v-model:value="code"
            theme="chrome"
            class="h-100"
          />
          <span>row: {{ cursorPosition.row }}, column: {{ cursorPosition.column }}</span>
        </v-col>

        <v-col cols="6">
          <v-ace-editor
            :value="formattedResult"
            theme="chrome"
            @init="() => {}"
            lang="golang"
            class="h-100"
            readonly
          />
        </v-col>
      </v-row>

      <v-row style="flex: 0 1 auto">
        <div style="width: 100%; text-align: center">
          Built with ❤️ by
          <a target="_blank" class="" href="https://andreasimonecosta.dev/"
            >Andrea Simone Entroterra</a
          >
          &
          <a target="_blank" href="https://www.linkedin.com/in/alessandro-scala/"
            >Alessandro Ascensore</a
          >
        </div>
      </v-row>
    </div>
    <v-dialog width="auto" v-model="isResultDialogOpen">
      <v-card>
        <v-card-title>
          <v-row>
            <v-col class="d-flex justify-end">
              <v-btn
                flat
                density="compact"
                :icon="mdiClose"
                @click="isResultDialogOpen = false"
              ></v-btn>
            </v-col>
          </v-row>
        </v-card-title>

        <v-card-text>
          <pre>{{ runResult.trim() }}</pre>
        </v-card-text>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script setup lang="ts">
import * as pitogo from 'pitogo'
import { computed, ref } from 'vue'
import { VAceEditor } from 'vue3-ace-editor'
import { useStorage } from '@vueuse/core'
import ace from 'ace-builds'
import { onMounted } from 'vue'
import { watchEffect, watch } from 'vue'
import debounce from 'lodash.debounce'
import { mdiClose } from '@mdi/js'

const pi = ref<ace.Ace.Editor>()

const cursorPosition = useStorage('pitogo-position', { row: 1, column: 1 })
const annotations = ref<ace.Ace.Annotation[]>([])
onMounted(() => {
  pi.value = ace.edit('pi')

  pi.value.moveCursorToPosition(cursorPosition.value)
  pi.value.getSession().setAnnotations(annotations.value)

  pi.value.getSession().selection.on('changeCursor', () => {
    const cp = pi.value?.getCursorPosition()
    cursorPosition.value = { row: (cp?.row ?? 0) + 1, column: (cp?.column ?? 0) + 1 }
  })
})

const defaultCode = `A(c) = c(a).log<a>;
B(c) = c(b).log<b>;
main = (ch1)(ch2) A<ch1> | B<ch2> | (ch1<"ma"> + ch2<"mb"> + log<"ciao">);
`

const code = useStorage('pitogo-code', defaultCode)

const resetCode = (): void => {
  code.value = defaultCode
}

async function gofmtr(source: string): Promise<string> {
  // const formData = new FormData()
  // URL
  // formData.append('body', JSON.stringify(source))
  // formData.append('imports', 'true')

  return fetch('https://corsproxy.io/?https://go.dev/_/fmt?backend=', {
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
      'X-Requested-With': 'XMLHttpRequest'
    },
    method: 'post',
    body: new URLSearchParams({
      body: source,
      imports: 'true'
    })
  }).then(async (res) => {
    return (await res.json())['Body']
  })
}

let marker: number | undefined
let prevTimeout: any
const result = computed(() => {
  try {
    prevTimeout && clearTimeout(prevTimeout)

    marker != undefined && pi?.value?.getSession().removeMarker(marker)
    pi.value?.getSession().setAnnotations([])
    const parseResult = pitogo.P.parse(pitogo.S.scanner(code.value))
    pitogo.T.isRecursionGuarded(parseResult)

    const res = pitogo.T.transpileToGo(parseResult)

    return res
  } catch (e: any) {
    // e has type: {
    //  message: string
    //  position: { row_start: number; row_end: number; column_start: number; column_end: number }
    // }

    pi.value?.getSession().setAnnotations([
      {
        row: e.position.row_start - 1,
        column: e.position.column_start - 1,
        text: e.message,
        type: 'error'
      }
    ])

    const underlineErrorRange = new ace.Range(
      e.position.row_start - 1,
      e.position.column_start - 1,
      e.position.row_end - 1,
      e.position.column_end
    )

    marker = pi.value?.getSession().addMarker(underlineErrorRange, 'error-marker', 'text')

    // eslint-disable-next-line vue/no-async-in-computed-properties
    prevTimeout = setTimeout(() => {
      const errorMarkerDiv = document.querySelector('.error-marker')
      if (errorMarkerDiv) {
        const textToUnderline = pi.value
          ?.getSession()
          .getDocument()
          .getTextRange(underlineErrorRange)
        ;(errorMarkerDiv as any).innerText = textToUnderline
      } else {
        prevTimeout = setTimeout(() => {
          const errorMarkerDiv = document.querySelector('.error-marker')
          if (errorMarkerDiv) {
            const textToUnderline = pi.value
              ?.getSession()
              .getDocument()
              .getTextRange(underlineErrorRange)
            ;(errorMarkerDiv as any).innerText = textToUnderline
          } else {
            prevTimeout = setTimeout(() => {
              const errorMarkerDiv = document.querySelector('.error-marker')
              if (errorMarkerDiv) {
                const textToUnderline = pi.value
                  ?.getSession()
                  .getDocument()
                  .getTextRange(underlineErrorRange)
                ;(errorMarkerDiv as any).innerText = textToUnderline
              } else {
                prevTimeout = setTimeout(() => {
                  const errorMarkerDiv = document.querySelector('.error-marker')
                  if (errorMarkerDiv) {
                    const textToUnderline = pi.value
                      ?.getSession()
                      .getDocument()
                      .getTextRange(underlineErrorRange)
                    ;(errorMarkerDiv as any).innerText = textToUnderline
                  } else {
                    prevTimeout = setTimeout(() => {
                      const errorMarkerDiv = document.querySelector('.error-marker')
                      if (errorMarkerDiv) {
                        const textToUnderline = pi.value
                          ?.getSession()
                          .getDocument()
                          .getTextRange(underlineErrorRange)
                        ;(errorMarkerDiv as any).innerText = textToUnderline
                      }
                    }, 5)
                  }
                }, 5)
              }
            }, 5)
          }
        }, 5)
      }
    }, 5)

    return ''
  }
})

const formattedResult = ref(result.value)
watch(
  result,
  debounce(async () => {
    const fmtstr = await gofmtr(result.value)
    formattedResult.value = fmtstr
  }, 500),
  { immediate: true }
)

function copyCode() {
  navigator.clipboard.writeText(result.value)
}

const runResult = ref('')
const isResultDialogOpen = ref(false)
const isAwaitRunning = ref(false)

async function runCode() {
  isAwaitRunning.value = true
  return await fetch('https://corsproxy.io/?https://go.dev/_/compile?backend=', {
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
      'X-Requested-With': 'XMLHttpRequest'
    },
    method: 'post',
    body: new URLSearchParams({
      body: formattedResult.value,
      imports: 'true'
    })
  })
    .then(async (res) => {
      const runRes: { compile_errors: string; output: string } = await res.json()
      if (runRes.compile_errors !== '') {
        runResult.value = `Go Compiler Errors:
      ${runRes.compile_errors}`
      } else {
        const ind = runRes.output.indexOf('fatal error: all goroutines are asleep - deadlock!')
        if (ind >= 0) {
          runResult.value = runRes.output.substring(0, ind) + '\nDeadlock Reached!'
        } else {
          runResult.value = runRes.output
        }
      }
      isResultDialogOpen.value = true
    })
    .finally(() => {
      isAwaitRunning.value = false
    })
}
</script>

<style>
.error-marker {
  position: absolute !important;
  text-decoration-color: red !important;
  /* background-color: red !important; */
  color: red !important;
  text-decoration: underline !important;
  text-decoration-style: wavy !important;
}

a {
  color: #eeeeee;
  text-decoration: none;
}

a:hover {
  color: #abcdef;
}
</style>
