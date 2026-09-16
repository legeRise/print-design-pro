<script setup lang="ts">
import type { TemplateDocument } from '@pro-print/editor'
import { downloadBlob, exportPdf, exportPng, exportTemplate, importTemplate, PrintDesigner } from '@pro-print/editor'
import '@pro-print/editor/style.css'

// Editor host page: loads from IndexedDB, autosaves debounced snapshots the
// editor emits, offers JSON export/import. Client-only (ssr disabled via
// routeRules) - the editor is a pure canvas app.
definePageMeta({ layout: false })

const route = useRoute()
const repo = useTemplateRepository()
const toast = useToast()
const { locale } = useAppLocale()

const template = ref<TemplateDocument | null>(null)
const notFound = ref(false)
const saving = ref(false)
const isDraggingFile = ref(false)
let dragDepth = 0

async function load(id: string): Promise<void> {
  const record = await repo.get(id)
  if (record) {
    template.value = record.doc
    notFound.value = false
  }
  else {
    notFound.value = true
  }
}

onMounted(() => load(route.params.id as string))
// Nuxt reuses the page component when only the param changes.
watch(() => route.params.id, id => load(id as string))

// Autosave: the editor already debounces its snapshots (400ms); a short
// second debounce here batches bursts of snapshots into one IndexedDB write.
const AUTOSAVE_MS = 800
let saveTimer: ReturnType<typeof setTimeout> | null = null

async function persist(doc: TemplateDocument): Promise<void> {
  try {
    await repo.save(doc)
  }
  catch (error) {
    toast.add({
      title: 'Autosave failed — your changes are NOT saved',
      description: error instanceof Error ? error.message.slice(0, 200) : 'Storage error',
      color: 'error',
    })
  }
  finally {
    saving.value = false
  }
}

function onDocumentUpdate(doc: TemplateDocument): void {
  template.value = doc
  if (saveTimer)
    clearTimeout(saveTimer)
  saving.value = true
  saveTimer = setTimeout(() => {
    saveTimer = null
    void persist(doc)
  }, AUTOSAVE_MS)
}

// Browser close / hard reload: flush any pending write best-effort (the SPA
// navigation path is already covered by the editor's unmount flush).
function flushPending(): void {
  if (saveTimer && template.value) {
    clearTimeout(saveTimer)
    saveTimer = null
    void persist(template.value)
  }
}
onMounted(() => window.addEventListener('pagehide', flushPending))
onBeforeUnmount(() => {
  window.removeEventListener('pagehide', flushPending)
  flushPending()
})

function baseFilename(): string {
  return template.value?.name.trim() || 'template'
}

async function exportJson(): Promise<void> {
  if (!template.value)
    return

  const json = exportTemplate(template.value)

  // Keep the existing download behaviour, but also put the exact same JSON
  // into the clipboard so it can immediately be pasted into an issue, editor,
  // API request, etc.
  downloadBlob(
    new Blob([json], { type: 'application/json' }),
    `${baseFilename()}.json`,
  )

  try {
    await navigator.clipboard.writeText(json)
    toast.add({ title: 'Template JSON downloaded and copied', color: 'success' })
  }
  catch {
    // Clipboard permissions can be unavailable in some browser contexts. The
    // download has already succeeded, so don't report the whole export as a failure.
    toast.add({ title: 'Template JSON downloaded', description: 'Could not copy JSON to clipboard.', color: 'warning' })
  }
}

const exporting = ref(false)

async function exportFile(kind: 'png' | 'pdf'): Promise<void> {
  if (!template.value || exporting.value)
    return
  exporting.value = true
  try {
    const blob = kind === 'png'
      ? await exportPng(template.value)
      : await exportPdf(template.value)
    downloadBlob(blob, `${baseFilename()}.${kind}`)
  }
  catch (error) {
    toast.add({
      title: `Export ${kind.toUpperCase()} failed`,
      description: error instanceof Error ? error.message.slice(0, 200) : 'Unknown error',
      color: 'error',
    })
  }
  finally {
    exporting.value = false
  }
}

