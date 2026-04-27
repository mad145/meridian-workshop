<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.subtitle') }}</p>
    </div>

    <!-- Budget input -->
    <div class="budget-bar">
      <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
      <div class="budget-input-wrap">
        <span class="currency-symbol">{{ currentCurrency === 'JPY' ? '¥' : '$' }}</span>
        <input
          v-model="budget"
          type="number"
          min="0"
          class="budget-input"
          :placeholder="t('restocking.budgetPlaceholder')"
        />
      </div>
      <div v-if="active.length > 0" class="budget-summary">
        {{ withinBudget.length }} items · est. {{ formatCurrency(totalCost) }}
        <span v-if="budgetValue < Infinity"> · {{ formatCurrency(remaining) }} remaining</span>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="active.length === 0" class="empty-state">
      {{ t('restocking.noRecommendations') }}
    </div>
    <div v-else class="table-container">
      <table class="restocking-table">
        <thead>
          <tr>
            <th>{{ t('restocking.columns.priority') }}</th>
            <th>{{ t('restocking.columns.sku') }}</th>
            <th>{{ t('restocking.columns.name') }}</th>
            <th>{{ t('restocking.columns.warehouse') }}</th>
            <th>{{ t('restocking.columns.category') }}</th>
            <th class="num">{{ t('restocking.columns.onHand') }}</th>
            <th class="num">{{ t('restocking.columns.reorderPoint') }}</th>
            <th>{{ t('restocking.columns.trend') }}</th>
            <th class="num">{{ t('restocking.columns.qtyToOrder') }}</th>
            <th class="num">{{ t('restocking.columns.estCost') }}</th>
            <th>{{ t('restocking.columns.action') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="item in active"
            :key="item.sku"
            :class="{ 'over-budget': !withinBudgetSet.has(item.sku) && budgetValue < Infinity }"
          >
            <td>
              <span :class="priorityClass(item.priority)">
                {{ t('restocking.priority.' + item.priority) }}
              </span>
            </td>
            <td class="mono">{{ item.sku }}</td>
            <td>{{ item.name }}</td>
            <td>{{ translateWarehouse(item.warehouse) }}</td>
            <td>{{ item.category }}</td>
            <td class="num">{{ item.quantity_on_hand }}</td>
            <td class="num">{{ item.reorder_point }}</td>
            <td>
              <span :class="trendClass(item.trend)">
                {{ t('restocking.trend.' + item.trend) }}
              </span>
            </td>
            <td class="num">{{ item.quantity_to_order }}</td>
            <td class="num">{{ formatCurrency(item.estimated_cost) }}</td>
            <td>
              <button class="dismiss-btn" @click="dismiss(item.sku)">
                {{ t('restocking.dismiss') }}
              </button>
            </td>
          </tr>
        </tbody>
        <tfoot>
          <tr class="footer-row">
            <td colspan="9" class="footer-label">
              {{ t('restocking.footer.withinBudget', { count: withinBudget.length }) }}
            </td>
            <td class="num footer-total">{{ formatCurrency(totalCost) }}</td>
            <td></td>
          </tr>
        </tfoot>
      </table>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateWarehouse } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref('')
    const dismissed = ref(new Set())

    const budgetValue = computed(() => {
      const v = parseFloat(budget.value)
      return isNaN(v) || v <= 0 ? Infinity : v
    })

    const active = computed(() =>
      recommendations.value.filter(r => !dismissed.value.has(r.sku))
    )

    const withinBudget = computed(() => {
      if (budgetValue.value === Infinity) return active.value
      let running = 0
      return active.value.filter(r => {
        if (running + r.estimated_cost <= budgetValue.value) {
          running += r.estimated_cost
          return true
        }
        return false
      })
    })

    const withinBudgetSet = computed(() => new Set(withinBudget.value.map(r => r.sku)))

    const totalCost = computed(() =>
      withinBudget.value.reduce((sum, r) => sum + r.estimated_cost, 0)
    )

    const remaining = computed(() => budgetValue.value - totalCost.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        dismissed.value = new Set()
        recommendations.value = await api.getRestockingRecommendations(getCurrentFilters())
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], loadData)
    onMounted(loadData)

    const dismiss = (sku) => {
      dismissed.value = new Set([...dismissed.value, sku])
    }

    const formatCurrency = (num) => {
      const locale = currentCurrency.value === 'JPY' ? 'ja-JP' : 'en-US'
      return num.toLocaleString(locale, { style: 'currency', currency: currentCurrency.value, maximumFractionDigits: 0 })
    }

    const priorityClass = (priority) => {
      return { critical: 'badge danger', high: 'badge warning', medium: 'badge info' }[priority] || 'badge'
    }

    const trendClass = (trend) => {
      return trend === 'increasing' ? 'badge danger' : trend === 'decreasing' ? 'badge success' : 'badge neutral'
    }

    return {
      t, currentCurrency, translateWarehouse,
      loading, error, budget,
      active, withinBudget, withinBudgetSet,
      totalCost, remaining, budgetValue,
      dismiss, formatCurrency, priorityClass, trendClass
    }
  }
}
</script>

<style scoped>
.restocking { padding: 0; }

.budget-bar {
  display: flex;
  align-items: center;
  gap: 16px;
  background: white;
  border-radius: 12px;
  padding: 16px 24px;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  flex-wrap: wrap;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
  white-space: nowrap;
}

.budget-input-wrap {
  display: flex;
  align-items: center;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  overflow: hidden;
}

.currency-symbol {
  padding: 8px 10px;
  background: #f3f4f6;
  color: #6b7280;
  font-size: 0.875rem;
  border-right: 1px solid #d1d5db;
}

.budget-input {
  border: none;
  outline: none;
  padding: 8px 12px;
  font-size: 0.875rem;
  width: 160px;
}

.budget-summary {
  font-size: 0.875rem;
  color: #6b7280;
  margin-left: auto;
}

.table-container { overflow-x: auto; }

.restocking-table {
  width: 100%;
  border-collapse: collapse;
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.restocking-table th {
  background: #f8fafc;
  padding: 0.75rem 1rem;
  text-align: left;
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 2px solid #e2e8f0;
  white-space: nowrap;
}

.restocking-table th.num,
.restocking-table td.num { text-align: right; }

.restocking-table td {
  padding: 0.75rem 1rem;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.875rem;
  color: #374151;
  vertical-align: middle;
}

.restocking-table tr:hover td { background: #f8fafc; }

.restocking-table tr.over-budget td { opacity: 0.4; }

.mono { font-family: 'Consolas', monospace; font-size: 0.8rem; color: #64748b; }

tfoot .footer-row td {
  padding: 0.75rem 1rem;
  background: #f8fafc;
  font-weight: 600;
  font-size: 0.875rem;
  color: #374151;
  border-top: 2px solid #e2e8f0;
}

.footer-total { color: #0f172a; font-size: 1rem; }

.badge {
  display: inline-block;
  padding: 0.2rem 0.6rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
  white-space: nowrap;
}
.badge.danger  { background: #fee2e2; color: #991b1b; }
.badge.warning { background: #fef3c7; color: #92400e; }
.badge.info    { background: #dbeafe; color: #1e40af; }
.badge.success { background: #dcfce7; color: #166534; }
.badge.neutral { background: #f1f5f9; color: #475569; }

.dismiss-btn {
  background: none;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 0.75rem;
  color: #64748b;
  cursor: pointer;
}
.dismiss-btn:hover { background: #fee2e2; color: #991b1b; border-color: #fca5a5; }

.loading, .empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.error {
  background: #fee2e2;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
}
</style>
