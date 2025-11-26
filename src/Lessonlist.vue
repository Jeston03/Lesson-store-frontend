<template>
  <div class="page">
    <div class="grid">
      <article
        v-for="lesson in props.lessons"
        :key="lesson._id"
        class="card"
      >
        <header class="card-header">
          <div class="icon">
            {{ lesson.topic ? lesson.topic[0].toUpperCase() : '?' }}
          </div>
          <h2 class="card-title">{{ lesson.topic }}</h2>
        </header>

        <div class="card-body">
          <p>
            <span class="label">Location:</span> {{ lesson.location }}
          </p>
          <p>
            <span class="label">Price:</span> £{{ lesson.price }}
          </p>
          <p
            :class="['spaces', { 'spaces--zero': lesson.space === 0 }]"
          >
            <span class="label">Spaces left:</span> {{ lesson.space }}
          </p>
        </div>

        <footer class="card-footer">
          <button
            class="btn"
            :disabled="lesson.space === 0"
            @click="props.addToCart(lesson)"
          >
            {{ lesson.space === 0 ? 'Sold out' : 'Add to cart' }}
          </button>
        </footer>
      </article>
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  lessons: {
    type: Array,
    default: () => []
  },
  addToCart: {
    type: Function,
    required: true
  }
})
</script>

<style scoped>
.page {
  padding: 0;
}

/* grid of cards */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
  gap: 1.5rem;
}

/* individual card */
.card {
  background: #111827;
  border-radius: 1rem;
  padding: 1.2rem;
  box-shadow: 0 18px 45px rgba(0, 0, 0, 0.5);
  border: 1px solid #1f2937;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

/* header with icon + title */
.card-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.icon {
  width: 40px;
  height: 40px;
  border-radius: 999px;
  background: #020617;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 1.2rem;
}

.card-title {
  font-size: 1.2rem;
  margin: 0;
}

/* body text */
.card-body p {
  margin: 0.25rem 0;
}

.label {
  font-weight: 600;
  color: #9ca3af;
  margin-right: 0.25rem;
}

.spaces {
  margin-top: 0.5rem;
}

.spaces--zero {
  color: #f97373;
}

/* footer button */
.card-footer {
  margin-top: 1rem;
}

.btn {
  width: 100%;
  padding: 0.6rem 0.8rem;
  border-radius: 999px;
  border: none;
  cursor: pointer;
  font-weight: 600;
  background: #22c55e;
  color: #022c22;
  transition: transform 0.1s ease, box-shadow 0.1s ease, opacity 0.1s ease;
}

.btn:hover:enabled {
  transform: translateY(-1px);
  box-shadow: 0 10px 25px rgba(34, 197, 94, 0.4);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background: #4b5563;
  color: #e5e7eb;
}
</style>