async function importJsonFile(file: File): Promise<void> {
  if (!template.value)
    return

  if (file.type && file.type !== 'application/json' && !file.name.toLowerCase().endsWith('.json')) {
    toast.add({ title: 'Import failed', description: 'Please drop a JSON template file.', color: 'error' })
    return
  }

  try {
    const imported = importTemplate(await file.text())
    // Keep the current record identity so the import lands in this template.
    imported.id = template.value.id
    // Drop any pending autosave of the pre-import document - it would
    // otherwise overwrite the import moments later.
    if (saveTimer) {
      clearTimeout(saveTimer)
      saveTimer = null
    }
    template.value = imported
    await repo.save(imported)
    toast.add({ title: 'Template imported', color: 'success' })
  }
  catch (error) {
    toast.add({
      title: 'Import failed',
      description: error instanceof Error ? error.message.slice(0, 300) : 'Invalid file',
      color: 'error',
    })
  }
}

async function importJson(event: Event): Promise<void> {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (file)
    await importJsonFile(file)
}

function hasFiles(event: DragEvent): boolean {
  return Array.from(event.dataTransfer?.types ?? []).includes('Files')
}

function onDragEnter(event: DragEvent): void {
  if (!hasFiles(event))
    return
  event.preventDefault()
  dragDepth++
  isDraggingFile.value = true
}

function onDragOver(event: DragEvent): void {
  if (!hasFiles(event))
    return
  event.preventDefault()
  event.dataTransfer!.dropEffect = 'copy'
}

function onDragLeave(event: DragEvent): void {
  if (!hasFiles(event))
    return
  event.preventDefault()
  dragDepth = Math.max(0, dragDepth - 1)
  if (dragDepth === 0)
    isDraggingFile.value = false
}

async function onDrop(event: DragEvent): Promise<void> {
  if (!hasFiles(event))
    return
  event.preventDefault()
  event.stopPropagation()
  dragDepth = 0
  isDraggingFile.value = false

  const file = event.dataTransfer?.files?.[0]
  if (file)
    await importJsonFile(file)
}
</script>

<template>
  <div
    class="relative h-screen"
    @dragenter="onDragEnter"
    @dragover="onDragOver"
    @dragleave="onDragLeave"
    @drop="onDrop"
  >
    <div
      v-if="notFound"
      class="flex h-full flex-col items-center justify-center gap-4"
    >
      <p class="text-lg font-medium">
        Template not found
      </p>
      <UButton to="/templates">
        Back to templates
      </UButton>
    </div>

    <ClientOnly v-else>
      <PrintDesigner
        v-if="template"
        :model-value="template"
        :saving="saving"
        :locale="locale"
        class="h-full"
        @update:model-value="onDocumentUpdate"
        @home="navigateTo('/templates')"
      >
        <template #actions>
          <UDropdownMenu
            :items="[
              { label: 'PNG (300 DPI)', icon: 'i-lucide-image', onSelect: () => exportFile('png') },
              { label: 'PDF', icon: 'i-lucide-file-text', onSelect: () => exportFile('pdf') },
              { label: 'JSON (download + copy)', icon: 'i-lucide-braces', onSelect: () => exportJson() },
            ]"
          >
            <UButton
              size="sm"
              variant="soft"
              icon="i-lucide-download"
              :loading="exporting"
              data-test-export
            >
              Export
            </UButton>
          </UDropdownMenu>
          <label>
            <UButton
              size="sm"
              variant="soft"
              icon="i-lucide-upload"
              as="span"
            >
              Import
            </UButton>
            <input
              type="file"
              accept=".json,application/json"
              class="hidden"
              @change="importJson"
            >
          </label>
        </template>
      </PrintDesigner>

      <div
        v-else
        class="flex h-full items-center justify-center text-muted"
      >
        Loading editor…
      </div>
    </ClientOnly>

    <div
      v-if="isDraggingFile"
      class="pointer-events-none absolute inset-3 z-50 flex items-center justify-center rounded-2xl border-2 border-dashed border-accent-500 bg-accent-soft/80 backdrop-blur-[2px]"
    >
      <div class="rounded-xl border border-accent-500 bg-app-panel px-6 py-4 text-center shadow-lg">
        <div class="text-sm font-semibold">
          Drop template JSON here
        </div>
        <div class="mt-1 text-xs text-app-text2">
          The dropped template will replace the current design
        </div>
      </div>
    </div>
  </div>
</template>
