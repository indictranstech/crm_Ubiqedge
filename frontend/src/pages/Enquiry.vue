<template>
  <LayoutHeader>
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
    <template v-if="!errorTitle" #right-header>
      <CustomActions
        v-if="document._actions?.length"
        :actions="document._actions"
      />
      <AssignTo
        v-model="assignees.data"
        doctype="Enquiry Form"
        :docname="enquiryId"
      />
<Dropdown
  v-if="workflowTransitions.length && !doc.converted_deal && !doc.converted_lead"
  :options="workflowDropdownOptions"
  placement="right"
>  
<template #default>
    <Button
  :label="__('Actions')"
  variant="solid"
  theme="blue"
  icon-right="lucide-chevron-down"
  :loading="!!applyingWorkflowAction"
/>
  </template>
</Dropdown>
<Badge v-if="doc.workflow_state" size="lg" variant="subtle">
  <template #prefix>
    <IndicatorIcon :class="statusColor(doc.workflow_state)" />
  </template>
  {{ statusLabel(doc.workflow_state) }}
</Badge>

<Button
  v-if="showConvertToDeal"
  :label="__('Convert To Deal')"
  variant="solid"
  class="!bg-orange-500 !text-white hover:!bg-orange-600"
  :loading="convertingDeal"
  @click="convertToDeal"
/>
<Button
  v-if="doc.converted_deal"
  :label="__('Go To Deal')"
  variant="subtle"
  class="!bg-orange-500 !text-white hover:!bg-orange-600"
  @click="router.push({ name: 'Deal', params: { dealId: doc.converted_deal } })"
/>
<Button
  v-if="showConvertToLead"
  :label="__('Convert To Lead')"
  variant="solid"
  class="!bg-orange-500 !text-white hover:!bg-orange-600"
  :loading="convertingLead"
  @click="convertToLead"
/>
<Button
  v-if="doc.converted_lead"
  :label="__('Go To Lead')"
  variant="subtle"
  class="!bg-orange-500 !text-white hover:!bg-orange-600"
  @click="router.push({ name: 'Lead', params: { leadId: doc.converted_lead } })"
/>
    </template>
  </LayoutHeader>
  <div v-if="doc.name" class="flex h-full overflow-hidden">
    <Tabs
      v-model="tabIndex"
      :tabs="tabs"
      class="flex flex-1 overflow-hidden flex-col [&_[role='tab']]:px-0 [&_[role='tab']]:shrink-0 [&_[role='tablist']]:px-5 [&_[role='tablist']::-webkit-scrollbar]:h-0 [&_[role='tablist']]:min-h-[45px] [&_[role='tablist']]:gap-7.5 [&_[role='tabpanel']:not([hidden])]:flex [&_[role='tabpanel']:not([hidden])]:grow"
    >
      <template #tab-panel>
        <Activities
          ref="activities"
          v-model:reload="reload"
          v-model:tabIndex="tabIndex"
          doctype="Enquiry Form"
          :docname="enquiryId"
          :tabs="tabs"
          :readOnly="isFormReadOnly"
          @beforeSave="beforeFieldChange"
          @afterSave="reloadResources"
        />
      </template>
    </Tabs>
    <Resizer class="flex flex-col justify-between border-l" side="right">
      <div
        class="flex h-[45px] cursor-copy items-center border-b px-5 py-2.5 text-lg-medium text-ink-gray-9"
        @click="copyToClipboard(enquiryId)"
      >
        {{ __(enquiryId) }}
      </div>
      <div class="flex items-center justify-start gap-5 border-b p-5">
        <Avatar size="3xl" class="size-12" :label="title" />
        <div class="flex flex-col gap-2.5 truncate">
          <Tooltip :text="doc.contact_name || __('Set Contact Name')">
            <div class="truncate text-3xl-medium text-ink-gray-9">
              {{ title }}
            </div>
          </Tooltip>
          <div class="flex gap-1.5">
            <Button
              v-if="callEnabled"
              :tooltip="__('Make a Call')"
              :icon="PhoneIcon"
              @click="
                () =>
                  doc.phone
                    ? makeCall(doc.phone)
                    : toast.error(__('Please set a phone number to make calls'))
              "
            />
            <Button
              :tooltip="__('Send an Email')"
              :icon="Email2Icon"
              @click="
                doc.email
                  ? openEmailBox()
                  : toast.error(__('Please set an email address to send emails'))
              "
            />
            <Button
              :tooltip="__('Attach a File')"
              :icon="AttachmentIcon"
              @click="showFilesUploader = true"
            />
            <Button
              v-if="canDelete"
              :tooltip="__('Delete')"
              variant="subtle"
              theme="red"
              icon="lucide-trash-2"
              @click="deleteEnquiry"
            />
          </div>
          <ErrorMessage :message="__(error)" />
        </div>
      </div>
      <div
        v-if="sections.data"
        class="flex flex-1 flex-col justify-between overflow-hidden"
      >
        <SidePanelLayout
          :sections="sections.data"
          doctype="Enquiry Form"
          :docname="enquiryId"
          @reload="sections.reload"
          @beforeFieldChange="beforeFieldChange"
          @afterFieldChange="reloadResources"
        />
      </div>
    </Resizer>
  </div>
  <ErrorPage
    v-else-if="errorTitle"
    :errorTitle="errorTitle"
    :errorMessage="errorMessage"
  />
  <FilesUploader
    v-model="showFilesUploader"
    doctype="Enquiry Form"
    :docname="enquiryId"
    @after="
      () => {
        activities?.all_activities?.reload()
        changeTabTo('attachments')
      }
    "
  />
  <Dialog v-model:open="showRemarkDialog" :size="'sm'">
  <template #body>
    <div class="bg-surface-elevation-1 px-4 pb-6 pt-5 sm:px-6">
      <div class="mb-5 flex items-center justify-between">
        <h3 class="text-2xl-semibold text-ink-gray-9">
          {{ __(remarkAction) }} {{ __('with Remark') }}
        </h3>
        <Button
          variant="ghost"
          class="w-7"
          icon="lucide-x"
          @click="showRemarkDialog = false"
        />
      </div>
      <FormControl
        type="textarea"
        :label="__('Reason')"
        v-model="remarkReason"
      />
      <div class="mt-5 flex flex-row-reverse gap-2">
        <Button
          variant="solid"
          :label="__('Submit')"
          :loading="applyingWorkflowAction === remarkAction"
          @click="submitRemarkDialog"
        />
      </div>
    </div>
  </template>
