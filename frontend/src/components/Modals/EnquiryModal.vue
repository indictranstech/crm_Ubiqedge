<template>
  <Dialog v-model:open="show" :size="'3xl'">
    <template #body>
      <div class="bg-surface-elevation-1 px-4 pb-6 pt-5 sm:px-6">
        <div class="mb-5 flex items-center justify-between">
          <div>
            <h3 class="text-3xl-semibold leading-6 text-ink-gray-9">
              {{ __('Create Enquiry') }}
            </h3>
          </div>
          <div class="flex items-center gap-1">
            <Button
              v-if="isManager() && !isMobileView"
              variant="ghost"
              class="w-7"
              :tooltip="__('Edit Fields Layout')"
              :icon="EditIcon"
              @click="openQuickEntryModal"
            />
            <Button
              variant="ghost"
              class="w-7"
              icon="lucide-x"
              @click="show = false"
            />
          </div>
        </div>
        <div>
          
          <!-- <FieldLayout v-if="tabs.data" :tabs="tabs.data" v-model:data="enquiry.doc"/> -->
<FieldLayout
  v-if="tabs.data"
  :tabs="tabs.data"
  :data="enquiry.doc"
  doctype="Enquiry Form"
  :docname="enquiry.doc.name || ''"
/>
         <ErrorMessage v-if="error" class="mt-4" :message="__(error)" />
        </div>
      </div>
      <div class="px-4 pb-7 pt-4 sm:px-6">
        <div class="flex flex-row-reverse gap-2">
          <Button
            variant="solid"
            :label="__('Create')"
            :loading="isEnquiryCreating"
            @click="createNewEnquiry"
          />
        </div>
      </div>
    </template>
  </Dialog>
</template>

<script setup>
import EditIcon from '@/components/Icons/EditIcon.vue'
import FieldLayout from '@/components/FieldLayout/FieldLayout.vue'
import { usersStore } from '@/stores/users'
import { sessionStore } from '@/stores/session'
import { isMobileView } from '@/composables/settings'
import { showQuickEntryModal, quickEntryProps } from '@/composables/modals'
import { useOnboarding, useTelemetry } from 'frappe-ui/frappe'
import { call, createResource } from 'frappe-ui'
import { useDocument } from '@/data/document'
import { ref, onMounted, nextTick, watch } from 'vue'
import { useRouter } from 'vue-router'

const props = defineProps({
  defaults: { type: Object, default: () => ({}) },
})

const { user } = sessionStore()
const { getUser, isManager } = usersStore()
const { updateOnboardingStep } = useOnboarding('frappecrm')

const show = defineModel({ type: Boolean })
const router = useRouter()
const error = ref(null)
const isEnquiryCreating = ref(false)

const { document: enquiry, triggerOnBeforeCreate } = useDocument('Enquiry Form')

const { capture } = useTelemetry()

const LINK_FIELD_OVERRIDES = {
  organization: { fieldtype: 'Link', options: 'CRM Organization' },
  source: { fieldtype: 'Link', options: 'Lead Source' },
  customer: { fieldtype: 'Link', options: 'Customer' },
  owner_user: {
    fieldtype: 'Link',
    options: 'User',
    get_query: () => ({
      query: 'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.get_sales_executive_users',
    }),
  },
}

const tabs = createResource({
  url: 'crm.fcrm.doctype.crm_fields_layout.crm_fields_layout.get_fields_layout',
  cache: ['QuickEntry', 'Enquiry Form', 'v2'],
  params: { doctype: 'Enquiry Form', type: 'Quick Entry' },
  auto: true,
  transform: (_tabs) => {
    _tabs.forEach((tab) => {
        tab.sections.forEach((section) => {
            section.columns.forEach((column) => {
                column.fields.forEach((field) => {
                    if (field.fieldtype === 'Table') {
                        enquiry.doc[field.fieldname] = []
                    }

                    const override = LINK_FIELD_OVERRIDES[field.fieldname]

                    if (override) {
                        field.fieldtype = override.fieldtype
                        field.options = override.options
                    }
                })
            })
        })
    })

    return _tabs
},
})

