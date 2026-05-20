<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="emit('close')">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="emit('close')">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item info (read-only in both modes) -->
            <div class="item-header">
              <div class="item-title-section">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <div class="shortage-summary">
              <div class="summary-card">
                <div class="summary-label">Quantity Needed</div>
                <div class="summary-value">{{ backlogItem.quantity_needed }} units</div>
              </div>
              <div class="summary-card">
                <div class="summary-label">Quantity Available</div>
                <div class="summary-value">{{ backlogItem.quantity_available }} units</div>
              </div>
              <div class="summary-card danger">
                <div class="summary-label">Shortage</div>
                <div class="summary-value">{{ shortage }} units</div>
              </div>
            </div>

            <!-- CREATE mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitPO">
              <div class="form-group">
                <label class="form-label" for="supplier-name">Supplier Name</label>
                <input
                  id="supplier-name"
                  v-model="supplierName"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="order-quantity">Order Quantity</label>
                  <input
                    id="order-quantity"
                    v-model.number="orderQuantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit-cost">Unit Cost ($)</label>
                  <input
                    id="unit-cost"
                    v-model.number="unitCost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label" for="expected-delivery">Expected Delivery Date</label>
                <input
                  id="expected-delivery"
                  v-model="expectedDelivery"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div v-if="orderQuantity > 0 && unitCost > 0" class="total-cost-preview">
                <span class="total-label">Total Cost</span>
                <span class="total-value">${{ (orderQuantity * unitCost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
              </div>
            </form>

            <!-- VIEW mode: PO details -->
            <div v-else-if="mode === 'view' && backlogItem.purchase_order" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value po-id">{{ backlogItem.purchase_order.id }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="status-badge" :class="backlogItem.purchase_order.status">
                      {{ backlogItem.purchase_order.status }}
                    </span>
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Order Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ backlogItem.purchase_order.unit_cost?.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value highlight">${{ backlogItem.purchase_order.total_cost?.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Created At</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_at) }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <template v-if="mode === 'create'">
              <button type="button" class="btn-secondary" @click="emit('close')">Cancel</button>
              <button type="submit" class="btn-primary" form="po-form-submit" @click="submitPO">
                Create Purchase Order
              </button>
            </template>
            <template v-else>
              <button class="btn-secondary" @click="emit('close')">Close</button>
            </template>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
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
    validator: (v) => ['create', 'view'].includes(v)
  }
})

const emit = defineEmits(['close', 'po-created'])

// Form state — reset when modal opens
const supplierName = ref('')
const orderQuantity = ref(0)
const unitCost = ref(0)
const expectedDelivery = ref('')

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
})

// Pre-fill order quantity with shortage when modal opens in create mode
watch(
  () => props.isOpen,
  (open) => {
    if (open && props.mode === 'create') {
      supplierName.value = ''
      // Pre-fill with shortage amount, minimum 1
      orderQuantity.value = Math.max(shortage.value, 1)
      unitCost.value = 0
      expectedDelivery.value = ''
    }
  }
)

const submitPO = () => {
  if (!supplierName.value || !orderQuantity.value || !unitCost.value || !expectedDelivery.value) {
    return
  }

  const poData = {
    id: 'PO-' + Date.now(),
    backlog_item_id: props.backlogItem.id,
    item_sku: props.backlogItem.item_sku,
    item_name: props.backlogItem.item_name,
    supplier: supplierName.value,
    quantity: orderQuantity.value,
    unit_cost: unitCost.value,
    total_cost: orderQuantity.value * unitCost.value,
    expected_delivery: expectedDelivery.value,
    status: 'pending',
    created_at: new Date().toISOString()
  }

  emit('po-created', poData)
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return 'N/A'
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
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
  padding: 2rem;
}

.item-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-title-section {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.375rem 0;
}

.item-sku {
  font-size: 0.875rem;
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

.shortage-summary {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.75rem;
  margin-bottom: 1.75rem;
}

.summary-card {
  padding: 1rem;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  background: #f8fafc;
}

.summary-card.danger {
  border-color: #fecaca;
  background: #fef2f2;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.375rem;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-card.danger .summary-value {
  color: #dc2626;
}

/* Form styles */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  font-family: inherit;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-input::placeholder {
  color: #9ca3af;
}

.total-cost-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.25rem;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
}

.total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #1e40af;
}

.total-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #1d4ed8;
}

/* View mode styles */
.po-details {
  /* container for view mode PO info */
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.po-id {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
  font-size: 0.875rem;
}

.info-value.highlight {
  color: #0f172a;
  font-weight: 700;
  font-size: 1rem;
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

.status-badge.confirmed {
  background: #dcfce7;
  color: #166534;
}

.status-badge.shipped {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.cancelled {
  background: #fee2e2;
  color: #991b1b;
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

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
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

.btn-primary:hover {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

/* Modal transition animations */
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
