<template>
  <div v-on="pageMode ? {} : { scroll: handleScroll }" :style="containerStyle" ref="vb">
    <div :style="{ height: `${offsetTop}px` }">
    </div>
    <div v-for="item in renderList" :style="{ height: `${blockHeight}px` }" :key="`${item[key]}`">
      <slot :item="item">
      </slot>
    </div>
    <div :style="{ height: `${offsetBot}px` }">
    </div>
  </div>
</template>

<script setup lang="ts" generic="T = {id: number | string, height: number }">
import { ref, onMounted, onBeforeMount, onUnmounted, watch, computed, nextTick, type Ref, type StyleValue } from "vue";
// data is required
// height is required if pageMode is set to false
// when blockHeight is specified, the height key in data will be ignored
type PropType = {
  data: T[],
  pageMode: boolean
  height?: number,
  blockHeight: number,
  key: string
} /* & ({
} | {
}) */

const props = withDefaults(defineProps<PropType>(), {
  pageMode: true,
  height: 0,
  blockHeight: 0,
  key: 'id'
})

const vb = ref<HTMLDivElement>() as Ref<HTMLDivElement>
const viewportBegin = ref(0)
const viewportEnd = ref(props.height)
const offsetTop = ref(0)
const offsetBot = ref(0)
const renderList = ref<T[]>([])
const transformedData = ref<number[]>([])
const top = ref(0)

watch(() => props.data, function (newVal, oldVal) {
  computeTransformedData(newVal);
  // code blow used to update view when data changed
  if (oldVal) {
    nextTick(
      () => {
        // reset the scrollTop for container
        // update view by handleScroll()
        vb.value.scrollTop = 0;
        handleScroll();
      }
    );
  }
}, {
  immediate: true // when not in page mode, initailize data here
})

watch(() => props.pageMode, (newVal) => {
  if (newVal) {
    window.addEventListener('scroll', handleScroll);
  } else {
    window.removeEventListener('scroll', handleScroll);
  }
  // recompute transformed data when pageMode changed
  computeTransformedData(props.data);
  nextTick(
    () => {
      // reset the scrollTop for container
      // update view by handleScroll()
      vb.value.scrollTop = 0;
      handleScroll()
    }
  );
})
watch(() => props.blockHeight, () => {
  // update view when blockHeight changed
  handleScroll();
})

onBeforeMount(() => {
  if (props.pageMode) {
    // add scroll onto window
    window.addEventListener('scroll', handleScroll);
  }
})

onMounted(() => {
  if (props.pageMode) {
    top.value = vb.value.getBoundingClientRect().top + window.scrollY
    // in page mode, initialize transformed data here
    computeTransformedData(props.data);
  }
  // initialize view by calling updateVb
  updateVb(0);
})

onUnmounted(() => {
  if (props.pageMode) {
    window.removeEventListener('scroll', handleScroll);
  }
})

function computeTransformedData(oriArr: T[]) {
  // compute accumulative height value for each block
  // note the function related to the variable 'pageMode'
  // and when fixedRowHeight is specified, transformedData is not needed
  if (!props.blockHeight && ((props.pageMode && vb.value) || !props.pageMode)) {
    let curHeight: number = props.pageMode ? vb.value.offsetTop : 0;
    let rt = [curHeight];
    oriArr.forEach(
      item => {
        curHeight += item['height'];
        rt.push(curHeight);
      }
    );
    transformedData.value = rt;
  }
}
function handleScroll() {
  // scrollTop is relative to the varible pageMode
  const scrollTop = props.pageMode ? window.scrollY : vb.value.scrollTop;
  // use requestAnimationFrame to ensure smooth scrolling visual effects
  window.requestAnimationFrame(
    () => {
      updateVb(scrollTop);
    }
  );
}

