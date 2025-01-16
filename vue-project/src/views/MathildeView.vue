<template>
  <div class="about">
    <h1>InfraDon2</h1>
    <p class="description">This is Mathilde's page</p>
    <p class="counter">Counter: {{ postsData.length }}</p>

    <!-- Boutons pour les fonctionnalités -->
    <div class="buttons">
      <button class="action-button" type="button" @click="addPost">Ajouter un post</button>
      <button class="action-button" type="button" @click="putPost">
        Ajouter un post spécifique
      </button>
      <button class="action-button" type="button" @click="fetchPosts">Récupérer les posts</button>
      <button class="action-button" type="button" @click="updateLocalDatabase">
        Synchroniser local
      </button>
      <button class="action-button" type="button" @click="updateDistantDatabase">
        Synchroniser distant
      </button>
      <button class="action-button" type="button" @click="createIndex">Créer un index</button>
    </div>

    <!-- Formulaire de recherche -->
    <div class="search">
      <input
        class="search-input"
        type="text"
        v-model="searchQuery"
        placeholder="Rechercher un post"
      />
      <button class="action-button" type="button" @click="searchPosts(searchQuery)">
        Rechercher
      </button>
    </div>

    <!-- Liste des posts -->
    <div v-for="post in postsData" :key="post._id" class="post">
      <!-- Gestion du champ de modification -->
      <div v-if="post.isEditing">
        <input
          class="edit-input"
          type="text"
          v-model="post.editingName"
          placeholder="Modifier le nom"
        />
        <button class="post-action" @click="saveEdit(post)">Valider</button>
        <button class="post-action" @click="cancelEdit(post)">Annuler</button>
      </div>
      <div v-else>
        <p><strong>Nom : </strong>{{ post.post_name }}</p>
        <button class="post-action" @click="editPost(post)">Modifier</button>
        <button class="post-action" @click="deletePost(post._id, post._rev)">Supprimer</button>
      </div>

      <p v-if="post.post_content"><strong>Contenu : </strong>{{ post.post_content }}</p>
      <p v-else><strong>Contenu : </strong>Non défini</p>

      <!-- Ajout de média -->
      <div>
        <label for="attachment">Ajouter un média :</label>
        <input type="file" @change="handleFileChange(post._id, $event)" />

        <!-- Barre de progression -->
        <div v-if="uploadProgress > 0">
          <p>Chargement en cours : {{ uploadProgress }}%</p>
        </div>
      </div>

      <!-- Aperçu des médias disponibles -->
      <div v-if="post._attachments && Object.keys(post._attachments).length > 0">
        <p><strong>Médias disponibles :</strong></p>
        <div v-for="(attachment, name) in post._attachments" :key="name">
          <!-- Affiche une image si c'est un fichier image -->
          <img
            v-if="attachment.content_type.startsWith('image/')"
            :src="attachmentUrls[`${post._id}_${String(name)}`]"
            :alt="String(name)"
            style="max-width: 100px; max-height: 100px"
          />
          <!-- Bouton pour télécharger -->
          <a v-else @click="getAttachment(post._id, String(name))"> Télécharger {{ name }} </a>
        </div>
      </div>

      <p>
        <strong>Date de création : </strong
        >{{ formatDate(post.attributes.creation_date) || 'Non défini' }}
      </p>
    </div>
  </div>
</template>

<script lang="ts">
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

console.log('PouchFind activé :', PouchDB.plugin === PouchFind)

import { v4 as uuidv4 } from 'uuid'

interface Post {
  _id: string
  _rev?: string
  post_name?: string
  post_content?: string
  attributes: {
    creation_date?: string
  }
  _attachments?: { [key: string]: any } // Permet de gérer les pièces jointes
  isEditing?: boolean // Pour gérer l'état de modification
  editingName?: string // Pour stocker temporairement le nouveau nom
}