</Dialog>
  <DeleteLinkedDocModal
    v-if="showDeleteLinkedDocModal"
    v-model="showDeleteLinkedDocModal"
    :doctype="'Enquiry Form'"
    :docname="enquiryId"
    name="Enquiries"
  />
</template>
<script setup>
import DeleteLinkedDocModal from '@/components/DeleteLinkedDocModal.vue'
import ErrorPage from '@/components/ErrorPage.vue'
import Icon from '@/components/Icon.vue'
import Resizer from '@/components/Resizer.vue'
import ActivityIcon from '@/components/Icons/ActivityIcon.vue'
import EmailIcon from '@/components/Icons/EmailIcon.vue'
import Email2Icon from '@/components/Icons/Email2Icon.vue'
import CommentIcon from '@/components/Icons/CommentIcon.vue'
import DetailsIcon from '@/components/Icons/DetailsIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import TaskIcon from '@/components/Icons/TaskIcon.vue'
import NoteIcon from '@/components/Icons/NoteIcon.vue'
import IndicatorIcon from '@/components/Icons/IndicatorIcon.vue'
import AttachmentIcon from '@/components/Icons/AttachmentIcon.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import Activities from '@/components/Activities/Activities.vue'
import AssignTo from '@/components/AssignTo.vue'
import FilesUploader from '@/components/FilesUploader/FilesUploader.vue'
import SidePanelLayout from '@/components/SidePanelLayout.vue'
import CustomActions from '@/components/CustomActions.vue'
import { copyToClipboard, isTranslatable, setupCustomizations } from '@/utils'
import { getView } from '@/utils/view'
import { getSettings } from '@/stores/settings'
import { globalStore } from '@/stores/global'
import { getMeta } from '@/stores/meta'
import { useDocument } from '@/data/document'
import { callEnabled } from '@/composables/telephony'
import {
  createResource,
  Tooltip,
  Avatar,
  Badge,
  Tabs,
  Breadcrumbs,
  Dropdown,
  call,
  usePageMeta,
  toast,
} from 'frappe-ui'
import { ref, computed, watch, nextTick, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useActiveTabManager } from '@/composables/useActiveTabManager'
const { brand } = getSettings()
const { $dialog, $socket, makeCall } = globalStore()
const { doctypeMeta } = getMeta('Enquiry Form')

const route = useRoute()
const router = useRouter()

const props = defineProps({
  enquiryId: { type: String, required: true },
})

const reload = ref(false)
const activities = ref(null)
const errorTitle = ref('')
const errorMessage = ref('')
const showDeleteLinkedDocModal = ref(false)
const showFilesUploader = ref(false)

const {
  triggerOnChange,
  triggerOnRender,
  assignees,
  permissions,
  document,
  scripts,
  error,
} = useDocument('Enquiry Form', props.enquiryId)

const canDelete = computed(() => permissions.data?.permissions?.delete || false)
const doc = computed(() => document.doc || {})
const isFormReadOnly = computed(() => doc.value.workflow_state === 'Qualified')

