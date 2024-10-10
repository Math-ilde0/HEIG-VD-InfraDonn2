<template>
  <div class="about">
    <h1>InfraDon2</h1>
    <p>This is an about page</p>
    <p>Counter: {{ datas }}</p>
    <button type="button" role="button" @click="inc">+1</button>
  </div>
</template>

<script lang="ts">
import PouchDB from 'pouchdb' // Import avec la bonne casse

export default {
  data() {
    return {
      datas: 0, // initialize the counter with 0 or any value you prefer
      databaseReference: null as PouchDB.Database | null
    }
  },

  methods: {
    inc() {
      this.datas++ // increment the counter
    },

    initDatabase() {
      // Initialisation de PouchDB avec l'adresse IP et le port
      const db = new PouchDB('http://127.0.0.1:5986/cours3') // Assurez-vous que 'cours3' est le nom correct de la base de données
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      // Vérification en affichant dans la console
      console.log(db)

      // Enregistre la référence à la base de données
      this.databaseReference = db
    },

    fetchData() {
      // Logique pour récupérer des données depuis PouchDB
    }
  },

  mounted() {
    this.initDatabase() // called when the component is mounted
  }
}
</script>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
    flex-direction: column;
    justify-content: center;
  }

  button {
    margin-top: 20px;
    padding: 10px 20px;
    font-size: 16px;
  }
}
</style>
