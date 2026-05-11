<script setup lang="ts">
useHead({
  title: 'Chess 3D - DanielRosa.dev',
  meta: [
    { charset: 'utf-8' },
    { name: 'viewport', content: 'width=device-width, initial-scale=1' },
  ],
  link: [
    { rel: 'icon', href: '/favicon.ico' }
  ],
  htmlAttrs: {
    lang: 'es'
  }
})
</script>

<template>
  <div>
  <Background />
    <h1 style="text-align: center; color: white; padding: 20px 0;">
      ♟️ Chess 3D
    </h1>
    
    <!-- ClientOnly es importante para Three.js -->
    <ClientOnly>
      <TableroJuego />
    </ClientOnly>
  </div>
</template>

<style>
body { 
  background: #1a1a1a; 
  margin: 0; 
  font-family: system-ui, sans-serif;
}
</style>