onMounted(async () => {
  if (document.doc) await triggerOnRender()
})

watch(error, (err) => {
  if (err) {
    errorTitle.value = __(
      err.exc_type == 'DoesNotExistError'
        ? 'Document not found'
        : 'Error occurred',
    )
    errorMessage.value = __(err.messages?.[0] || 'An error occurred')
  } else {
    errorTitle.value = ''
    errorMessage.value = ''
  }
})

watch(
  () => document.doc,
  async (_doc) => {
    if (scripts.data?.length) {
      let s = await setupCustomizations(scripts.data, {
        doc: _doc,
        $dialog,
        $socket,
        router,
        toast,
        updateField,
        createToast: toast.create,
        deleteDoc: deleteEnquiry,
        call,
      })
      document._actions = s.actions || []
    }
  },
  { once: true },
)

const breadcrumbs = computed(() => {
  let items = [{ label: __('Enquiries'), route: { name: 'Enquiries' } }]

  if (route.query.view || route.query.viewType) {
    let view = getView(route.query.view, route.query.viewType, 'Enquiry Form')
    if (view) {
      items.push({
        label: __(view.label),
        icon: view.icon,
        route: {
          name: 'Enquiries',
          params: { viewType: route.query.viewType },
          query: { view: route.query.view },
        },
      })
    }
  }

  items.push({
    label: title.value,
    route: {
      name: 'Enquiry',
      params: { enquiryId: props.enquiryId },
      query: route.query,
    },
  })
  return items
})

const title = computed(() => {
  let t = doctypeMeta.value?.title_field || 'name'
  return doc.value?.[t] || props.enquiryId
})

usePageMeta(() => {
  return { title: title.value, icon: brand?.favicon }
})

const tabs = computed(() => {
  let tabOptions = [
    { name: 'Activity', label: __('Activity'), icon: ActivityIcon },
    { name: 'Emails', label: __('Emails'), icon: EmailIcon },
    { name: 'Comments', label: __('Comments'), icon: CommentIcon },
    { name: 'Data', label: __('Data'), icon: DetailsIcon },
    { name: 'Calls', label: __('Calls'), icon: PhoneIcon },
    { name: 'Tasks', label: __('Tasks'), icon: TaskIcon },
    { name: 'Notes', label: __('Notes'), icon: NoteIcon },
    { name: 'Attachments', label: __('Attachments'), icon: AttachmentIcon },
  ]
  return tabOptions
})

const { tabIndex, changeTabTo } = useActiveTabManager(tabs, 'lastEnquiryTab')

const sections = createResource({
  url: 'crm.fcrm.doctype.crm_fields_layout.crm_fields_layout.get_sidepanel_sections',
  cache: ['sidePanelSections', 'Enquiry Form'],
  params: { doctype: 'Enquiry Form' },
  auto: true,
})

// Local color map — Enquiry status is a plain Select, not a linked
// "CRM Enquiry Status" doctype the way Lead/Deal statuses are.
const enquiryStatusColor = {
  Qualified: 'text-green-600',
  'Not Qualified': 'text-gray-600',
  'In Review': 'text-blue-600',
  'In Process': 'text-blue-600',
  'Not Pursued': 'text-orange-600',
  DisQualified: 'text-red-600',
}


function statusColor(status) {
  return enquiryStatusColor[status] || 'text-gray-600'
}

function updateField(name, value) {
  value = Array.isArray(name) ? '' : value
  let oldValues = Array.isArray(name) ? {} : doc.value[name]

  if (Array.isArray(name)) {
    name.forEach((field) => (doc.value[field] = value))
  } else {
    doc.value[name] = value
  }

  document.save.submit(null, {
    onSuccess: () => (reload.value = true),
    onError: (err) => {
      if (Array.isArray(name)) {
        name.forEach((field) => (doc.value[field] = oldValues[field]))
      } else {
        doc.value[name] = oldValues
      }
      toast.error(err.messages?.[0] || __('Error updating field'))
    },
  })
}
const convertingDeal = ref(false)
const convertingLead = ref(false)

const showConvertToDeal = computed(() => {
  return (
    !doc.value.converted_deal &&
    doc.value.enquirer_type === 'Existing Customer' &&
    doc.value.workflow_state === 'Qualified'
  )
})

const showConvertToLead = computed(() => {
  return (
    !doc.value.converted_lead &&
    !doc.value.converted_deal &&
    doc.value.enquirer_type === 'New Prospect' &&
    doc.value.workflow_state === 'Qualified'
  )
})

