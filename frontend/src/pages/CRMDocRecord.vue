<template>
  <LayoutHeader v-if="isNew || crmDoc.data">
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
  </LayoutHeader>

  <div v-if="isNew || crmDoc.data" class="flex h-full overflow-hidden">
    <div class="flex-1 p-4 overflow-y-auto">

      <DataFields
        doctype="CRM Doc"
        :docname="isNew ? null : crmDoc.data.name"
        :editMode="isNew || isEditMode"
        @afterSave="() => crmDoc.reload()"
      />

    </div>
  </div>

  <ErrorPage
    v-else-if="errorTitle"
    :errorTitle="errorTitle"
    :errorMessage="errorMessage"
  />

  <FilesUploader
    v-if="crmDoc.data?.name"
    v-model="showFilesUploader"
    doctype="CRM Doc"
    :docname="crmDoc.data.name"
    @after="
      () => {
        activities?.all_activities?.reload()
        changeTabTo('attachments')
      }
    "
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

import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'

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

const errorTitle = ref('')
const errorMessage = ref('')

/* ---------------------------
   MODE DETECTION
----------------------------*/
const isNew = computed(() => props.docId === 'new')
const isEditMode = computed(() => route.query.mode === 'edit')

/* ---------------------------
   DOC RESOURCE
----------------------------*/
const crmDoc = createResource({
  url: 'crm.fcrm.doctype.crm_doc.api.get_doc',

  params: () => ({
    name: isNew.value ? '' : props.docId,
  }),

  cache: () => ['crmDoc', props.docId],

  onSuccess: (data) => {
    errorTitle.value = ''
    errorMessage.value = ''

    setupCustomizations(crmDoc, {
      doc: data,
      $dialog,
      $socket,
      router,
      toast,
      updateField,
      createToast: toast.create,
      deleteDoc,
      resource: { crmDoc },
      call,
    })
  },

  onError: (err) => {
    errorTitle.value = __('Not permitted')
    errorMessage.value = __(err.messages?.[0] || '')
  },
})

/* ---------------------------
   LIFECYCLE
----------------------------*/
onMounted(() => {
  console.log("Route docId:", props.docId)

  if (!isNew.value) {
    crmDoc.fetch()
  }
})

/* ---------------------------
   UPDATE FIELD
----------------------------*/
function updateDoc(fieldname, value, callback) {
  createResource({
    url: 'frappe.client.set_value',
    params: {
      doctype: 'CRM Doc',
      name: props.docId,
      fieldname,
      value,
    },
    auto: true,
    onSuccess: () => {
      crmDoc.reload()
      toast.success(__('Doc updated'))
      callback?.()
    },
    onError: (err) => {
      toast.error(__('Error updating doc: {0}', [err.messages?.[0]]))
    },
  })
}

function updateField(name, value, callback) {
  updateDoc(name, value, () => {
    if (crmDoc.data) crmDoc.data[name] = value
    callback?.()
  })
}

/* ---------------------------
   DELETE
----------------------------*/
async function deleteDoc(name) {
  await call('frappe.client.delete', {
    doctype: 'CRM Doc',
    name,
  })
  router.push({ name: 'CRM Doc' })
}

/* ---------------------------
   RESET (NEW MODE)
----------------------------*/
function resetDoc() {
  crmDoc.data = {
    item: [],
    address: '',
    city: '',
    id: '',
    name1: '',
    type1: '',
  }
}

/* ---------------------------
   BREADCRUMBS
----------------------------*/
const breadcrumbs = computed(() => {
  let items = [
    {
      label: __('CRM Doc'),
      route: { name: 'CRM Doc' }
    }
  ]

  items.push({
    label: isNew.value ? __('New') : (crmDoc.data?.name || props.docId),
    route: {
      name: 'CRMDocID',
      params: {
        docId: isNew.value ? 'new' : crmDoc.data?.name
      }
    }
  })

  return items
})

/* ---------------------------
   TITLE
----------------------------*/
const title = computed(() => {
  let t = doctypeMeta['CRM Doc']?.title_field || 'name'
  return crmDoc.data?.[t] || props.docId
})

usePageMeta(() => ({
  title: title.value,
  icon: brand.favicon,
}))
</script>
