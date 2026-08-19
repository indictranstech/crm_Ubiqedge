<template>
  <LayoutHeader>
    <template #left-header>
      <ViewBreadcrumbs v-model="viewControls" routeName="Enquiries" />
    </template>
    <template #right-header>
      <CustomActions
        v-if="enquiriesListView?.customListActions"
        :actions="enquiriesListView.customListActions"
      />
      <Button
        variant="solid"
        :label="__('Create')"
        iconLeft="plus"
        @click="showEnquiryModal = true"
      />
    </template>
  </LayoutHeader>
  <ViewControls
    ref="viewControls"
    v-model="enquiries"
    v-model:loadMore="loadMore"
    v-model:resizeColumn="triggerResize"
    v-model:updatedPageCount="updatedPageCount"
    doctype="Enquiry Form"
    :filters="{}"
    :options="{
      allowedViews: ['list', 'group_by'],
    }"
  />
  <EnquiriesListView
    v-if="enquiries.data && rows.length"
    ref="enquiriesListView"
    v-model="enquiries.data.page_length_count"
    v-model:list="enquiries"
    :rows="rows"
    :columns="columns"
    :options="{
      showTooltip: false,
      resizeColumn: true,
      rowCount: enquiries.data.row_count,
      totalCount: enquiries.data.total_count,
    }"
    @loadMore="() => loadMore++"
    @columnWidthUpdated="() => triggerResize++"
    @updatePageCount="(count) => (updatedPageCount = count)"
    @applyFilter="(data) => viewControls.applyFilter(data)"
    @applyLikeFilter="(data) => viewControls.applyLikeFilter(data)"
    @likeDoc="(data) => viewControls.likeDoc(data)"
    @selectionsChanged="
      (selections) => viewControls.updateSelections(selections)
    "
  />
  <EmptyState
    v-else-if="enquiries.data && !rows.length"
    name="Enquiries"
    :icon="EnquiriesIcon"
  />
  <EnquiryModal
    v-if="showEnquiryModal"
    v-model="showEnquiryModal"
    :defaults="defaults"
  />
</template>

<script setup>
import ViewBreadcrumbs from '@/components/ViewBreadcrumbs.vue'
import MultipleAvatar from '@/components/MultipleAvatar.vue'
import CustomActions from '@/components/CustomActions.vue'
import EmailAtIcon from '@/components/Icons/EmailAtIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import NoteIcon from '@/components/Icons/NoteIcon.vue'
import TaskIcon from '@/components/Icons/TaskIcon.vue'
import CommentIcon from '@/components/Icons/CommentIcon.vue'
import IndicatorIcon from '@/components/Icons/IndicatorIcon.vue'
import EnquiriesIcon from '@/components/Icons/EnquiriesIcon.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import EnquiriesListView from '@/components/ListViews/EnquiriesListView.vue'
import EmptyState from '@/components/ListViews/EmptyState.vue'
import EnquiryModal from '@/components/Modals/EnquiryModal.vue'
import ViewControls from '@/components/ViewControls.vue'
import { useDoctypeModal } from '@/composables/doctypeModal'
import { getMeta } from '@/stores/meta'
import { globalStore } from '@/stores/global'
import { usersStore } from '@/stores/users'
import { callEnabled } from '@/composables/telephony'
import { useBroadcast } from '@/composables/useBroadcast'
import { formatDate, timeAgo, formatTime } from '@/utils'
import { timestampCell } from '@/composables/useTimelinePreferences'
import { useOnboarding, useTelemetry } from 'frappe-ui/frappe'
import { Tooltip, Dropdown } from 'frappe-ui'
import { ref, computed, reactive, h } from 'vue'

const { getFormattedPercent, getFormattedFloat, getFormattedCurrency } =
  getMeta('Enquiry Form')
const { makeCall } = globalStore()
const { getUser } = usersStore()
const { on } = useBroadcast()
const { updateOnboardingStep } = useOnboarding('frappecrm')
const { capture } = useTelemetry()
const { showModal } = useDoctypeModal()

// Local color map since Enquiry status is a plain Select, not a linked
// "CRM Enquiry Status" doctype like Lead/Deal use.
const enquiryStatusColor = {
  Qualified: 'text-green-600',
  'Not Qualified': 'text-gray-600',
  'In Process': 'text-blue-600',
  'Not Pursued': 'text-orange-600',
  DisQualified: 'text-red-600',
}

const enquiriesListView = ref(null)
const showEnquiryModal = ref(false)

on('trigger_enquiry_create', (data) => {
  showEnquiryModal.value = Boolean(data)
})

const defaults = reactive({})

// enquiries data is loaded in the ViewControls component
const enquiries = ref({})
const loadMore = ref(1)
const triggerResize = ref(1)
const updatedPageCount = ref(20)
const viewControls = ref(null)

function getRow(name, field) {
  function getValue(value) {
    if (value && typeof value === 'object' && !Array.isArray(value)) {
      return value
    }
    return { label: value }
  }
  return getValue(rows.value?.find((row) => row.name == name)[field])
}

