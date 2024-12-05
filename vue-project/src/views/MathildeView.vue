<template>
  <div class="about">
    <h1>InfraDon2</h1>
    <p>This is Mathilde's page</p>
    <p>Counter: {{ postsData.length }}</p>
    <button type="button" role="button" @click="inc">+1</button>
    <button type="button" @click="addDocument">Ajouter un document</button>
    <button type="button" @click="putDocument">Ajouter un post</button>
    <button type="button" @click="fetchData">Récupérer les posts</button>
    <button type="button" @click="updateLocalDatabase">Synchroniser local</button>
    <button type="button" @click="updateDistantDatabase">Synchroniser distant</button>
    <div
      v-for="post in postsData"
      :key="post._id"
      style="border: 1px solid black; margin: 10px; padding: 10px"
    >
      <p v-if="post.post_name">Nom : {{ post.post_name }}</p>
      <p v-else>Nom : Non défini</p>

      <p v-if="post.post_content">Contenu : {{ post.post_content }}</p>
      <p v-else>Contenu : Non défini</p>

      <p>Date de création : {{ post.attributes.creation_date || 'Non défini' }}</p>
      <button @click="deleteData(post._id, post._rev)">Supprimer</button>
    </div>
  </div>
</template>

<script lang="ts">
import PouchDB from 'pouchdb'
import { v4 as uuidv4 } from 'uuid'

// Définition de l'interface Post
interface Post {
  _id: string
  _rev?: string
  post_name?: string
  post_content?: string
  attributes: {
    creation_date?: string
  }
}

export default {
  data() {
    return {
      postsData: [] as Post[], // Liste des posts récupérés
      localDB: null as PouchDB.Database | null,
      remoteDB: 'http://admin:admin@localhost:5984/post' // URL de la base distante
    }
  },

  methods: {
    inc() {
      console.log('Increment appelé')
    },

    initDatabase() {
      this.localDB = new PouchDB('post')
      console.log('Base locale initialisée')
      this.updateLocalDatabase()
      this.fetchData()
    },

    fetchData() {
      if (this.localDB) {
        this.localDB
          .allDocs({
            include_docs: true
          })
          .then((result: any) => {
            console.log('fetchData success =>', result.rows)
            this.postsData = result.rows.map((row: any) => ({
              _id: row.doc._id || 'Sans ID',
              _rev: row.doc._rev || '',
              post_name: row.doc.post_name || 'Nom non défini',
              post_content: row.doc.post_content || 'Contenu non défini',
              attributes: {
                creation_date: row.doc.attributes?.creation_date || 'Date non définie'
              }
            }))
          })
          .catch((error) => {
            console.error('fetchData error', error)
          })
      }
    },

    addDocument() {
      const doc = {
        _id: 'doc_' + Date.now(),
        title: 'Nouveau Document ' + new Date().toLocaleString()
      }

      if (this.localDB) {
        this.localDB
          .put(doc)
          .then((response) => {
            console.log('Document ajouté avec succès :', response)
            this.fetchData()
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du document :", error)
          })
      } else {
        console.error('Base de données locale non initialisée')
      }
    },

    putDocument() {
      const post: Post = {
        _id: uuidv4(),
        post_name: 'Post_' + new Date().toISOString(),
        post_content: "Contenu de l'article",
        attributes: {
          creation_date: new Date().toISOString()
        }
      }

      if (this.localDB) {
        this.localDB
          .put(post)
          .then(() => {
            console.log('Post ajouté avec succès')
            this.fetchData()
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du post", error)
          })
      } else {
        console.error('Base de données locale non initialisée')
      }
    },

    updateLocalDatabase() {
      if (this.localDB) {
        this.localDB.replicate
          .from(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation locale terminée')
            this.fetchData()
          })
          .on('error', (error) => {
            console.error('Erreur de synchronisation locale', error)
          })
      }
    },

    updateDistantDatabase() {
      if (this.localDB) {
        this.localDB.replicate
          .to(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation distante terminée')
          })
          .on('error', (error) => {
            console.error('Erreur de synchronisation distante', error)
          })
      }
    },

    async deleteData(id: string, rev: string) {
      console.log('Call deleteData', id, rev)
      if (!id || !rev) {
        console.error('Invalid id or rev for deletion')
        return
      }
      if (this.localDB) {
        try {
          await this.localDB.remove(id, rev)
          console.log('deleteData success')
          this.fetchData()
        } catch (error) {
          console.error('deleteData error', error)
        }
      } else {
        console.error('Database not initialized')
      }
    }
  },

  mounted() {
    this.initDatabase()
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
