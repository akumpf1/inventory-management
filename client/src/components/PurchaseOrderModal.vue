<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item summary row -->
            <div class="item-summary">
              <div class="item-summary-info">
                <div class="item-name">{{ backlogItem.item_name }}</div>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority?.toLowerCase()">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- CREATE mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div v-if="formError" class="form-error">{{ formError }}</div>

              <div class="form-grid">
                <div class="form-group full-width">
                  <label class="form-label" for="po-supplier">Supplier Name</label>
                  <input
                    id="po-supplier"
                    v-model="form.supplier_name"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="po-quantity">Quantity</label>
                  <input
                    id="po-quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="po-unit-cost">Unit Cost ($)</label>
                  <input
                    id="po-unit-cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                    required
                  />
                </div>

                <div class="form-group full-width">
                  <label class="form-label" for="po-delivery-date">Expected Delivery Date</label>
                  <input
                    id="po-delivery-date"
                    v-model="form.expected_delivery_date"
                    type="date"
                    class="form-input"
                    required
                  />
                </div>

                <div class="form-group full-width">
                  <label class="form-label" for="po-notes">Notes</label>
                  <textarea
                    id="po-notes"
                    v-model="form.notes"
                    class="form-textarea"
                    placeholder="Optional notes..."
                    rows="3"
                  />
                </div>
              </div>

              <!-- Calculated total -->
              <div v-if="form.quantity && form.unit_cost" class="total-preview">
                <span class="total-label">Estimated Total</span>
                <span class="total-value">{{ formatCurrency(form.quantity * form.unit_cost) }}</span>
              </div>
            </form>

            <!-- VIEW mode: PO details -->
            <div v-else-if="mode === 'view' && purchaseOrder" class="po-details">
              <div class="details-grid">
                <div class="detail-item">
                  <div class="detail-label">PO ID</div>
                  <div class="detail-value mono">{{ purchaseOrder.id || backlogItem.purchase_order_id }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Supplier</div>
                  <div class="detail-value">{{ purchaseOrder.supplier_name }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Quantity</div>
                  <div class="detail-value">{{ purchaseOrder.quantity }} units</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Unit Cost</div>
                  <div class="detail-value">{{ formatCurrency(purchaseOrder.unit_cost) }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Total Cost</div>
                  <div class="detail-value highlight">{{ formatCurrency(purchaseOrder.total_cost || (purchaseOrder.quantity * purchaseOrder.unit_cost)) }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Expected Delivery</div>
                  <div class="detail-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                </div>
                <div class="detail-item">
                  <div class="detail-label">Status</div>
                  <div class="detail-value">
                    <span class="status-badge" :class="purchaseOrder.status?.toLowerCase()">
                      {{ purchaseOrder.status || 'Pending' }}
                    </span>
                  </div>
                </div>
                <div v-if="purchaseOrder.notes" class="detail-item full-width">
                  <div class="detail-label">Notes</div>
                  <div class="detail-value">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
            </div>

            <div v-else-if="mode === 'view'" class="no-po">
              No purchase order data available.
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="submitting">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              type="submit"
              :disabled="submitting"
              @click="submitForm"
            >
              <span v-if="submitting">Creating...</span>
              <span v-else>Create Purchase Order</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    mode: {
      type: String,
      default: 'create',
      validator: (val) => ['create', 'view'].includes(val)
    }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const submitting = ref(false)
    const formError = ref(null)

    // Computed shortage for default quantity
    const shortage = computed(() => {
      if (!props.backlogItem) return 0
      return Math.max(0, (props.backlogItem.quantity_needed || 0) - (props.backlogItem.quantity_available || 0))
    })

    // Form state — reset when modal opens or backlogItem changes
    const form = ref({
      supplier_name: '',
      quantity: 0,
      unit_cost: '',
      expected_delivery_date: '',
      notes: ''
    })

    const resetForm = () => {
      formError.value = null
      form.value = {
        supplier_name: '',
        quantity: shortage.value || 1,
        unit_cost: '',
        expected_delivery_date: '',
        notes: ''
      }
    }

    // Reset form whenever the modal opens
    watch(() => props.isOpen, (isOpen) => {
      if (isOpen) resetForm()
    })

    watch(() => props.backlogItem, () => {
      if (props.isOpen) resetForm()
    })

    const purchaseOrder = computed(() => {
      return props.backlogItem?.purchase_order || null
    })

    const formatCurrency = (value) => {
      if (value == null || isNaN(value)) return '-'
      return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(value)
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const close = () => {
      if (!submitting.value) emit('close')
    }

    const submitForm = async () => {
      if (submitting.value) return
      formError.value = null
      submitting.value = true

      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          item_sku: props.backlogItem.item_sku,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || ''
        }
        const result = await api.createPurchaseOrder(payload)
        emit('po-created', result)
      } catch (err) {
        formError.value = err?.response?.data?.detail || 'Failed to create purchase order. Please try again.'
        console.error('PO creation failed:', err)
      } finally {
        submitting.value = false
      }
    }

    return {
      form,
      submitting,
      formError,
      shortage,
      purchaseOrder,
      formatCurrency,
      formatDate,
      close,
      submitForm
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem;
}

/* Item summary */
.item-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  margin-bottom: 1.5rem;
}

.item-summary-info {
  min-width: 0;
}

.item-name {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

/* Form */
.form-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.875rem;
  padding: 0.75rem 1rem;
  margin-bottom: 1.25rem;
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-group.full-width {
  grid-column: 1 / -1;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-input,
.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.total-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 1.25rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  margin-top: 1.25rem;
}

.total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #166534;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.total-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #16a34a;
}

/* View mode */
.details-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.detail-item.full-width {
  grid-column: 1 / -1;
}

.detail-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.detail-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.detail-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.detail-value.highlight {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.approved {
  background: #dcfce7;
  color: #166534;
}

.status-badge.shipped {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.delivered {
  background: #dcfce7;
  color: #166534;
}

.status-badge.cancelled {
  background: #f1f5f9;
  color: #64748b;
}

.no-po {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.875rem;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
