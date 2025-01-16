<template>
  <div class="about">
    <h1>Gestionnaire de Posts</h1>
    <p class="description">
      Gérez vos posts et médias simplement à partir de cette interface intuitive.
    </p>
    <p class="counter">Nombre de posts: {{ postsData.length }}</p>

    <!-- Section Gestion des Posts -->
    <div class="section">
      <h2>Ajout de Posts</h2>
      <div class="buttons">
        <button class="action-button" type="button" @click="addPost">Ajouter un post</button>
      </div>
    </div>

    <!-- Section Synchronisation des Bases de Données -->
    <div class="section">
      <h2>Synchronisation des Bases</h2>
      <div class="buttons">
        <button class="action-button" type="button" @click="updateLocalDatabase">
          Synchroniser local
        </button>
        <button class="action-button" type="button" @click="updateDistantDatabase">
          Synchroniser distant
        </button>
      </div>
    </div>

    <!-- Section Gestion des Logs -->
    <div class="section">
      <h2>Gestion des Logs</h2>
      <div class="buttons">
        <!-- Bouton pour afficher ou masquer les logs -->
        <button class="action-button" type="button" @click="toggleLogs">
          {{ showLogs ? 'Masquer les logs' : 'Afficher les logs' }}
        </button>
        <button class="action-button" type="button" @click="updateLocalLogs">
          Synchroniser logs local
        </button>
        <button class="action-button" type="button" @click="updateDistantLogs">
          Synchroniser logs distant
        </button>
      </div>

      <!-- Affichage des logs -->
      <div v-if="showLogs">
        <h3>Liste des logs :</h3>
        <ul class="log-list">
          <li v-for="log in logsData" :key="log._id" class="log-item">
            <strong>{{ formatDate(log.timestamp) }}</strong> - {{ log.message }}
          </li>
        </ul>
      </div>
    </div>

    <!-- Section Recherche -->
    <div class="section">
      <h2>Recherche</h2>
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
        <br />
        <button class="action-button" type="button" @click="fetchPosts">
          Rafraîchir la liste des posts
        </button>
      </div>
    </div>

    <!-- Section Liste des Posts -->
    <div class="section">
      <h2>Liste des Posts</h2>
      <div v-for="post in postsData" :key="post._id" class="post">
        <!-- Mode Édition -->
        <div v-if="post.isEditing">
          <input
            class="edit-input"
            type="text"
            v-model="post.editingName"
            placeholder="Modifier le nom"
          />
          <textarea
            class="edit-input"
            v-model="post.editingContent"
            placeholder="Modifier le contenu"
            rows="4"
          ></textarea>
          <button class="post-action" @click="saveEdit(post)">Valider</button>
          <button class="post-action" @click="cancelEdit(post)">Annuler</button>
        </div>

        <!-- Mode Normal -->
        <div v-else>
          <p><strong>Nom : </strong>{{ post.post_name }}</p>
          <p v-if="post.post_content"><strong>Contenu : </strong>{{ post.post_content }}</p>
          <p v-else><strong>Contenu : </strong>Non défini</p>
          <button class="post-action" @click="editPost(post)">Modifier</button>
          <button class="post-action" @click="deletePost(post._id, post._rev)">Supprimer</button>
        </div>

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

    <!-- Notification utilisateur -->
    <div v-if="showNotification" class="notification">
      {{ notificationMessage }}
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
  attributes?: {
    creation_date?: string
  }
  _attachments?: { [key: string]: any }
  isEditing?: boolean
  editingName?: string
  editingContent?: string
  isUploading?: boolean
}

