<template>
  <a
    class="sidebar-element cursor-pointer"
    :id="id"
    :class="{ active: $route.path == href }"
    :target="target || '_self'"
  >
    <span class="icon" :class="[`icon-${id}`]"> </span>
    <span> {{ label }} </span>
  </a>
</template>
<script setup lang="ts">
defineProps({
  id: {
    type: String
  },
  href: {
    type: String
  },
  label: {
    type: String
  },
  icon: {
    type: String
  },
  iconHover: {
    type: String
  },
  isActive: {
    type: Boolean
  },
  mobile: {
    type: String
  },
  target: {
    type: String
  }
})
</script>
<style scoped>
[class^='icon-']:before,
[class*=' icon-']:before {
  font-family: 'nolus';
  font-style: normal;
  font-weight: normal;
  display: inline-block;
  text-decoration: inherit;
  width: unset;
  margin-right: 0;
  text-align: center;
  opacity: 1;
  font-variant: normal;
  text-transform: none;
  line-height: unset;
  margin-left: 0;
  font-size: 26px;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
