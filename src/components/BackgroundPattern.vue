<template>
  <div
    class="circle"
    v-for="(_, i) in 100"
    :key="i"
    :style="circlePosition()"
  ></div>
</template>

<script>
export default {
  name: "BackgroundPattern",
  methods: {
    circlePosition() {
      const innerWidth = window.innerWidth;
      const innerHeight = window.innerHeight;
      const leftRand = Math.random() * innerWidth;
      const topRand = Math.random() * innerHeight;
      const size = Math.random() * 100;
      return {
        width: `${size}px`,
        height: `${size}px`,
        left: `${leftRand}px`,
        top: `${topRand}px`,
      };
    },
  },
};
</script>

<style></style>
