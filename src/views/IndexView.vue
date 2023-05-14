<template>
  <v-container class="bg-surface-variant h-screen" fluid>
    <div class="h-100" style="display: flex; flex-direction: column">
      <v-row style="flex: 0 1 auto">
        <v-col cols="6"><h2>PiTOGo</h2></v-col>
        <v-col cols="6">
          <div>
            <v-tooltip text="Reset to initial example" location="left" open-delay="300">
              <template v-slot:activator="{ props }">
                <v-btn @click="resetCode" class="mr-2" v-bind="props">Reset</v-btn>
              </template>
            </v-tooltip>

            <v-tooltip
              text="Copy the transpiled code in your clipboard"
              location="right"
              open-delay="300"
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
            :value="result"
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
          Built with ❤️ by Andrea Simone Entroterra & Alessandro Ascensore
        </div>
      </v-row>
    </div>
  </v-container>
</template>

<script setup lang="ts">
import * as pitogo from 'pitogo'
import { computed, nextTick, ref, type Ref } from 'vue'
import { VAceEditor } from 'vue3-ace-editor'
import { useStorage } from '@vueuse/core'
import ace from 'ace-builds'
import { onMounted } from 'vue'

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

const defaultCode = `
  A = log<123>;
  C(x) = x<"a">;
  main = (as)A + C<as>;
`

const code = useStorage('pitogo-code', defaultCode)

const resetCode = (): void => {
  code.value = defaultCode
}

let marker: number | undefined
const result = computed(() => {
  try {
    marker != undefined && pi?.value?.getSession().removeMarker(marker)
    pi.value?.getSession().setAnnotations([])
    return pitogo.T.transpileToGo(pitogo.P.parse(pitogo.S.scanner(code.value)))
  } catch (e: any) {
    // e has type: {
    //  message: string
    //  position: { row_start: number; row_end: number; column_start: number; column_end: number }
    // }

    console.log({ error: e })

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

    const textToUnderline = pi.value?.getSession().getDocument().getTextRange(underlineErrorRange)

    // eslint-disable-next-line vue/no-async-in-computed-properties
    setTimeout(() => {
      const errorMarkerDiv = document.querySelector('.error-marker')
      if (errorMarkerDiv) {
        ;(errorMarkerDiv as any).innerText = textToUnderline
      } else {
        setTimeout(() => {
          const errorMarkerDiv = document.querySelector('.error-marker')
          if (errorMarkerDiv) {
            ;(errorMarkerDiv as any).innerText = textToUnderline
          } else {
            setTimeout(() => {
              const errorMarkerDiv = document.querySelector('.error-marker')
              if (errorMarkerDiv) {
                ;(errorMarkerDiv as any).innerText = textToUnderline
              } else {
                setTimeout(() => {
                  const errorMarkerDiv = document.querySelector('.error-marker')
                  if (errorMarkerDiv) {
                    ;(errorMarkerDiv as any).innerText = textToUnderline
                  } else {
                    setTimeout(() => {
                      const errorMarkerDiv = document.querySelector('.error-marker')
                      if (errorMarkerDiv) {
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

function copyCode() {
  navigator.clipboard.writeText(result.value)
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
</style>