export default {
  data() {
    return {
      postsData: [] as Post[],
      localDB: null as PouchDB.Database | null,
      remoteDB: 'http://admin:Math200221@localhost:5984/post',
      searchQuery: '',
      attachmentUrls: {} as { [key: string]: string }, // URLs des pièces jointes
      uploadProgress: 0 // Progression du chargement
    }
  },

  methods: {
    initDatabase() {
      console.log('Initialisation de la base de données...')
      this.localDB = new PouchDB('post')

      console.log('Base locale initialisée.')
      this.updateLocalDatabase()
      this.fetchPosts()
      this.watchChanges() // Ajout de la surveillance des changements en temps réel
    },

    formatDate(isoDate: string): string {
      const options: Intl.DateTimeFormatOptions = {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      }
      return new Date(isoDate).toLocaleString('fr-FR', options)
    },

    fetchPosts() {
      if (this.localDB) {
        this.localDB
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then(async (result: any) => {
            console.log('fetchPosts success =>', result.rows)
            this.postsData = result.rows.map((row: any) => ({
              _id: row.doc._id || 'Sans ID',
              _rev: row.doc._rev || '',
              post_name: row.doc.post_name || 'Nom non défini',
              post_content: row.doc.post_content || 'Contenu non défini',
              attributes: {
                creation_date: row.doc.attributes?.creation_date || 'Date non définie'
              },
              _attachments: row.doc._attachments || {}, // Inclure les informations des attachments
              isEditing: false,
              editingName: ''
            }))

            // Préchargement des URLs
            for (const post of this.postsData) {
              if (post._attachments) {
                for (const name in post._attachments) {
                  const url = await this.getAttachmentUrl(post._id, name)
                  this.attachmentUrls[`${post._id}_${name}`] = url
                }
              }
            }
          })
          .catch((error) => {
            console.error('fetchPosts error', error)
          })
      }
    },

    addPost() {
      const post = {
        _id: 'post_' + Date.now(),
        post_name: 'Nouveau Post ' + new Date().toLocaleString(),
        post_content: 'Contenu par défaut',
        attributes: {
          creation_date: new Date().toISOString()
        }
      }

      if (this.localDB) {
        this.localDB
          .put(post)
          .then((response) => {
            console.log('Post ajouté avec succès :', response)
            this.fetchPosts()
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du post :", error)
          })
      } else {
        console.error('Base de données locale non initialisée')
      }
    },

    putPost() {
      const post: Post = {
        _id: uuidv4(),
        post_name: 'Post_' + new Date().toISOString(),
        post_content: "Contenu de l'article",
        attributes: {
          creation_date: new Date().toISOString()
        },
        _attachments: {} // Ajoutez une propriété vide par défaut pour les pièces jointes
      }

      if (this.localDB) {
        this.localDB
          .put(post as PouchDB.Core.PutDocument<Post>) // Force le typage
          .then(() => {
            console.log('Post ajouté avec succès')
            this.fetchPosts()
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du post", error)
          })
      } else {
        console.error('Base de données locale non initialisée')
      }
    },

    updatePost(id: string, updates: Partial<Post>) {
      if (this.localDB) {
        this.localDB
          .get(id)
          .then((doc: any) => {
            const updatedDoc = {
              ...doc,
              ...updates,
              _id: doc._id,
              _rev: doc._rev
            }
            return this.localDB.put(updatedDoc)
          })
          .then(() => {
            console.log('Post mis à jour avec succès')
            this.fetchPosts()
          })
          .catch((error) => {
            console.error('Erreur lors de la mise à jour du post :', error)
          })
      } else {
        console.error('Base de données locale non initialisée.')
      }
    },

    deletePost(id: string, rev: string) {
      if (!id || !rev) {
        console.error('Invalid id or rev for deletion')
        return
      }

      if (this.localDB) {
        this.localDB
          .remove(id, rev)
          .then(() => {
            console.log('Post supprimé avec succès')
            this.fetchPosts()
          })
          .catch((error) => {
            console.error('Erreur lors de la suppression du post :', error)
          })
      }
    },

    watchChanges() {
      if (this.localDB) {
        this.localDB
          .changes({
            since: 'now',
            live: true,
            include_docs: true
          })
          .on('change', (change) => {
            console.log('Changement détecté :', change)
            this.fetchPosts()
          })
          .on('error', (error) => {
            console.error('Erreur lors de la surveillance des changements :', error)
          })
      }
    },

    createIndex() {
      if (this.localDB) {
        console.log("Création d'un index en cours...")
        this.localDB
          .createIndex({
            index: {
              fields: ['post_name'] // Crée un index sur le champ 'post_name'
            }
          })
          .then(() => {
            console.log('Index créé avec succès.')
          })
          .catch((error) => {
            console.error("Erreur lors de la création de l'index :", error)
          })
      } else {
        console.error('Base de données locale non initialisée.')
      }
    },

    searchPosts(query: string) {
      if (!query.trim()) {
        console.log('Veuillez saisir une requête valide pour la recherche.')
        return
      }

      if (this.localDB) {
        this.localDB
          .find({
            selector: {
              post_name: { $regex: query } // Recherche sur post_name avec regex
            }
          })
          .then((result: any) => {
            console.log('Résultats de la recherche :', result.docs)
            this.postsData = result.docs // Mise à jour des posts affichés
          })
          .catch((error) => {
            console.error('Erreur lors de la recherche :', error)
          })
      } else {
        console.error('Base de données locale non initialisée.')
      }
    },

    handleFileChange(postId: string, event: Event) {
      const input = event.target as HTMLInputElement // Assurez-vous que c'est un input
      if (input.files && input.files.length > 0) {
        const file = input.files[0] // Récupère le premier fichier sélectionné
        this.addAttachment(postId, file)
      } else {
        console.error('Aucun fichier sélectionné.')
      }
    },

    addAttachment(postId: string, file: File) {
      const reader = new FileReader()

      reader.onload = () => {
        if (this.localDB) {
          this.localDB
            .get(postId)
            .then((doc: any) => {
              if (reader.result instanceof ArrayBuffer) {
                const attachmentData = new Blob([reader.result], { type: file.type })
                this.uploadProgress = 0

                // Ajout de l'attachment
                return this.localDB
                  .putAttachment(postId, file.name, doc._rev, attachmentData, file.type)
                  .then(() => {
                    this.uploadProgress = 100 // Upload terminé
                    console.log('Média ajouté avec succès.')
                    this.fetchPosts() // Rafraîchir les données
                  })
              } else {
                console.error("Erreur : reader.result n'est pas un ArrayBuffer")
              }
            })
            .catch((error) => {
              console.error("Erreur lors de l'ajout de l'attachment :", error)
            })
        }
      }

      reader.onerror = (error) => {
        console.error('Erreur lors de la lecture du fichier :', error)
      }

      reader.readAsArrayBuffer(file)
    },
    getAttachment(postId: string, attachmentName: string) {
      if (this.localDB) {
        this.localDB
          .getAttachment(postId, attachmentName)
          .then((data: Blob) => {
            const url = URL.createObjectURL(data) // Génère une URL temporaire pour afficher ou télécharger le fichier
            window.open(url) // Ouvre le fichier dans une nouvelle fenêtre
            console.log('Pièce jointe téléchargée :', url)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération de la pièce jointe :', error)
          })
      } else {
        console.error('Base de données locale non initialisée.')
      }
    },

    getAttachmentUrl(postId: string, attachmentName: string): Promise<string> {
      if (this.localDB) {
        return this.localDB
          .getAttachment(postId, attachmentName)
          .then((blob: Blob) => URL.createObjectURL(blob))
      }
      return Promise.resolve('')
    },

    // Activer le mode édition
    editPost(post: any) {
      post.isEditing = true
      post.editingName = post.post_name // Pré-remplir avec le nom actuel
    },

    saveEdit(post: any) {
      const updates = { post_name: post.editingName }
      this.updatePost(post._id, updates) // Met à jour le post avec le nouveau nom
      post.isEditing = false // Désactive l'état d'édition
    },

    cancelEdit(post: any) {
      post.isEditing = false // Annule l'édition
      post.editingName = '' // Réinitialise le champ temporaire
    },

    updateLocalDatabase() {
      if (this.localDB) {
        this.localDB.replicate
          .from(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation locale terminée')
            this.fetchPosts() // Recharger les posts après synchronisation
          })
          .on('error', (error) => {
            console.error('Erreur de synchronisation locale', error)
          })
      } else {
        console.error('Base de données locale non initialisée.')
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
    }
  },

  mounted() {
    this.localDB = new PouchDB('post')
    this.fetchPosts()
  }
}
</script>

