<template>
  <!-- Header -->
  <LayoutHeader v-if="isNewDoc || crmDoc.data">
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
  </LayoutHeader>

  <!-- Main -->
  <div v-if="isNewDoc || crmDoc.data" class="flex h-full overflow-hidden flex-col">

    <!-- New label -->
    <div v-if="isNewDoc" class="p-4 text-lg font-semibold text-gray-600">
      New Document
    </div>

    <div class="flex flex-1 overflow-hidden">
      <div class="flex-1 p-4 overflow-y-auto">

        <!-- 🔥 KEY FIX HERE -->
        <DataFields
          doctype="CRM Doc"
          :docname="isNewDoc ? 'new' : crmDoc.data.name"
          @afterSave="handleAfterSave"
        />

      </div>
    </div>
  </div>

  <!-- Error -->
  <ErrorPage
    v-else-if="errorTitle"
    :errorTitle="errorTitle"
    :errorMessage="errorMessage"
  />

  <!-- Files uploader (only for existing docs) -->
  <FilesUploader
    v-if="!isNewDoc && crmDoc.data?.name"
    v-model="showFilesUploader"
    doctype="CRM Doc"
    :docname="crmDoc.data.name"
  />
</template>

<script setup>
import ErrorPage from '@/components/ErrorPage.vue'
import Icon from '@/components/Icon.vue'
import DataFields from '@/components/Activities/DataFields.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import FilesUploader from '@/components/FilesUploader/FilesUploader.vue'

import { getView } from '@/utils/view'
import { getSettings } from '@/stores/settings'
import { globalStore } from '@/stores/global'
import { getMeta } from '@/stores/meta'

import {
  createResource,
  Breadcrumbs,
  call,
  usePageMeta,
  toast,
} from 'frappe-ui'

import { useOnboarding } from 'frappe-ui/frappe'
import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'

/* ---------------------- Setup ---------------------- */

const { brand } = getSettings()
const { $dialog, $socket } = globalStore()
const { doctypeMeta } = getMeta('CRM Doc')

const route = useRoute()
const router = useRouter()

const props = defineProps({
  docId: {
    type: String,
    required: true,
  },
})

const isNewDoc = computed(() => props.docId === 'new')

const errorTitle = ref('')
const errorMessage = ref('')
const showFilesUploader = ref(false)

/* ---------------------- Existing Doc Resource ---------------------- */

const crmDoc = createResource({
  url: 'crm.fcrm.doctype.crm_doc.api.get_doc',
  params: { name: props.docId },
  cache: ['crmDoc', props.docId],

  onSuccess: (data) => {
    errorTitle.value = ''
    errorMessage.value = ''
  },

  onError: (err) => {
    if (err.messages?.[0]) {
      errorTitle.value = __('Not permitted')
      errorMessage.value = __(err.messages?.[0])
    } else {
      router.push({ name: 'CRM Doc' })
    }
  },
})

/* ---------------------- Lifecycle ---------------------- */

onMounted(() => {
  // Only fetch if NOT new
  if (!isNewDoc.value) {
    crmDoc.fetch()
  }
})

/* ---------------------- Breadcrumbs ---------------------- */

const breadcrumbs = computed(() => {
  let items = [
    {
      label: __('CRM Doc'),
      route: { name: 'CRM Doc' },
    },
  ]

  if (route.query.view || route.query.viewType) {
    let view = getView(route.query.view, route.query.viewType, 'CRM Doc')
    if (view) {
      items.push({
        label: __(view.label),
        icon: view.icon,
        route: {
          name: 'CRM Doc',
          params: { viewType: route.query.viewType },
          query: { view: route.query.view },
        },
      })
    }
  }

  items.push({
    label: isNewDoc.value ? 'New CRM Doc' : title.value,
    route: isNewDoc.value
      ? undefined
      : {
          name: 'CRMDocID',
          params: { docId: crmDoc.data?.name },
        },
  })

  return items
})

/* ---------------------- Title ---------------------- */

const title = computed(() => {
  if (isNewDoc.value) return 'New CRM Doc'

  let t = doctypeMeta['CRM Doc']?.title_field || 'name'
  return crmDoc.data?.[t] || props.docId
})

usePageMeta(() => {
  return {
    title: title.value,
    icon: brand.favicon,
  }
})

/* ---------------------- After Save ---------------------- */

function handleAfterSave() {
  // Existing doc → reload
  if (!isNewDoc.value) {
    crmDoc.reload()
  } else {
    // New doc → you will implement later
    console.log('New doc saved (handle later)')
  }
}
</script>