// Rows
const rows = computed(() => {
  if (!enquiries.value?.data?.data) return []
  if (enquiries.value.data.view_type === 'group_by') {
    if (!enquiries.value?.data.group_by_field?.fieldname) return []
    return getGroupedByRows(
      enquiries.value?.data.data,
      enquiries.value?.data.group_by_field,
      enquiries.value.data.columns,
    )
  } else {
    return parseRows(enquiries.value?.data.data, enquiries.value.data.columns)
  }
})

const columns = computed(() => {
  let _columns = enquiries.value?.data?.columns || []

  // Set align right for last column
  if (_columns.length) {
    _columns = _columns.map((col, index) => {
      if (index === _columns.length - 1) {
        return { ...col, align: 'right' }
      }
      return col
    })
  }

  return _columns
})

function getGroupedByRows(listRows, groupByField, columns) {
  let groupedRows = []

  groupByField.options?.forEach((option) => {
    let filteredRows

    if (!option) {
      filteredRows = listRows.filter((row) => !row[groupByField.fieldname])
    } else {
      filteredRows = listRows.filter(
        (row) => row[groupByField.fieldname] == option,
      )
    }

    let groupDetail = {
      label: groupByField.label,
      group: option || __(' '),
      collapsed: false,
      rows: parseRows(filteredRows, columns),
    }
    if (groupByField.fieldname == 'status') {
      groupDetail.icon = () =>
        h(IndicatorIcon, {
          class: enquiryStatusColor[option] || 'text-gray-600',
        })
    }
    groupedRows.push(groupDetail)
  })

  return groupedRows || listRows
}

function parseRows(rows, columns = []) {
  let key = 'key'
  let type = 'type'

  return rows.map((enquiry) => {
    let _rows = {}
    enquiries.value?.data.rows.forEach((row) => {
      _rows[row] = enquiry[row]

      let fieldType = columns?.find((col) => (col[key] || col.value) == row)?.[
        type
      ]

      if (
        fieldType &&
        ['Date', 'Datetime'].includes(fieldType) &&
        !['modified', 'creation'].includes(row)
      ) {
        _rows[row] = formatDate(enquiry[row], '', true, fieldType == 'Datetime')
      }

      if (fieldType && fieldType == 'Currency') {
        _rows[row] = getFormattedCurrency(row, enquiry)
      }

      if (fieldType && fieldType == 'Float') {
        _rows[row] = getFormattedFloat(row, enquiry)
      }

      if (fieldType && fieldType == 'Percent') {
        _rows[row] = getFormattedPercent(row, enquiry)
      }

      if (row == 'contact_name') {
        _rows[row] = {
          label: enquiry.contact_name,
        }
      } else if (row == 'organization') {
        _rows[row] = enquiry.organization
      } else if (row == 'status') {
        _rows[row] = {
          label: enquiry.status,
          color: enquiryStatusColor[enquiry.status] || 'text-gray-600',
        }
      } else if (row == 'owner_user') {
        _rows[row] = {
          label: enquiry.owner_user && getUser(enquiry.owner_user).full_name,
          ...(enquiry.owner_user && getUser(enquiry.owner_user)),
        }
      } else if (row == '_assign') {
        let assignees = JSON.parse(enquiry._assign || '[]')
        _rows[row] = assignees.map((user) => ({
          name: user,
          image: getUser(user).user_image,
          label: getUser(user).full_name,
        }))
      } else if (['modified', 'creation'].includes(row)) {
        _rows[row] = timestampCell(enquiry[row])
      }
    })
    _rows['_email_count'] = enquiry._email_count
    _rows['_note_count'] = enquiry._note_count
    _rows['_task_count'] = enquiry._task_count
    _rows['_comment_count'] = enquiry._comment_count
    return _rows
  })
}

function actions(itemName) {
  let phone = getRow(itemName, 'phone')?.label || ''
  let actions = [
    {
      icon: h(PhoneIcon, { class: 'h-4 w-4' }),
      label: __('Make a Call'),
      onClick: () => makeCall(phone),
      condition: () => phone && callEnabled.value,
    },
    {
      icon: h(NoteIcon, { class: 'h-4 w-4' }),
      label: __('New Note'),
      onClick: () => showNote(itemName),
    },
    {
      icon: h(TaskIcon, { class: 'h-4 w-4' }),
      label: __('New Task'),
      onClick: () => showTask(itemName),
    },
  ]
  return actions.filter((action) =>
    action.condition ? action.condition() : true,
  )
}

function showNote(name) {
  showModal({
    doctype: 'FCRM Note',
    title: 'Note',
    defaults: {
      reference_doctype: 'Enquiry Form',
      reference_docname: name,
    },
    callbacks: {
      afterInsert: (d) => after(d, true),
      afterUpdate: after,
    },
  })
}

function showTask(name) {
  showModal({
    doctype: 'CRM Task',
    title: 'Task',
    defaults: {
      reference_doctype: 'Enquiry Form',
      reference_docname: name,
    },
    callbacks: {
      afterInsert: (d) => after(d, true),
      afterUpdate: after,
    },
  })
}

function after(d, isNew = false) {
  let a = d.doctype == 'FCRM Note' ? 'note' : 'task'
  if (isNew) {
    updateOnboardingStep('create_first_' + a)
    capture(a + '_created')
  } else {
    capture(a + '_updated')
  }
}
</script>