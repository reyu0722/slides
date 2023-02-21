<script setup lang="ts">
const props = defineProps({
  class: {
    type: String,
  },
});
</script>

<template>
  <div class="slidev-layout two-columns w-full h-full">
    <slot />
    <div class="grid grid-cols-2 gap-2">
      <div class="col-left" :class="props.class">
        <slot name="left" />
      </div>
      <div class="col-right" :class="props.class">
        <slot name="right" />
      </div>
      <div class="col-bottom" :class="props.class">
        <slot name="bottom" />
      </div>
    </div>
  </div>
</template>