const createEnquiry = createResource({
  url: 'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.create_enquiry',
})

watch(
  () => enquiry.doc.customer,
  async (customer) => {
    if (!customer) {
      enquiry.doc.contact_name = ''
      enquiry.doc.email = ''
      enquiry.doc.phone = ''
      return
    }
    const contactName = await call(
      'ubiqedge.ubiqedge_erp.doctype.enquiry_form.enquiry_form.get_default_contact',
      { doctype: 'Customer', name: customer },
    )
    if (contactName) {
      const contact = await call('frappe.client.get', {
        doctype: 'Contact',
        name: contactName,
      })
      enquiry.doc.contact_name = contact.name
      enquiry.doc.email = contact.email_id
      enquiry.doc.phone = contact.mobile_no
    }
  },
)

// Mirrors the desk form's enquirer_type -> clear_customer_details behavior:
// switching back to New Prospect clears the customer + auto-filled contact.
watch(
  () => enquiry.doc.enquirer_type,
  (type, oldType) => {
    if (type === 'New Prospect' && oldType === 'Existing Customer') {
      enquiry.doc.customer = ''
      enquiry.doc.contact_name = ''
      enquiry.doc.email = ''
      enquiry.doc.phone = ''
    }
  },
)
async function createNewEnquiry() {
  await triggerOnBeforeCreate?.()

  await nextTick()

  const payload = JSON.parse(JSON.stringify(enquiry.doc))

  console.log('SUBMIT PAYLOAD:', payload)

  error.value = null

  if (payload.enquirer_type === 'Existing Customer' && !payload.customer) {
    error.value = __('Customer is mandatory for Existing Customer')
    return
  }

  if (!payload.contact_name) {
    error.value = __('Contact Name is mandatory')
    return
  }

  if (!payload.email) {
    error.value = __('Email is mandatory')
    return
  }

  if (!payload.email.includes('@')) {
    error.value = __('Invalid email address')
    return
  }
if (!payload.phone) {
  error.value = __('Phone is mandatory')
  return
}

const cleanedPhone = String(payload.phone).replace(/[-+() ]/g, '')

if (isNaN(cleanedPhone)) {
  error.value = __('Phone should be a number')
  return
}

if (cleanedPhone.length !== 10) {
  error.value = __('Phone number must be exactly 10 digits')
  return
}

  if (isNaN(String(payload.phone).replace(/[-+() ]/g, ''))) {
    error.value = __('Phone should be a number')
    return
  }

  if (!payload.organization) {
    error.value = __('Organization is mandatory')
    return
  }

  if (!payload.owner_user) {
    error.value = __('Owner User is mandatory')
    return
  }

  if (!payload.source) {
    error.value = __('Source is mandatory')
    return
  }

  if (!payload.priority) {
    error.value = __('Priority is mandatory')
    return
  }

  isEnquiryCreating.value = true

  createEnquiry.submit(
    { doc: payload },
    {
      onSuccess(name) {
        capture('enquiry_created')
        isEnquiryCreating.value = false
        show.value = false
        enquiry.doc = {}
        router.push({
          name: 'Enquiry',
          params: { enquiryId: name },
        })

        updateOnboardingStep(
          'create_first_enquiry',
          true,
          false,
          () => {
            localStorage.setItem('firstEnquiry' + user, name)
          },
        )
      },

      onError(err) {
        isEnquiryCreating.value = false

        if (!err.messages) {
          error.value = err.message
          return
        }

        error.value = err.messages.join('\n')
      },
    },
  )
}
function openQuickEntryModal() {
  showQuickEntryModal.value = true
  quickEntryProps.value = { doctype: 'Enquiry Form' }
  nextTick(() => (show.value = false))
}

onMounted(() => {
  enquiry.doc.enquirer_type = 'New Prospect'
  Object.assign(enquiry.doc, props.defaults)
})
watch(
  () => enquiry.doc,
  (value) => {
    console.log('ENQUIRY DOC:', JSON.parse(JSON.stringify(value)))
  },
  { deep: true },
)
</script>