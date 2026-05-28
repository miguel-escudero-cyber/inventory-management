<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts and your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submittedOrder" class="success-banner">
        <span>
          Restocking order <strong>{{ submittedOrder.order_number }}</strong> placed successfully.
          Expected delivery in 14 days. View in the Orders tab.
        </span>
        <button class="success-close" @click="submittedOrder = null" aria-label="Close">&times;</button>
      </div>

      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-value">{{ formatCurrency(budget) }}</div>
        <div class="slider-container">
          <input
            type="range"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span>$50,000</span>
          </div>
        </div>
        <div class="budget-summary">{{ recommendedItems.length }} items recommended within budget</div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items
            <span class="count-badge">{{ recommendedItems.length }}</span>
          </h3>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Current Demand</th>
                <th>Forecasted Demand</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', item.trend === 'increasing' ? 'increasing' : 'stable']">
                    {{ item.trend }}
                  </span>
                </td>
                <td>{{ item.current_demand }}</td>
                <td><strong>{{ item.forecasted_demand }}</strong></td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.forecasted_demand * item.unit_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Order Summary Footer -->
        <div class="order-footer">
          <div class="order-totals">
            <div>
              <div class="total-label">Total Order Cost</div>
              <div class="total-value">{{ formatCurrency(totalCost) }}</div>
            </div>
            <div>
              <div class="total-label">Budget Remaining</div>
              <div :class="['total-value', 'remaining-value', budgetRemaining >= 0 ? 'positive' : '']">
                {{ formatCurrency(budgetRemaining) }}
              </div>
            </div>
          </div>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
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
    const { currentCurrency } = useI18n()
    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(10000)
    const submitting = ref(false)
    const submittedOrder = ref(null)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // All eligible candidates: non-decreasing trend, joined with inventory for unit_cost
    const allCandidates = computed(() => {
      return demandForecasts.value
        .filter(d => d.trend !== 'decreasing')
        .map(d => {
          const inv = inventoryItems.value.find(i => i.sku === d.item_sku)
          if (!inv) return null
          return { ...d, unit_cost: inv.unit_cost, warehouse: inv.warehouse }
        })
        .filter(Boolean)
        .sort((a, b) => {
          // increasing before stable
          if (a.trend !== b.trend) return a.trend === 'increasing' ? -1 : 1
          // within same trend, higher demand gap first
          const gapA = a.forecasted_demand - a.current_demand
          const gapB = b.forecasted_demand - b.current_demand
          return gapB - gapA
        })
    })

    // Greedy: include items while budget allows
    const recommendedItems = computed(() => {
      let remaining = budget.value
      return allCandidates.value.filter(item => {
        const cost = item.forecasted_demand * item.unit_cost
        if (cost <= remaining) { remaining -= cost; return true }
        return false
      })
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.forecasted_demand * item.unit_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const formatCurrency = (value) => {
      const symbol = currentCurrency.value === 'JPY' ? '¥' : '$'
      return symbol + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0) return
      try {
        submitting.value = true
        error.value = null
        // Determine warehouse from first item (most items likely share warehouse)
        const warehouse = recommendedItems.value[0].warehouse || 'San Francisco'
        const orderData = {
          warehouse,
          customer: 'Internal Restocking',
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_price: item.unit_cost
          }))
        }
        const order = await api.createRestockOrder(orderData)
        submittedOrder.value = order
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      submitting,
      submittedOrder,
      recommendedItems,
      totalCost,
      budgetRemaining,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #065f46;
  font-size: 0.938rem;
}

.success-close {
  background: none;
  border: none;
  cursor: pointer;
  color: #065f46;
  font-size: 1.25rem;
  line-height: 1;
  padding: 0 0 0 1rem;
  flex-shrink: 0;
}

.success-close:hover {
  opacity: 0.7;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.5rem;
}

.slider-container {
  width: 100%;
  margin: 1rem 0;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
  display: block;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.75rem;
}

.count-badge {
  display: inline-block;
  background: #dbeafe;
  color: #1e40af;
  font-size: 0.813rem;
  font-weight: 600;
  border-radius: 12px;
  padding: 0.125rem 0.625rem;
  margin-left: 0.5rem;
  vertical-align: middle;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 0;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.order-totals {
  display: flex;
  gap: 2rem;
}

.total-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.total-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.remaining-value.positive {
  color: #059669;
}

.place-order-btn {
  background: #3b82f6;
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.938rem;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #2563eb;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
