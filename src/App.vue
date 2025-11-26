<script setup>
import { ref, computed, onMounted } from 'vue'
import Lessonlist from './Lessonlist.vue'

const API_BASE = "https://lesson-store-backend.onrender.com" // ← your Render URL

// ------------ STATE ------------

// lessons from backend
const allLessons = ref([])
const loadingLessons = ref(true)
const fetchError = ref(null)

// cart + view
const cart = ref([])
const showingLessons = ref(true)

// sorting
const sortKey = ref('topic')
const sortOrder = ref('asc')

// checkout form
const customerName = ref('')
const customerPhone = ref('')
const orderMessage = ref('')

// search
const searchQuery = ref('')
const searchLoading = ref(false)
const searchError = ref(null)

// ------------ COMPUTED ------------

const cartCount = computed(() => cart.value.length)

const cartTotal = computed(() =>
  cart.value.reduce((sum, lesson) => sum + (lesson.price || 0), 0)
)

const nameValid = computed(() =>
  /^[A-Za-z\s]+$/.test(customerName.value)
)

const phoneValid = computed(() =>
  /^[0-9]+$/.test(customerPhone.value)
)

const canCheckout = computed(() =>
  cart.value.length > 0 && nameValid.value && phoneValid.value
)

const sortedLessons = computed(() => {
  const arr = [...allLessons.value]

  arr.sort((a, b) => {
    let A = a[sortKey.value]
    let B = b[sortKey.value]

    if (typeof A === 'string') A = A.toLowerCase()
    if (typeof B === 'string') B = B.toLowerCase()

    if (A < B) return sortOrder.value === 'asc' ? -1 : 1
    if (A > B) return sortOrder.value === 'asc' ? 1 : -1
    return 0
  })

  return arr
})

// ------------ METHODS ------------

function toggleView () {
  if (showingLessons.value && cartCount.value === 0) return
  showingLessons.value = !showingLessons.value
}

function addToCart (lesson) {
  if (!lesson || lesson.space <= 0) return
  cart.value.push(lesson)
  lesson.space -= 1
  orderMessage.value = ''
}

function removeFromCart (index) {
  const lesson = cart.value[index]
  if (lesson) {
    lesson.space += 1
  }
  cart.value.splice(index, 1)

  if (cart.value.length === 0) {
    showingLessons.value = true
  }
}

async function loadLessons () {
  loadingLessons.value = true
  fetchError.value = null
  try {
    const res = await fetch(`${API_BASE}/lessons`)
    if (!res.ok) throw new Error('HTTP ' + res.status)
    allLessons.value = await res.json()
    console.log('LESSONS FROM API:', allLessons.value)
  } catch (err) {
    console.error('Error fetching lessons:', err)
    fetchError.value = err.message
  } finally {
    loadingLessons.value = false
  }
}

async function performSearch () {
  searchError.value = null

  // empty search → reload all lessons
  if (!searchQuery.value.trim()) {
    await loadLessons()
    return
  }

  searchLoading.value = true
  try {
    const q = encodeURIComponent(searchQuery.value.trim())
    const res = await fetch(`${API_BASE}/search?query=${q}`)
    if (!res.ok) throw new Error('HTTP ' + res.status)
    const results = await res.json()
    allLessons.value = results
  } catch (err) {
    console.error('Search error:', err)
    searchError.value = err.message
  } finally {
    searchLoading.value = false
  }
}