<style>
/* Import Google Fonts */
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap');

/* Global Styles */
body {
  font-family: 'Roboto', sans-serif;
  background-color: #121212;
  color: #f0f0f0;
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}

.about {
  max-width: 600px;
  margin: 20px;
  text-align: center;
}

h1 {
  font-size: 2.5rem;
  color: #61dafb;
}

.description {
  font-size: 1.2rem;
  margin: 10px 0;
  color: #cccccc;
}

.counter {
  font-size: 1.5rem;
  margin-bottom: 20px;
  color: #ffffff;
}

.buttons,
.search {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

.action-button {
  background-color: #61dafb;
  color: #121212;
  border: none;
  padding: 10px 15px;
  font-size: 1rem;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.3s;
}

.action-button:hover {
  background-color: #21a1f1;
}

.search-input {
  padding: 10px;
  border: 2px solid #61dafb;
  border-radius: 5px;
  font-size: 1rem;
  color: #121212;
}

.post {
  background-color: #1e1e1e;
  border: 1px solid #444;
  border-radius: 10px;
  padding: 15px;
  margin-bottom: 15px;
  text-align: left;
}

.post-buttons {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.post-action {
  background-color: #444;
  color: #fff;
  border: none;
  padding: 5px 10px;
  border-radius: 5px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.3s;
}

.post-action:hover {
  background-color: #555;
}
</style>
