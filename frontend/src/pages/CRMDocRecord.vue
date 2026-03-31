<template>
  <LayoutHeader>
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
  </LayoutHeader>

  <!-- ALWAYS SHOW FORM (NEW + EDIT) -->
  <div class="flex h-full overflow-hidden">
    <div class="flex-1 p-4 overflow-y-auto">

      <DataFields
        doctype="CRM Doc"
        :docname="crmDoc.data?.name"
        :editMode="true"
        @afterSave="handleAfterSave"
      />

    </div>
  </div>

  <!-- ERROR -->
  <ErrorPage
    v-if="errorTitle"
    :errorTitle="errorTitle"
    :errorMessage="errorMessage"
  />

  <!-- FILES -->
  <FilesUploader
    v-if="crmDoc.data?.name"
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

import { ref, computed, onMounted, watch } from 'vue'
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
const showFilesUploader = ref(false)

/* ---------------------------
   MODE
----------------------------*/
const isNew = computed(() => props.docId === 'new')

/* ---------------------------
   DOC STATE
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

    crmDoc.data = data
  },

  onError: (err) => {
    errorTitle.value = 'Not permitted'
    errorMessage.value = err.messages?.[0] || ''
  },
})

/* ---------------------------
   INIT
----------------------------*/
onMounted(() => {
  if (isNew.value) {
    crmDoc.data = {
      item: [],
      address: '',
      city: '',
      id: '',
      name1: '',
      type1: '',
    }
  } else {
    crmDoc.fetch()
  }
})

/* ---------------------------
   SAVE HANDLER (NEW + UPDATE)
----------------------------*/
async function handleAfterSave(payload) {
  try {
    // CREATE NEW
    if (isNew.value) {
      const res = await call('frappe.client.insert', {
        doc: {
          doctype: 'CRM Doc',
          ...crmDoc.data,
        }
      })

      const newName = res.name

      toast.success('Created successfully')

      router.replace({
        name: 'CRMDocID',
        params: { docId: newName },
        query: { mode: 'edit' }
      })

      crmDoc.data.name = newName
    }

    // UPDATE EXISTING
    else {
      await call('frappe.client.set_value', {
        doctype: 'CRM Doc',
        name: props.docId,
        fieldname: payload.fieldname,
        value: payload.value,
      })

      toast.success('Updated')
    }

  } catch (err) {
    toast.error(err.message || 'Save failed')
  }
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
   BREADCRUMBS
----------------------------*/
const breadcrumbs = computed(() => {
  return [
    {
      label: __('CRM Doc'),
      route: { name: 'CRM Doc' }
    },
    {
      label: crmDoc.data?.name || __('New'),
      route: {
        name: 'CRMDocID',
        params: {
          docId: crmDoc.data?.name || 'new'
        }
      }
    }
  ]
})

/* ---------------------------
   PAGE TITLE
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
