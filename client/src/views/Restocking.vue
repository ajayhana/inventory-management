<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.budgetHelp') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Success state -->
    <div v-else-if="submitted" class="success-panel">
      <div class="success-icon">&#10003;</div>
      <h3 class="success-title">{{ t('restocking.successTitle') }}</h3>
      <p class="success-message">{{ t('restocking.successMessage') }}</p>
      <p v-if="submittedOrder && submittedOrder.order_number" class="success-order-number">
        Order: <strong>{{ submittedOrder.order_number }}</strong>
      </p>
      <div class="success-actions">
        <router-link to="/orders" class="btn-secondary">{{ t('restocking.viewOrders') }}</router-link>
        <button class="btn-primary" @click="resetForm">{{ t('restocking.placeAnother') }}</button>
      </div>
    </div>

    <div v-else>
      <!-- Budget slider card -->
      <div class="card budget-card">
        <div class="budget-header">
          <label class="budget-label">{{ t('restocking.budget') }}</label>
          <div class="budget-value">${{ budget.toLocaleString() }}</div>
        </div>
        <input
          type="range"
          class="budget-slider"
          v-model.number="budget"
          :min="0"
          :max="maxBudget"
          :step="1000"
        />
        <div class="budget-range-labels">
          <span>$0</span>
          <span>${{ maxBudget.toLocaleString() }}</span>
        </div>

        <!-- Summary bar -->
        <div class="budget-summary">
          <div class="budget-stat">
            <div class="budget-stat-value">{{ totalSelected }}</div>
            <div class="budget-stat-label">{{ t('restocking.itemsSelected') }}</div>
          </div>
          <div class="budget-stat">
            <div class="budget-stat-value">${{ budgetUsed.toLocaleString() }}</div>
            <div class="budget-stat-label">{{ t('restocking.budgetUsed') }}</div>
          </div>
          <div class="budget-stat" :class="{ 'stat-remaining': budgetRemaining >= 0 }">
            <div class="budget-stat-value">${{ budgetRemaining.toLocaleString() }}</div>
            <div class="budget-stat-label">{{ t('restocking.remaining') }}</div>
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div v-if="recommendedItems.length === 0" class="empty-state">
        {{ t('restocking.noItems') }}
      </div>
      <div v-else class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.title') }}</h3>
          <span class="warehouse-label">
            {{ t('restocking.warehouse') }}:
            <strong>{{ selectedLocation !== 'all' ? selectedLocation : t('restocking.allWarehouses') }}</strong>
          </span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.include') }}</th>
                <th>{{ t('restocking.sku') }}</th>
                <th>{{ t('restocking.item') }}</th>
                <th>{{ t('restocking.trend') }}</th>
                <th>{{ t('restocking.gapQty') }}</th>
                <th>{{ t('restocking.unitCost') }}</th>
                <th>{{ t('restocking.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendedItems"
                :key="item.sku"
                :class="{ 'row-excluded': !item.included }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="item.included"
                    @change="toggleItem(item.sku)"
                    class="item-checkbox"
                  />
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>{{ item.gap.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>${{ Math.round(item.item_total).toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place order bar -->
      <div class="place-order-bar">
        <button
          class="btn-primary btn-place-order"
          :disabled="totalSelected === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const inventoryItems = ref([])

    // Submission state
    const submitting = ref(false)
    const submitted = ref(false)
    const submittedOrder = ref(null)

    // Manual exclusion set — replaced by reference to trigger reactivity
    const manuallyExcluded = ref(new Set())

    // Join forecasts with inventory by SKU, compute gap and item_total, filter gap > 0
    const enrichedItems = computed(() => {
      const inventoryMap = new Map(inventoryItems.value.map(i => [i.sku, i]))
      return allForecasts.value
        .filter(f => {
          const inv = inventoryMap.get(f.item_sku)
          if (!inv) return false
          const gap = Math.max(0, f.forecasted_demand - f.current_demand)
          return gap > 0
        })
        .map(f => {
          const inv = inventoryMap.get(f.item_sku)
          const gap = Math.max(0, f.forecasted_demand - f.current_demand)
          return {
            sku: f.item_sku,
            name: f.item_name,
            trend: f.trend,
            current_demand: f.current_demand,
            forecasted_demand: f.forecasted_demand,
            gap,
            unit_cost: inv.unit_cost,
            item_total: inv.unit_cost * gap,
            id: f.id
          }
        })
    })

    // Sort: increasing trend first, then stable, then decreasing; within each tier by item_total desc
    const sortedItems = computed(() => {
      return [...enrichedItems.value].sort((a, b) => {
        const trendDiff = (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99)
        if (trendDiff !== 0) return trendDiff
        return b.item_total - a.item_total
      })
    })

    // Max budget is the total cost of all items, rounded up to nearest $10k
    const maxBudget = computed(() => {
      const total = enrichedItems.value.reduce((sum, i) => sum + i.item_total, 0)
      return Math.ceil(total / 10000) * 10000 || 100000
    })

    const budget = ref(0)

    // When maxBudget changes (e.g. after data loads), reset budget to half
    watch(maxBudget, (val) => {
      budget.value = Math.round(val / 2)
    }, { immediate: true })

    // Greedy auto-selection within budget, respecting manual exclusions
    const recommendedItems = computed(() => {
      let running = 0
      return sortedItems.value.map(item => {
        const fits = running + item.item_total <= budget.value
        if (fits && !manuallyExcluded.value.has(item.sku)) {
          running += item.item_total
          return { ...item, included: true }
        }
        return { ...item, included: false }
      })
    })

    const toggleItem = (sku) => {
      // Replace the Set reference so Vue picks up the change
      const excluded = new Set(manuallyExcluded.value)
      if (excluded.has(sku)) {
        excluded.delete(sku)
      } else {
        excluded.add(sku)
      }
      manuallyExcluded.value = excluded
    }

    // Summary stats
    const selectedItems = computed(() => recommendedItems.value.filter(i => i.included))
    const totalSelected = computed(() => selectedItems.value.length)
    const budgetUsed = computed(() => selectedItems.value.reduce((s, i) => s + i.item_total, 0))
    const budgetRemaining = computed(() => budget.value - budgetUsed.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const filters = getCurrentFilters()

        // Load both in parallel, same as Demand.vue
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])

        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Reload when warehouse or category filter changes
    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      try {
        submitting.value = true
        // Use selected warehouse, default to San Francisco if "all"
        const warehouse = selectedLocation.value !== 'all' ? selectedLocation.value : 'San Francisco'
        const items = selectedItems.value.map(i => ({
          sku: i.sku,
          name: i.name,
          quantity: i.gap,
          unit_price: i.unit_cost
        }))
        submittedOrder.value = await api.createRestockOrder({
          items,
          warehouse,
          total_value: budgetUsed.value
        })
        submitted.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const resetForm = () => {
      submitted.value = false
      submittedOrder.value = null
      manuallyExcluded.value = new Set()
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      selectedLocation,
      budget,
      maxBudget,
      recommendedItems,
      totalSelected,
      budgetUsed,
      budgetRemaining,
      toggleItem,
      submitting,
      submitted,
      submittedOrder,
      placeOrder,
      resetForm
    }
  }
}
</script>

<style scoped>
.restocking {
  /* page container — main-content already provides padding */
}

/* Budget card */
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-header {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 8px;
  border-radius: 4px;
  background: linear-gradient(
    to right,
    #2563eb 0%,
    #2563eb calc(var(--slider-pct, 50%) * 1%),
    #e2e8f0 calc(var(--slider-pct, 50%) * 1%),
    #e2e8f0 100%
  );
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  border: 3px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  cursor: pointer;
  transition: box-shadow 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  border: 3px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.375rem;
  margin-bottom: 1.25rem;
}

/* Budget summary bar */
.budget-summary {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
  border-top: 1px solid #e2e8f0;
  padding-top: 1.25rem;
}

.budget-stat {
  text-align: center;
}

.budget-stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-top: 0.25rem;
}

.stat-remaining .budget-stat-value {
  color: #059669;
}

/* Table card extras */
.warehouse-label {
  font-size: 0.875rem;
  color: #64748b;
}

/* Excluded row styling */
.row-excluded {
  opacity: 0.4;
}

.item-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

/* Place order bar */
.place-order-bar {
  display: flex;
  justify-content: flex-end;
  padding: 1rem 0 2rem;
}

.btn-primary {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.75rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, box-shadow 0.2s;
  text-decoration: none;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.btn-place-order {
  min-width: 220px;
  padding: 0.875rem 2rem;
  font-size: 1rem;
}

.btn-secondary {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.75rem 1.75rem;
  background: white;
  color: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  text-decoration: none;
}

.btn-secondary:hover {
  background: #eff6ff;
}

/* Empty state */
.empty-state {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
  margin-bottom: 1.25rem;
}

/* Success panel */
.success-panel {
  background: white;
  border: 1px solid #d1fae5;
  border-radius: 10px;
  padding: 3rem 2rem;
  text-align: center;
  max-width: 560px;
  margin: 2rem auto;
}

.success-icon {
  width: 72px;
  height: 72px;
  border-radius: 50%;
  background: #d1fae5;
  color: #059669;
  font-size: 2.5rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.5rem;
  line-height: 1;
}

.success-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.75rem;
  letter-spacing: -0.025em;
}

.success-message {
  color: #64748b;
  font-size: 0.938rem;
  margin-bottom: 0.5rem;
}

.success-order-number {
  color: #64748b;
  font-size: 0.875rem;
  margin-bottom: 2rem;
}

.success-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
}
</style>