async function checkout () {
  if (!canCheckout.value) return

  try {
    // 1) POST order
    const orderPayload = {
      name: customerName.value,
      phone: customerPhone.value,
      items: cart.value.map(l => ({
        lessonId: l._id,
        topic: l.topic,
        price: l.price
      })),
      total: cartTotal.value,
      createdAt: new Date().toISOString()
    }

    const res = await fetch(`${API_BASE}/orders`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(orderPayload)
    })

    if (!res.ok) throw new Error('Order failed with status ' + res.status)

    // 2) PUT updated spaces for each lesson
    for (const lesson of cart.value) {
      await fetch(`${API_BASE}/lessons/${lesson._id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ space: lesson.space })
      })
    }

    orderMessage.value = 'Your order has been submitted!'
    cart.value = []
    customerName.value = ''
    customerPhone.value = ''
    showingLessons.value = true
  } catch (err) {
    console.error('Checkout error:', err)
    orderMessage.value = 'Error submitting order: ' + err.message
  }
}

onMounted(loadLessons)
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <h1 class="brand">Lesson Store</h1>

      <button
        class="cart-toggle"
        :disabled="cartCount === 0 && showingLessons"
        @click="toggleView"
      >
        <span v-if="showingLessons">
          View cart ({{ cartCount }})
        </span>
        <span v-else>
          Back to lessons
        </span>
      </button>
    </header>

    <!-- LESSONS PAGE -->
    <section v-if="showingLessons" class="lessons-page">
      <h2 class="page-title">Available Lessons</h2>

      <p v-if="loadingLessons" class="status">Loading lessons...</p>
      <p v-else-if="fetchError" class="status error">
        Error: {{ fetchError }}
      </p>

      <div v-else>
        <!-- SEARCH BAR -->
        <div class="search-bar">
          <input
            v-model="searchQuery"
            @input="performSearch"
            type="text"
            placeholder="Search by subject, location, price, or spaces..."
          />
          <span
            v-if="searchLoading"
            class="search-status"
          >
            Searching...
          </span>
        </div>

        <p v-if="searchError" class="status error">
          Search error: {{ searchError }}
        </p>

        <!-- Sorting controls -->
        <div class="sort-bar">
          <label>Sort by:</label>
          <select v-model="sortKey">
            <option value="topic">Subject</option>
            <option value="location">Location</option>
            <option value="price">Price</option>
            <option value="space">Spaces</option>
          </select>

          <label>Order:</label>
          <select v-model="sortOrder">
            <option value="asc">Ascending</option>
            <option value="desc">Descending</option>
          </select>
        </div>

        <!-- Lesson list component -->
        <Lessonlist
          :lessons="sortedLessons"
          :add-to-cart="addToCart"
        />
      </div>
    </section>

    <!-- CART PAGE -->
    <section v-else class="cart-page">
      <h2>Your Cart</h2>

      <p v-if="cartCount === 0">
        Your cart is empty.
      </p>

      <div v-else>
        <ul class="cart-list">
          <li
            v-for="(lesson, index) in cart"
            :key="index"
            class="cart-item"
          >
            <div>
              <strong>{{ lesson.topic }}</strong>
              <span class="cart-location">
                ({{ lesson.location }})
              </span>
            </div>
            <div class="cart-right">
              <span class="cart-price">£{{ lesson.price }}</span>
              <button
                class="remove-btn"
                @click="removeFromCart(index)"
              >
                Remove
              </button>
            </div>
          </li>
        </ul>

        <div class="cart-summary">
          <p>Items: <strong>{{ cartCount }}</strong></p>
          <p>Total: <strong>£{{ cartTotal }}</strong></p>
        </div>
      </div>

      <!-- CHECKOUT FORM -->
      <div class="checkout-box">
        <h3>Checkout</h3>

        <form @submit.prevent="checkout">
          <div class="field">
            <label>Name</label>
            <input v-model="customerName" type="text" />
            <small
              v-if="customerName && !nameValid"
              class="error-text"
            >
              Name must contain letters and spaces only.
            </small>
          </div>

          <div class="field">
            <label>Phone</label>
            <input v-model="customerPhone" type="text" />
            <small
              v-if="customerPhone && !phoneValid"
              class="error-text"
            >
              Phone must contain numbers only.
            </small>
          </div>

          <button
            type="submit"
            class="checkout-btn"
            :disabled="!canCheckout"
          >
            Submit Order
          </button>
        </form>

        <p
          v-if="orderMessage"
          class="order-message"
        >
          {{ orderMessage }}
        </p>
      </div>
    </section>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: #0f172a;
  color: #e5e7eb;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
    sans-serif;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.25rem 2rem;
  background: #020617;
}

.brand {
  font-size: 1.8rem;
  margin: 0;
}

.cart-toggle {
  border: none;
  border-radius: 999px;
  padding: 0.5rem 1.2rem;
  font-weight: 600;
  cursor: pointer;
  background: #22c55e;
  color: #022c22;
}

.cart-toggle:disabled {
  background: #374151;
  color: #9ca3af;
  cursor: not-allowed;
}

.lessons-page,
.cart-page {
  padding: 1.5rem 2rem 2rem;
}

.page-title {
  margin-bottom: 1rem;
}

.status {
  margin-top: 0.5rem;
}

.error {
  color: #f97373;
}

/* SEARCH */
.search-bar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin: 0.5rem 0 0.75rem;
}

.search-bar input {
  flex: 1;
  padding: 0.45rem 0.7rem;
  border-radius: 0.5rem;
  border: none;
}

.search-status {
  font-size: 0.9rem;
  opacity: 0.8;
}

/* SORT */
.sort-bar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin: 0.5rem 0 1.5rem;
}

.sort-bar select {
  padding: 0.4rem 0.6rem;
  border-radius: 0.5rem;
  border: none;
}

/* CART */
.cart-list {
  list-style: none;
  padding: 0;
  margin: 1rem 0;
}

.cart-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #111827;
  border-radius: 0.75rem;
  padding: 0.75rem 1rem;
  margin-bottom: 0.5rem;
  border: 1px solid #1f2937;
}

.cart-location {
  color: #9ca3af;
  margin-left: 0.25rem;
}

.cart-right {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.cart-price {
  font-weight: 600;
}

.remove-btn {
  border: none;
  border-radius: 999px;
  padding: 0.3rem 0.8rem;
  background: #f97373;
  color: #111827;
  cursor: pointer;
  font-weight: 600;
}

.cart-summary {
  margin-bottom: 1rem;
}

/* CHECKOUT */
.checkout-box {
  margin-top: 1.5rem;
  padding: 1rem 1.2rem;
  background: #020617;
  border-radius: 0.75rem;
  border: 1px solid #1f2937;
}

.field {
  margin-bottom: 0.75rem;
  display: flex;
  flex-direction: column;
}

.field input {
  padding: 0.4rem 0.6rem;
  border-radius: 0.5rem;
  border: none;
  margin-top: 0.25rem;
}

.error-text {
  color: #f97373;
  font-size: 0.8rem;
}

.checkout-btn {
  margin-top: 0.5rem;
  border: none;
  border-radius: 999px;
  padding: 0.6rem 1.2rem;
  background: #22c55e;
  color: #022c22;
  font-weight: 600;
  cursor: pointer;
}

.checkout-btn:disabled {
  background: #4b5563;
  color: #e5e7eb;
  cursor: not-allowed;
}

.order-message {
  margin-top: 0.75rem;
}
</style>