export default {
  data() {
    return {
      postsData: [] as Post[],
      localDB: null as PouchDB.Database | null,
      remoteDB: 'http://admin:Math200221@localhost:5984/post', // Assurez-vous que cette URL est correcte
      secondaryDB: new PouchDB('http://admin:Math200221@localhost:5984/logs'),
      searchQuery: '',
      attachmentUrls: {} as { [key: string]: string },
      uploadProgress: 0,
      showLogs: false,
      logsData: [] as Array<{ _id: string; message: string; timestamp: string }>,
      notificationMessage: '',
      showNotification: false
    }
  },

  mounted() {
    console.log('Composant monté, initialisation de localDB...')
    this.localDB = new PouchDB('post') // Initialisation de la base locale
    this.fetchPosts() // Charger les posts dès le début
    this.secondaryDB = new PouchDB('logs') // Initialiser les logs
    this.fetchLogs() // Charger les logs
  },

  methods: {
    //  Méthodes liées à l'initialisation

    initDatabase() {
      console.log('Initialisation des bases de données...')
      this.localDB = new PouchDB('post')
      console.log('Base locale initialisée :', this.localDB)

      this.updateLocalDatabase()
      this.fetchPosts()
    },

    // Méthodes utilitaires

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

    // Méthodes liées aux posts

    fetchPosts() {
      if (this.localDB) {
        this.localDB
          .allDocs<Post>({
            include_docs: true,
            attachments: true
          })
          .then(async (result) => {
            console.log('fetchPosts result:', result.rows)

            // Mise à jour des posts et des URLs des pièces jointes
            this.postsData = result.rows.map((row) => {
              const doc = row.doc as Post // Cast explicite
              const attachments = doc._attachments || {}

              // Générer des URLs pour chaque pièce jointe
              Object.keys(attachments).forEach((attachmentName) => {
                this.getAttachmentUrl(doc._id, attachmentName).then((url) => {
                  // Mettre à jour directement attachmentUrls
                  this.attachmentUrls[`${doc._id}_${attachmentName}`] = url
                })
              })

              return {
                _id: doc._id || 'Sans ID',
                _rev: doc._rev || '',
                post_name: doc.post_name || 'Nom non défini',
                post_content: doc.post_content || 'Contenu non défini',
                attributes: {
                  creation_date: doc.attributes?.creation_date || 'Date non définie'
                },
                _attachments: attachments,
                isEditing: false,
                editingName: ''
              }
            })

            console.log('postsData:', this.postsData)
          })
          .catch((error) => {
            console.error('fetchPosts error:', error)
          })
      } else {
        console.error('localDB non initialisée.')
      }
    },

    addPost() {
      console.log('addPost')
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
            this.addLog(`Post ajouté : ${post.post_name}`)
            this.fetchPosts()

            // Notification utilisateur
            this.notificationMessage = `Le post "${post.post_name}" a été ajouté avec succès !`
            this.showNotification = true

            // Masquer la notification après 3 secondes
            setTimeout(() => {
              this.showNotification = false
            }, 3000)
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du post :", error)
            this.addLog(`Erreur lors de l'ajout du post : ${error.message}`)

            // Notification d'erreur
            this.notificationMessage = `Erreur : Impossible d'ajouter le post. (${error.message})`
            this.showNotification = true

            // Masquer la notification après 3 secondes
            setTimeout(() => {
              this.showNotification = false
            }, 3000)
          })
      } else {
        console.error('Base de données locale non initialisée')

        // Notification d'erreur si la base n'est pas initialisée
        this.notificationMessage = 'Erreur : Base de données locale non initialisée.'
        this.showNotification = true

        // Masquer la notification après 3 secondes
        setTimeout(() => {
          this.showNotification = false
        }, 3000)
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
            this.addLog(`Post supprimé : ${id}`)
            this.fetchPosts()
          })
          .catch((error) => {
            console.error('Erreur lors de la suppression du post :', error)
            this.addLog(`Erreur lors de la suppression du post : ${error.message}`)
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

    editPost(post: any) {
      post.isEditing = true
      post.editingName = post.post_name // Pré-remplir avec le nom actuel
      post.editingContent = post.post_content // Pré-remplir avec le contenu actuel
    },

    saveEdit(post: any) {
      const updates = {
        post_name: post.editingName,
        post_content: post.editingContent // Inclure le contenu modifié
      }
      this.updatePost(post._id, updates) // Met à jour titre et contenu
      post.isEditing = false // Désactiver mode édition
    },

    cancelEdit(post: any) {
      post.isEditing = false // Annule l'édition
      post.editingName = '' // Réinitialise le champ temporaire pour le nom
      post.editingContent = '' // Réinitialise le champ temporaire pour le contenu
    },

    //Méthodes liées aux fichiers et pièces jointes

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

                // Trouver le post correspondant et indiquer que le chargement commence
                const post = this.postsData.find((p) => p._id === postId)
                if (post) post.isUploading = true

                this.uploadProgress = 0

                // Ajout de l'attachment
                return this.localDB
                  .putAttachment(postId, file.name, doc._rev, attachmentData, file.type)
                  .then(() => {
                    this.uploadProgress = 100 // Upload terminé

                    // Désactiver le statut de chargement
                    if (post) post.isUploading = false

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

    // Méthodes liées aux logs

    fetchLogs() {
      if (this.secondaryDB) {
        this.secondaryDB
          .allDocs({ include_docs: true })
          .then((result) => {
            this.logsData = result.rows
              .map((row) => {
                if (row.doc && '_id' in row.doc && 'message' in row.doc && 'timestamp' in row.doc) {
                  return {
                    _id: row.doc._id as string,
                    message: row.doc.message as string,
                    timestamp: row.doc.timestamp as string
                  }
                }
                return null // Ignorer les entrées non conformes
              })
              .filter(
                (log): log is { _id: string; message: string; timestamp: string } => log !== null
              )
            console.log('Logs récupérés :', this.logsData)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération des logs :', error)
          })
      } else {
        console.error('Base secondaire non initialisée.')
      }
    },

    toggleLogs() {
      this.showLogs = !this.showLogs // Basculer l'état
      if (this.showLogs) {
        this.fetchLogs() // Charger les logs uniquement lorsqu'ils sont affichés
      }
    },

    addLog(message: string) {
      if (this.secondaryDB) {
        const logEntry = {
          _id: 'log_' + Date.now(),
          message,
          timestamp: new Date().toISOString()
        }

        this.secondaryDB
          .put(logEntry)
          .then(() => {
            console.log('Log ajouté avec succès :', message)
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du log :", error)
          })
      } else {
        console.error('Base secondaire non initialisée.')
      }
    },

    // Méthodes liées à la synchronisation

    updateLocalDatabase() {
      if (this.localDB) {
        this.localDB.replicate
          .from(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation locale terminée')
            this.fetchPosts() // Récupérez les données après la synchronisation
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

    updateLocalLogs() {
      if (this.secondaryDB) {
        this.secondaryDB.replicate
          .from('http://admin:Math200221@localhost:5984/logs')
          .on('complete', () => {
            console.log('Synchronisation locale des logs terminée.')
          })
          .on('error', (error) => {
            console.error('Erreur de synchronisation locale des logs :', error)
          })
      } else {
        console.error('Base secondaire non initialisée.')
      }
    },

    updateDistantLogs() {
      if (this.secondaryDB) {
        console.log('Début de la synchronisation distante des logs...')
        this.secondaryDB.replicate
          .to('http://admin:Math200221@localhost:5984/logs')
          .on('change', (info) => {
            console.log('Changements pendant la synchronisation :', info)
          })
          .on('complete', (info) => {
            console.log('Synchronisation distante des logs terminée :', info)
          })
          .on('error', (error) => {
            console.error('Erreur de synchronisation distante des logs :', error)
          })
      } else {
        console.error('Base secondaire non initialisée.')
      }
    },

    // Méthodes liées à la recherche et aux index

    async searchPosts(query: string) {
      if (!query.trim()) {
        console.log('Veuillez saisir une requête valide pour la recherche.')
        return
      }

      if (this.localDB) {
        this.localDB
          .find({
            selector: {
              post_name: { $regex: query } // Recherche sur 'post_name'
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

    async createIndex() {
      if (this.localDB) {
        this.localDB
          .createIndex({
            index: {
              fields: ['post_name'] // Crée un index sur 'post_name'
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
    }
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
.notification {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background-color: #61dafb;
  color: #121212;
  padding: 10px 20px;
  border-radius: 5px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  font-size: 1rem;
  z-index: 1000;
  animation: fadeIn 0.3s ease-in-out;
}

.edit-input {
  width: 100%;
  padding: 8px;
  margin-bottom: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  font-size: 1rem;
}

.logs-container {
  margin-top: 20px;
  padding: 10px;
  background-color: #1e1e1e;
  border: 1px solid #444;
  border-radius: 5px;
  max-height: 300px;
  overflow-y: auto;
}

.log-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.log-item {
  padding: 8px;
  margin-bottom: 5px;
  border-bottom: 1px solid #555;
  color: #f0f0f0;
}

.log-item:last-child {
  border-bottom: none;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