async function convertToDeal() {
  convertingDeal.value = true
  try {
    let dealName = await call(
      'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.create_crm_deal',
      { enquiry: props.enquiryId },
    )
    // No frm.save_or_update() equivalent server-side, so mirror the desk
    // script's client-side link-back explicitly.
    doc.value.converted_deal = dealName
    document.save.submit(null, {
      onSuccess: () => {
        toast.success(__('CRM Deal created successfully'))
        router.push({ name: 'Deal', params: { dealId: dealName } })
      },
      onError: (err) => {
        toast.error(err.messages?.[0] || __('Error saving converted_deal'))
      },
    })
  } catch (err) {
    toast.error(err.messages?.[0] || __('Error creating CRM Deal'))
  } finally {
    convertingDeal.value = false
  }
}

async function convertToLead() {
  convertingLead.value = true
  try {
    let leadName = await call(
      'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.create_lead',
      { enquiry: props.enquiryId },
    )
    doc.value.converted_lead = leadName
    document.save.submit(null, {
      onSuccess: () => {
        toast.success(__('Lead created successfully'))
        router.push({ name: 'Lead', params: { leadId: leadName } })
      },
      onError: (err) => {
        toast.error(err.messages?.[0] || __('Error saving converted_lead'))
      },
    })
  } catch (err) {
    toast.error(err.messages?.[0] || __('Error creating Lead'))
  } finally {
    convertingLead.value = false
  }
}

function deleteEnquiry() {
  showDeleteLinkedDocModal.value = true
}

function openEmailBox() {
  let currentTab = tabs.value[tabIndex.value]
  if (!['Emails', 'Comments', 'Activities'].includes(currentTab.name)) {
    activities.value.changeTabTo('emails')
  }
  nextTick(() => (activities.value.emailBox.show = true))
}

function statusLabel(status) {
  if (isTranslatable('Enquiry Form')) return __(status)
  return status
}

function beforeFieldChange(data) {
  document.save.submit(null, {
    onSuccess: () => reloadResources(data),
  })
}

function reloadResources(data) {
  if (Object.hasOwn(data ?? {}, 'owner_user')) {
    assignees.reload()
  }
}
const workflowTransitions = ref([])
const applyingWorkflowAction = ref(null)
const showRemarkDialog = ref(false)
const remarkAction = ref('')
const remarkReason = ref('')

// Exact strings from your Workflow's transition actions that require a
// remark, matching frm.selected_workflow_action checks in enquiry_form.js
const REMARK_REQUIRED_ACTIONS = ['DisQualified', 'Not Pursued', 'Not Qualified']
const workflowDropdownOptions = computed(() =>
  workflowTransitions.value.map((transition) => ({
    label: __(transition.action),
    onClick: () => onWorkflowActionClick(transition.action),
  })),
)
async function loadWorkflowTransitions() {
  if (!doc.value.name) {
    workflowTransitions.value = []
    return
  }
  try {
    workflowTransitions.value = await call(
      'frappe.model.workflow.get_transitions',
      { doc: doc.value },
    )
  } catch (err) {
    workflowTransitions.value = []
  }
}

watch(() => doc.value.workflow_state, () => loadWorkflowTransitions())

onMounted(() => {
  if (document.doc) loadWorkflowTransitions()
})

function onWorkflowActionClick(action) {
  if (REMARK_REQUIRED_ACTIONS.includes(action)) {
    remarkAction.value = action
    remarkReason.value = ''
    showRemarkDialog.value = true
    return
  }
  runWorkflowAction(action)
}

async function submitRemarkDialog() {
  if (!remarkReason.value) {
    toast.error(__('Reason is required'))
    return
  }
  showRemarkDialog.value = false
  await runWorkflowAction(remarkAction.value, remarkReason.value)
}

async function runWorkflowAction(action, reason) {
  applyingWorkflowAction.value = action
  try {
    let updatedDoc = await call('frappe.model.workflow.apply_workflow', {
      doc: doc.value,
      action,
    })
    Object.assign(doc.value, updatedDoc)

    if (action === 'DisQualified' && reason) {
      await call(
        'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.update_disqualification_reason',
        { docname: doc.value.name, reason },
      )
    } else if (action === 'Not Pursued' && reason) {
      await call(
        'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.update_not_pursued_reason',
        { docname: doc.value.name, reason },
      )
    } else if (action === 'Not Qualified' && reason) {
      // Desk form has no update_* whitelist method for this one — it just
      // sets not_qualified via frm.set_value + frm.save(). Mirroring that
      // directly since there's nothing server-side to call for it.
      doc.value.not_qualified = reason
      await document.save.submit()
    }

    toast.success(__('Status updated'))
    reload.value = true
    loadWorkflowTransitions()
  } catch (err) {
    toast.error(err.messages?.[0] || __('Error updating workflow state'))
  } finally {
    applyingWorkflowAction.value = null
  }
}
</script>