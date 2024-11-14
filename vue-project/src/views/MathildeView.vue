<template>
  <div class="about">
    <h1>InfraDon2</h1>
    <p>This is an about page</p>
    <p>Counter: {{ datas.length }}</p>
    <button type="button" role="button" @click="inc">+1</button>
    <button type="button" @click="addDocument">Ajouter un document</button>
    <div v-for="doc in datas" :key="doc._id">
      <p>{{ doc.title }}</p>
    </div>
  </div>
</template>

<script lang="ts">
import PouchDB from 'pouchdb'

export default {
  data() {
    return {
      datas: [], // Pour stocker les documents récupérés de la base de données
      databaseReference: null as PouchDB.Database | null,
      localDB: null as PouchDB.Database | null,
      remoteDB: null as PouchDB.Database | null
    }
  },

  methods: {
    inc() {
      this.datas.length++ // increment the counter (based on documents length)
    },

    initDatabase() {
      // Initialisation des bases de données locale et distante
      this.localDB = new PouchDB('local_database')
      this.remoteDB = new PouchDB('http://127.0.0.1:5986/cours3') // Remplacez 'cours3' par le nom correct
      this.databaseReference = this.localDB

      console.log('Bases de données initialisées')

      // Lancer la synchronisation
      this.syncDatabase()
      // Surveiller les changements en temps réel
      this.watchChanges()
      // Récupérer tous les documents au démarrage
      this.fetchAllDocuments()
    },

    syncDatabase() {
      if (this.localDB && this.remoteDB) {
        this.localDB
          .sync(this.remoteDB, {
            live: true,
            retry: true
          })
          .on('change', (info) => {
            console.log('Changement détecté pendant la synchronisation:', info)
          })
          .on('paused', (err) => {
            console.log('Synchronisation en pause:', err)
          })
          .on('active', () => {
            console.log('Synchronisation reprise')
          })
          .on('denied', (err) => {
            console.error('Synchronisation refusée:', err)
          })
          .on('complete', (info) => {
            console.log('Synchronisation complète:', info)
          })
          .on('error', (err) => {
            console.error('Erreur de synchronisation:', err)
          })
      }
    },

    addDocument() {
      const doc = {
        _id: 'doc_' + Date.now(),
        title: 'Nouveau Document ' + new Date().toLocaleString()
      }
      this.localDB
        ?.put(doc)
        .then((response) => {
          console.log('Document ajouté:', response)
        })
        .catch((err) => {
          console.error("Erreur lors de l'ajout du document:", err)
        })
    },

    watchChanges() {
      this.localDB
        ?.changes({
          since: 'now',
          live: true
        })
        .on('change', (change) => {
          console.log('Modification détectée dans la base de données:', change)
          this.fetchAllDocuments() // Actualiser les documents affichés
        })
    },

    fetchAllDocuments() {
      this.localDB
        ?.allDocs({ include_docs: true })
        .then((result) => {
          console.log('Documents récupérés:', result.rows)
          this.datas = result.rows.map((row) => row.doc) // Mise à jour des données locales
        })
        .catch((err) => {
          console.error('Erreur lors de la récupération des documents:', err)
        })
    }
  },

  mounted() {
    this.initDatabase() // Initialiser la base de données lors du montage du composant
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
