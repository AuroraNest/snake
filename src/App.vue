<script setup lang="ts"></script>

<template>
  <main class="app">
    <header class="topbar">
      <h1>贪吃蛇 Snake</h1>
      <a class="docs" href="https://vuejs.org/" target="_blank" rel="noopener">Vue 文档</a>
    </header>
    <RouterView />
  </main>

</template>

<style scoped>
.app { max-width: 1080px; margin: 20px auto; padding: 0 16px; }
.topbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
.topbar h1 { font-size: 20px; margin: 0; }
.docs { color: #3b82f6; text-decoration: none; }
</style>
