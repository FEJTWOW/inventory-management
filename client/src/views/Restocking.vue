<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend and order items based on demand forecasts and stock levels</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <span class="budget-display">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="budget-slider-row">
          <span class="slider-label">$0</span>
          <input
            type="range"
            min="0"
            max="500000"
            step="5000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="slider-label">$500,000</span>
        </div>

        <div class="budget-summary-row">
          <div class="budget-stat">
            <span class="budget-stat-label">Candidates</span>
            <span class="budget-stat-value">{{ candidates.length }}</span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">Selected</span>
            <span class="budget-stat-value">{{ selectedSkus.size }}</span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">Selected Cost</span>
            <span class="budget-stat-value">{{ formatCurrency(totalSelectedCost) }}</span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">Remaining</span>
            <span
              class="budget-stat-value"
              :class="{ 'over-budget': totalSelectedCost > budget }"
            >{{ formatCurrency(budget - totalSelectedCost) }}</span>
          </div>
        </div>

        <!-- Progress bar -->
        <div class="progress-bar-container">
          <div
            class="progress-bar"
            :class="progressBarClass"
            :style="{ width: progressPercent + '%' }"
          ></div>
        </div>
        <div class="progress-labels">
          <span class="progress-current">{{ formatCurrency(totalSelectedCost) }}</span>
          <span class="progress-budget">{{ formatCurrency(budget) }} budget</span>
        </div>
      </div>

      <!-- Over-budget warning -->
      <div v-if="totalSelectedCost > budget" class="warning-banner">
        Selection exceeds budget. Deselect items or increase your budget.
      </div>

      <!-- Success banner -->
      <div v-if="successOrder" class="success-banner">
        Order {{ successOrder.order_number }} placed! Expected delivery: {{ formatDate(successOrder.expected_delivery) }}
      </div>

      <!-- Candidates Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ candidates.length }})</h3>
          <div class="table-actions">
            <button class="btn-text" @click="selectAll">Select All</button>
            <button class="btn-text" @click="deselectAll">Deselect All</button>
          </div>
        </div>

        <div v-if="candidates.length === 0" class="empty-state">
          No items require restocking based on current demand forecasts and stock levels.
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-name">Item Name</th>
                <th class="col-trend">Trend</th>
                <th class="col-demand">Forecasted Demand</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Restock Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="candidate in candidates"
                :key="candidate.sku"
                :class="{ 'row-selected': selectedSkus.has(candidate.sku) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(candidate.sku)"
                    @change="toggleSelection(candidate.sku)"
                    class="row-checkbox"
                  />
                </td>
                <td class="col-sku">
                  <strong class="sku-text">{{ candidate.sku }}</strong>
                </td>
                <td class="col-name">{{ candidate.name }}</td>
                <td class="col-trend">
                  <span :class="['badge', candidate.trend]">{{ candidate.trend }}</span>
                </td>
                <td class="col-demand">{{ candidate.forecasted_demand.toLocaleString() }}</td>
                <td class="col-cost">
                  {{ candidate.unit_cost !== null ? formatCurrency(candidate.unit_cost) : '—' }}
                </td>
                <td class="col-total">
                  <strong v-if="candidate.restock_cost !== null">{{ formatCurrency(candidate.restock_cost) }}</strong>
                  <span v-else class="muted">—</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="table-footer">
          <div class="footer-total">
            <span class="footer-total-label">Selected Total:</span>
            <span class="footer-total-value" :class="{ 'over-budget': totalSelectedCost > budget }">
              {{ formatCurrency(totalSelectedCost) }}
            </span>
          </div>
          <button
            class="btn-primary"
            :disabled="isPlaceOrderDisabled || placingOrder"
            @click="placeOrder"
          >
            {{ placingOrder ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const placingOrder = ref(false)
    const successOrder = ref(null)

    const forecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(100000)

    // Track selection as a Set of SKU strings
    // Reassign the entire Set to trigger Vue reactivity
    const selectedSkus = ref(new Set())

    // Compute candidates by joining forecasts with inventory
    const candidates = computed(() => {
      const invMap = new Map(inventoryItems.value.map(inv => [inv.sku, inv]))

      const result = []
      for (const forecast of forecasts.value) {
        const inv = invMap.get(forecast.item_sku) ?? null

        const isTrendCandidate =
          forecast.trend === 'increasing' || forecast.trend === 'stable'
        const isBelowReorder =
          inv !== null && inv.quantity_on_hand <= inv.reorder_point

        if (!isTrendCandidate && !isBelowReorder) continue

        result.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          forecasted_demand: forecast.forecasted_demand,
          unit_cost: inv ? inv.unit_cost : null,
          restock_cost: inv ? forecast.forecasted_demand * inv.unit_cost : null
        })
      }

      // Sort: "increasing" first, then by restock_cost descending
      result.sort((a, b) => {
        if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
        if (a.trend !== 'increasing' && b.trend === 'increasing') return 1
        const costA = a.restock_cost ?? -Infinity
        const costB = b.restock_cost ?? -Infinity
        return costB - costA
      })

      return result
    })

    // Initialize selectedSkus when candidates first populate
    watch(candidates, (newCandidates) => {
      if (selectedSkus.value.size === 0 && newCandidates.length > 0) {
        selectedSkus.value = new Set(newCandidates.map(c => c.sku))
      }
    })

    const totalSelectedCost = computed(() => {
      return candidates.value.reduce((sum, c) => {
        if (selectedSkus.value.has(c.sku) && c.restock_cost !== null) {
          return sum + c.restock_cost
        }
        return sum
      }, 0)
    })

    const progressPercent = computed(() => {
      if (budget.value === 0) return totalSelectedCost.value > 0 ? 100 : 0
      return Math.min((totalSelectedCost.value / budget.value) * 100, 100)
    })

    const progressBarClass = computed(() => {
      const ratio = budget.value > 0 ? totalSelectedCost.value / budget.value : 0
      if (totalSelectedCost.value > budget.value) return 'progress-bar--over'
      if (ratio >= 0.8) return 'progress-bar--warning'
      return 'progress-bar--ok'
    })

    const isPlaceOrderDisabled = computed(() => {
      if (selectedSkus.value.size === 0) return true
      if (totalSelectedCost.value > budget.value) return true
      // Disabled if all selected items have null restock_cost
      const hasValidSelected = candidates.value.some(
        c => selectedSkus.value.has(c.sku) && c.restock_cost !== null
      )
      return !hasValidSelected
    })

    const toggleSelection = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const selectAll = () => {
      selectedSkus.value = new Set(candidates.value.map(c => c.sku))
    }

    const deselectAll = () => {
      selectedSkus.value = new Set()
    }

    const formatCurrency = (value) => {
      if (value === null || value === undefined) return '—'
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const placeOrder = async () => {
      if (isPlaceOrderDisabled.value || placingOrder.value) return

      placingOrder.value = true
      successOrder.value = null

      try {
        const items = candidates.value
          .filter(c => selectedSkus.value.has(c.sku) && c.restock_cost !== null)
          .map(c => ({
            sku: c.sku,
            name: c.name,
            quantity: c.forecasted_demand,
            unit_cost: c.unit_cost,
            item_cost: c.restock_cost
          }))

        const payload = {
          items,
          total_value: totalSelectedCost.value
        }

        const order = await api.createRestockingOrder(payload)
        successOrder.value = order

        // Reset selection and re-initialize from candidates
        selectedSkus.value = new Set(candidates.value.map(c => c.sku))
      } catch (err) {
        error.value = 'Failed to place restocking order: ' + err.message
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      placingOrder,
      successOrder,
      budget,
      candidates,
      selectedSkus,
      totalSelectedCost,
      progressPercent,
      progressBarClass,
      isPlaceOrderDisabled,
      toggleSelection,
      selectAll,
      deselectAll,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1.25rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
  flex-shrink: 0;
}

.budget-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  accent-color: #3b82f6;
}

.budget-summary-row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
  margin-bottom: 1rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-stat-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stat-value.over-budget {
  color: #dc2626;
}

/* Progress bar */
.progress-bar-container {
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.375rem;
}

.progress-bar {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background-color 0.3s ease;
}

.progress-bar--ok {
  background: #10b981;
}

.progress-bar--warning {
  background: #f59e0b;
}

.progress-bar--over {
  background: #ef4444;
}

.progress-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.progress-current {
  font-weight: 600;
  color: #0f172a;
}

/* Warning / success banners */
.warning-banner {
  background: #fffbeb;
  border: 1px solid #fcd34d;
  color: #92400e;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

.success-banner {
  background: #f0fdf4;
  border: 1px solid #86efac;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

/* Table actions */
.table-actions {
  display: flex;
  gap: 0.5rem;
}

.btn-text {
  background: none;
  border: none;
  padding: 0.375rem 0.625rem;
  font-size: 0.813rem;
  font-weight: 500;
  color: #3b82f6;
  cursor: pointer;
  border-radius: 4px;
  transition: background 0.15s;
}

.btn-text:hover {
  background: #eff6ff;
}

/* Table columns */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 44px;
}

.col-sku {
  width: 120px;
}

.col-name {
  /* flexible */
}

.col-trend {
  width: 110px;
}

.col-demand {
  width: 140px;
  text-align: right;
}

.col-cost {
  width: 110px;
  text-align: right;
}

.col-total {
  width: 130px;
  text-align: right;
}

th.col-demand,
th.col-cost,
th.col-total {
  text-align: right;
}

.row-checkbox {
  cursor: pointer;
  width: 16px;
  height: 16px;
  accent-color: #3b82f6;
}

.row-selected {
  background: #f0f9ff;
}

.sku-text {
  font-family: 'SF Mono', 'Fira Code', 'Cascadia Code', Consolas, monospace;
  font-size: 0.813rem;
  color: #0f172a;
}

.muted {
  color: #94a3b8;
}

/* Empty state */
.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Table footer */
.table-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 1rem;
  margin-top: 0.5rem;
  border-top: 1px solid #e2e8f0;
}

.footer-total {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.footer-total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
}

.footer-total-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.footer-total-value.over-budget {
  color: #dc2626;
}

/* Primary button */
.btn-primary {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.625rem 1.375rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
