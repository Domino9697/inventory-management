<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend and submit restocking orders based on demand forecast and your available budget.</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-control">
        <div class="budget-display">{{ formatCurrencyWhole(budget) }}</div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="250000"
          :step="5000"
          v-model.number="budget"
          @input="onBudgetInput"
        />
        <div class="budget-range-labels">
          <span>$0</span>
          <span>$250,000</span>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">Recommended Items</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Total Cost</div>
          <div class="stat-value">{{ formatCurrencyWhole(totalCost) }}</div>
        </div>
        <div :class="['stat-card', budgetRemaining >= 0 ? 'success' : 'danger']">
          <div class="stat-label">Budget Remaining</div>
          <div class="stat-value">{{ formatCurrencyWhole(budgetRemaining) }}</div>
        </div>
      </div>

      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No restock recommendations within this budget — try increasing the budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Current Stock</th>
                <th>Forecast</th>
                <th>Shortfall</th>
                <th>Recommended Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in recommendations" :key="rec.sku">
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>{{ rec.current_stock.toLocaleString() }}</td>
                <td>{{ rec.forecasted_demand.toLocaleString() }}</td>
                <td>{{ rec.shortfall.toLocaleString() }}</td>
                <td>{{ rec.recommended_quantity.toLocaleString() }}</td>
                <td>{{ formatCurrencyUnit(rec.unit_cost) }}</td>
                <td><strong>{{ formatCurrencyWhole(rec.line_total) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')

    let debounceTimer = null

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_total, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const formatCurrencyWhole = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const formatCurrencyUnit = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 2 })

    const loadRecommendations = async ({ keepMessage = false } = {}) => {
      loading.value = true
      error.value = null
      if (!keepMessage) successMessage.value = ''
      try {
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const onBudgetInput = () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 250)
    }

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      successMessage.value = ''
      try {
        const payload = {
          budget: budget.value,
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.recommended_quantity,
            unit_cost: r.unit_cost
          }))
        }
        const result = await api.submitRestockOrder(payload)
        successMessage.value = `Order ${result.order_number} placed successfully. Max lead time: ${result.max_lead_time_days} days. View it on the Orders tab.`
        await loadRecommendations({ keepMessage: true })
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      budget,
      recommendations,
      loading,
      error,
      submitting,
      successMessage,
      totalCost,
      budgetRemaining,
      formatCurrencyWhole,
      formatCurrencyUnit,
      onBudgetInput,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-control {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5rem 0;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  max-width: 600px;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  width: 100%;
  max-width: 600px;
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
