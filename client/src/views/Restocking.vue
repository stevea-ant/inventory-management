<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget slider card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            min="0"
            max="250000"
            step="1000"
          />
          <div class="budget-meta">{{ t('restocking.budgetMax', { max: currencySymbol + '250,000' }) }}</div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }} ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>

        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.item') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.quantity') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.id">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ translateProductName(item.item_name) }}</td>
                <td>
                  <span :class="['badge', item.trend]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td class="col-num">{{ item.quantity }}</td>
                <td class="col-num">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-num"><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="5" class="totals-label">{{ t('restocking.totalCost') }}</td>
                <td class="col-num totals-value"><strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong></td>
              </tr>
              <tr class="remaining-row">
                <td colspan="5" class="totals-label">{{ t('restocking.remainingBudget') }}</td>
                <td class="col-num totals-value remaining"><strong>{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>

        <div class="card-footer">
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>

    </div>

    <!-- Toast notification -->
    <div v-if="toast" class="toast" :class="toast.type">{{ toast.text }}</div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const candidates = ref([])
    const budget = ref(50000)
    const submitting = ref(false)
    const toast = ref(null)
    const loading = ref(true)
    const error = ref(null)

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // Greedy algorithm: rank by demand shortfall (forecasted - current), fill budget
    // Items are included fully or skipped — no partial quantities.
    const recommendations = computed(() => {
      const sorted = [...candidates.value].sort((a, b) =>
        (b.forecasted_demand - b.current_demand) - (a.forecasted_demand - a.current_demand)
      )
      const picked = []
      let remaining = budget.value
      for (const item of sorted) {
        const lineTotal = item.forecasted_demand * item.unit_cost
        if (lineTotal <= remaining) {
          picked.push({ ...item, quantity: item.forecasted_demand, line_total: lineTotal })
          remaining -= lineTotal
        }
      }
      return picked
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((s, r) => s + r.line_total, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadCandidates = async () => {
      loading.value = true
      error.value = null
      try {
        candidates.value = await api.getRestockingCandidates()
      } catch (err) {
        error.value = 'Failed to load restocking candidates'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const items = recommendations.value.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          quantity: r.quantity,
          unit_cost: r.unit_cost
        }))
        const order = await api.createRestockOrder(items)
        toast.value = { type: 'success', text: t('restocking.orderPlaced', { number: order.order_number }) }
        // Clear candidates so recommendations list empties; slider stays put
        candidates.value = []
        setTimeout(() => { toast.value = null }, 4000)
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadCandidates())

    return {
      t,
      candidates,
      budget,
      submitting,
      toast,
      loading,
      error,
      currencySymbol,
      translateProductName,
      recommendations,
      totalCost,
      remainingBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card layout */
.budget-body {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.budget-meta {
  font-size: 0.813rem;
  color: #64748b;
}

/* Slider */
.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  background: #0f172a;
  border-radius: 50%;
  cursor: pointer;
  transition: background 0.15s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #64748b;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #0f172a;
  border-radius: 50%;
  cursor: pointer;
  border: none;
}

.budget-slider::-webkit-slider-runnable-track {
  background: linear-gradient(to right, #0f172a var(--fill, 0%), #e2e8f0 var(--fill, 0%));
  border-radius: 3px;
  height: 6px;
}

/* Table columns */
.restock-table {
  width: 100%;
}

.col-num {
  text-align: right;
  white-space: nowrap;
}

/* Table footer totals */
.totals-row,
.remaining-row {
  background: #f8fafc;
}

.totals-label {
  text-align: right;
  padding-right: 1rem;
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.totals-value {
  font-size: 0.9375rem;
}

.remaining {
  color: #10b981;
}

/* Empty state */
.empty-state {
  padding: 3rem 1.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9375rem;
}

/* Card footer with action button */
.card-footer {
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
}

/* Place order button */
.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #0f172a;
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, opacity 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1e293b;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* Toast */
.toast {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  padding: 1rem 1.5rem;
  border-radius: 10px;
  font-size: 0.9375rem;
  font-weight: 500;
  color: #fff;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
  z-index: 1000;
  animation: slide-in 0.25s ease-out;
}

.toast.success {
  background: #10b981;
}

.toast.error {
  background: #ef4444;
}

@keyframes slide-in {
  from {
    transform: translateY(1rem);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}
</style>
