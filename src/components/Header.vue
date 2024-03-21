<template>
  <header>
    <v-app-bar :elevation="6" color="primary">
        <v-app-bar-title>
          Обработка изображений
          <v-btn href="/">Загрузить</v-btn>
          <v-btn href="/resize">Изменить размер</v-btn>
        </v-app-bar-title>
    </v-app-bar>
  </header>
</template>

<script>
  export default {
    data () {
      return {
        user: "Аккаунт"
      }
    },
  }
</script>