# @pixiedev/vue-virtual-scroll

Vue virtual list component for performant scrolling for vuejs 3.


```vue
<script setup>
import VirtualScroll from "@pixiedev/vue-virtual-scroll"

/* 
 * data: Array/list of the objects (must have any key field)
 * page-mode: boolean
 * blockHeight: fixed item height (height + margin + padding) else item have height property
 */

const dataList = ref([])
</script>

<template>
    <VirtualScroll :data="dataList" :page-mode="true" :blockHeight="100" key="id">
      <template v-slot="{ item }">
        <div class="note">
          {{ item.content }}
        </div>
      </template>
    </VirtualScroll>
</template>

<style>
.note {
  height: 86px;
  margin: 2px 0;
}
</style>
```