<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetCard') }}</h3>
        </div>
        <div class="budget-layout">
          <div class="budget-left">
            <div class="budget-display">{{ formatCurrency(budget) }}</div>
            <div class="budget-slider-section">
              <input
                type="range"
                :min="0"
                :max="MAX_BUDGET"
                :step="BUDGET_STEP"
                v-model.number="budget"
              />
              <div class="range-labels">
                <span>{{ currencySymbol }}0</span>
                <span>{{ formatCurrency(MAX_BUDGET) }}</span>
              </div>
              <p class="budget-helper-text">{{ t('restocking.budgetHelper') }}</p>
            </div>
          </div>
          <div class="budget-stats">
            <div class="stat-card info">
              <div class="stat-label">{{ t('restocking.itemsRecommended') }}</div>
              <div class="stat-value">{{ recommendedItems.length }}</div>
            </div>
            <div class="stat-card">
              <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
              <div class="stat-value budget-remaining">{{ formatCurrency(remainingBudget) }}</div>
            </div>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <div class="card-title-group">
            <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendedItems.length }})</h3>
            <p class="mode-caption">{{ t('restocking.subtitle') }}</p>
          </div>
        </div>
        <div v-if="recommendedItems.length === 0">
          <p class="empty-state">{{ emptyStateMessage }}</p>
        </div>
        <div v-else class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-name">{{ t('restocking.table.name') }}</th>
                <th class="col-category">{{ t('restocking.table.category') }}</th>
                <th class="col-num">{{ t('restocking.table.onHand') }}</th>
                <th class="col-num">{{ t('restocking.table.target') }}</th>
                <th class="col-num">{{ t('restocking.table.shortfall') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.suggestedQty') }}</th>
                <th class="col-num">{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td class="col-sku"><strong>{{ item.sku }}</strong></td>
                <td class="col-name">{{ item.name }}</td>
                <td class="col-category">{{ item.category }}</td>
                <td class="col-num">{{ item.onHand }}</td>
                <td class="col-num">{{ item.target }}</td>
                <td class="col-num">{{ item.shortfall }}</td>
                <td class="col-num">{{ formatCurrency(item.unitCost) }}</td>
                <td class="col-num"><strong>{{ item.suggestedQty }}</strong></td>
                <td class="col-num"><strong>{{ formatCurrency(item.lineTotal) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="action-row">
        <div v-if="confirmation" class="confirmation-banner">
          <span>{{ t('restocking.orderPlaced', { orderNumber: confirmation.order_number, total: formatCurrency(confirmation.total_value) }) }}</span>
          <router-link to="/orders" class="view-orders-link">{{ t('restocking.viewInOrders') }}</router-link>
        </div>
        <button
          class="place-order-btn"
          :disabled="recommendedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const confirmation = ref(null)

    // Unified row shape: { sku, name, category, onHand, target, shortfall, unitCost, fullCost }
    // `target` = max(forecasted_demand, reorder_point) — the level we're restocking toward.
    // `shortfall` = max(forecastGap, reorderGap, 0) — drives ranking.
    const candidates = ref([])

    const MAX_BUDGET = 1000000
    const BUDGET_STEP = 1000
    const budget = ref(500000)

    const recommendedItems = computed(() => {
      // Scale every item's quantity proportionally to the slider. At budget == baseCost each item
      // gets exactly its shortfall; higher budgets build buffer uniformly across all items, lower
      // budgets shrink everyone together. Floor keeps spend under budget.
      const baseCost = candidates.value.reduce((s, i) => s + i.fullCost, 0)
      if (baseCost === 0 || budget.value === 0) return []
      const scale = budget.value / baseCost
      const picks = []
      for (const item of candidates.value) {
        const qty = Math.floor(scale * item.shortfall)
        if (qty > 0) {
          picks.push({ ...item, suggestedQty: qty, lineTotal: qty * item.unitCost })
        }
      }
      picks.sort((a, b) => b.shortfall - a.shortfall)
      return picks
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((s, i) => s + i.lineTotal, 0)
    )
    const remainingBudget = computed(() => budget.value - totalCost.value)

    const emptyStateMessage = computed(() =>
      candidates.value.length === 0 ? t('restocking.noGap') : t('restocking.empty')
    )

    const formatCurrency = (n) => `${currencySymbol.value}${Math.round(n).toLocaleString()}`

    const loadData = async () => {
      try {
        loading.value = true
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        const forecastBySku = Object.fromEntries(forecasts.map(f => [f.item_sku, f]))

        const rows = []
        for (const inv of inventory) {
          const f = forecastBySku[inv.sku]
          const forecastGap = f ? f.forecasted_demand - inv.quantity_on_hand : 0
          const reorderGap = inv.reorder_point - inv.quantity_on_hand
          const shortfall = Math.max(forecastGap, reorderGap, 0)
          if (shortfall <= 0) continue
          const target = Math.max(f ? f.forecasted_demand : 0, inv.reorder_point)
          rows.push({
            sku: inv.sku,
            name: inv.name,
            category: inv.category,
            onHand: inv.quantity_on_hand,
            target,
            shortfall,
            unitCost: inv.unit_cost,
            fullCost: shortfall * inv.unit_cost
          })
        }
        candidates.value = rows
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      confirmation.value = null
      try {
        const payload = {
          customer: 'Internal Restocking',
          items: recommendedItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.suggestedQty,
            unit_price: i.unitCost
          }))
        }
        const order = await api.createOrder(payload)
        confirmation.value = order
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t, currencySymbol, loading, error, submitting, confirmation,
      candidates,
      budget, MAX_BUDGET, BUDGET_STEP,
      recommendedItems, totalCost, remainingBudget, emptyStateMessage,
      formatCurrency, placeOrder
    }
  }
}
</script>

<style scoped>
.budget-layout {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 2rem;
  align-items: start;
}

.budget-left {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  font-variant-numeric: tabular-nums;
}

.budget-slider-section {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

input[type="range"] {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.budget-helper-text {
  font-size: 0.813rem;
  color: #94a3b8;
  margin-top: 0.25rem;
}

.budget-stats {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  min-width: 220px;
}

.budget-remaining {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.card-title-group {
  display: flex;
  flex-direction: column;
}

.mode-caption {
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-sku {
  width: 110px;
}

.col-name {
  width: 200px;
}

.col-category {
  width: 140px;
}

.col-num {
  width: 100px;
}

.col-lead-time {
  width: 110px;
}

.empty-state {
  text-align: center;
  color: #94a3b8;
  padding: 2rem;
  font-size: 0.938rem;
}

.action-row {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 1rem;
  margin-top: 0.5rem;
  margin-bottom: 1.5rem;
}

.place-order-btn {
  padding: 0.75rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.confirmation-banner {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: #d1fae5;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  border: 1px solid #a7f3d0;
  font-size: 0.938rem;
  font-weight: 500;
  width: 100%;
}

.view-orders-link {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
}

.view-orders-link:hover {
  color: #047857;
}
</style>
