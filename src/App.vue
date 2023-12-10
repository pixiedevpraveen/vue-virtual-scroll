<script setup lang="ts">
import { ref, onMounted } from "vue";
import VirtualScroll from '../src/components/VirtualScroll.vue'

type Note = {
  id: number
  content: string
}

const dataList = ref<Note[]>([])
const size = ref(100)

function updateList() {
  const d: Note[] = []
  for (let idx = 0; idx < size.value; idx++) {
    d.push({ id: idx, content: `Item ${idx + 1}` })
  }
  dataList.value = d
  console.log(dataList.value);
}

onMounted(() => {
  updateList()
})
</script>

<template>
  <header>
    <h1>Vue Virtual Scroll</h1>
    <div class="options">
      <label for="size-input">Array Size: </label>
      <select name="size" id="size-input" v-model.number="size" @change="updateList">
        <option value="100">100</option>
        <option value="1000">1000</option>
        <option value="10000">10000</option>
        <option value="100000">100000</option>
      </select>
    </div>
  </header>
  <div v-if="dataList.length" class="container">
    <VirtualScroll :data="dataList" :page-mode="true" :blockHeight="100" key="id">
      <template v-slot="{ item }">
        <div class="note">
          {{ item.content }}
        </div>
      </template>
    </VirtualScroll>
  </div>
</template>

<style>
* {
  padding: 0;
  margin: 0;
}

h1 {
  margin: 1rem;
  padding: 1rem;
  text-align: center;
}

.options {
  text-align: center;
}

.container {
  padding-top: 10px 0;
  width: 100vw;
  height: 100%;
  background-color: #ececec;
}

.note {
  height: 86px;
  margin: 2px 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: coral;
}
</style>