function binarySearchLowerBound(s, arr) {
  // used to search the lower bound in-viewport index for data array
  // when height is not fixed
  let lo = 0;
  let hi = arr.length - 1;
  let mid;
  while (lo <= hi) {
    // integer division
    mid = ~~((hi + lo) / 2);
    if (arr[mid] > s) {
      if (mid === 0) {
        // start position less than the smallest element in arr
        return 0;
      } else {
        hi = mid - 1;
      }
    } else if (arr[mid] < s) {
      if (mid + 1 < arr.length) {
        if (arr[mid + 1] > s) {
          return mid;
        } else {
          // normal flow
          lo = mid + 1;
        }
      } else {
        // not a valid start position
        // start position > total height
        return -1;
      }
    } else {
      // only return the matched lower bound index
      // may be modified later for smooth
      return mid;
    }
  }
}

function binarySearchUpperBound(e, arr) {
  // used to search the upper bound in-viewport index for data array
  // when height is not fixed
  let lo = 0;
  let hi = arr.length - 1;
  let mid;
  while (lo <= hi) {
    mid = ~~((hi + lo) / 2);
    if (arr[mid] > e) {
      if (mid > 0) {
        if (arr[mid - 1] < e) {
          return mid;
        } else {
          // normal flow
          hi = mid - 1;
        }
      } else {
        // not a valid end position
        // end position < view port start position
        return -1;
      }
    } else if (arr[mid] < e) {
      if (mid === arr.length - 1) {
        // end position greater than the biggest element in arr
        return arr.length - 1;
      } else {
        lo = mid + 1;
      }
    } else {
      // lower bound should return previous block
      // the slice func handles the index offset issue
      return mid;
    }
  }
}

function blockHeightLowerBound(s, blockHeight) {
  // used to compute the lower bound in-viewport index for data array
  // when in fixed height mode
  const sAdjusted = props.pageMode ? s - vb.value.offsetTop : s;
  const computedStartIndex = ~~(sAdjusted / blockHeight);
  return computedStartIndex >= 0 ? computedStartIndex : 0;
}

function blockHeightUpperBound(e, blockHeight) {
  // used to compute the upper bound in-viewport index for data array
  // when in fixed height mode
  const eAdjusted = props.pageMode ? e - vb.value.offsetTop : e;
  const compuedEndIndex = Math.ceil(eAdjusted / blockHeight);
  return compuedEndIndex <= props.data.length ? compuedEndIndex : props.data.length;
}

function findBlocksInViewport(s, e, heightArr, blockArr) {
  if (s < e) {
    const lo: number = props.blockHeight ?
      blockHeightLowerBound(s, props.blockHeight) :
      binarySearchLowerBound(s, heightArr);
    const hi = props.blockHeight ?
      blockHeightUpperBound(e, props.blockHeight) :
      binarySearchUpperBound(e, heightArr);

    var vbOffset = props.pageMode ? vb.value.offsetTop : 0;
    // set top spacer
    if (props.blockHeight) {
      offsetTop.value = lo >= 0 ? lo * props.blockHeight : 0;
    } else {
      offsetTop.value = lo >= 0 ? heightArr[lo] - vbOffset : 0;
    }
    // set bot spacer
    if (props.blockHeight) {
      offsetBot.value = hi >= 0 ? (blockArr.length - hi) * props.blockHeight : 0;
    } else {
      offsetBot.value = hi >= 0 ? heightArr[heightArr.length - 1] - heightArr[hi] : 0;
    }
    // return the sliced the data array
    return blockArr.slice(lo, hi + 2);
  } else {
    offsetTop.value = 0;
    offsetBot.value = 0;
    return [];
  }
}

function updateVb(scrollTop) {
  scrollTop -= top.value
  // compute the viewport start position and end position based on the scrollTop value
  const viewportHeight = props.pageMode ? window.innerHeight : props.height;
  viewportBegin.value = scrollTop;
  viewportEnd.value = scrollTop + viewportHeight;
  renderList.value = findBlocksInViewport(viewportBegin.value, viewportEnd.value, transformedData.value, props.data);
}

const containerStyle = computed((): StyleValue => (props.pageMode ? {} : { height: props.height + 'px', overflowY: 'scroll' }))
</script>
