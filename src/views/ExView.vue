<template>
  <div class="ex">
    <h1>This is my page {{ btnValue }}</h1>

    <div class="buttons">
      <button @click="getAllDocs">Get All Documents</button>
      <button @click="syncLocalToRemote">Sync Local to Remote</button>
      <button @click="syncRemoteToLocal">Sync Remote to Local</button>
    </div>

    <!-- Formulaire pour ajouter un document -->
    <div class="add-document">
      <h2>Add a New Document</h2>
      <form @submit.prevent="addNewDoc">
        <label>
          Name:
          <input v-model="newDoc.name" type="text" required />
        </label>
        <label>
          Age:
          <input v-model.number="newDoc.age" type="number" required />
        </label>
        <label>
          Occupation:
          <input v-model="newDoc.occupation" type="text" required />
        </label>
        <label>
          Add to:
          <select v-model="newDoc.targetDb" required>
            <option value="local">Local Database</option>
            <option value="remote">Remote Database</option>
          </select>
        </label>
        <button type="submit">Add Document</button>
      </form>
    </div>

    <!-- Affichage des documents locaux -->
    <div class="documents">
      <h2>Local Documents ({{ localDocs.length }})</h2>
      <ul v-if="localDocs.length > 0">
        <li v-for="doc in localDocs" :key="doc.id">
          <strong>ID:</strong> {{ doc.id }} <br />
          <strong>Name:</strong> {{ doc.doc.name }} <br />
          <strong>Age:</strong> {{ doc.doc.age }} <br />
          <strong>Occupation:</strong> {{ doc.doc.occupation }} <br />
          <button @click="deleteDoc(doc.id, 'local')">Delete</button>
        </li>
      </ul>
      <p v-else>No local documents found.</p>
    </div>

    <!-- Affichage des documents distants -->
    <div class="documents">
      <h2>Remote Documents ({{ remoteDocs.length }})</h2>
      <ul v-if="remoteDocs.length > 0">
        <li v-for="doc in remoteDocs" :key="doc.id">
          <strong>ID:</strong> {{ doc.id }} <br />
          <strong>Name:</strong> {{ doc.doc.name }} <br />
          <strong>Age:</strong> {{ doc.doc.age }} <br />
          <strong>Occupation:</strong> {{ doc.doc.occupation }} <br />
          <button @click="deleteDoc(doc.id, 'remote')">Delete</button>
        </li>
      </ul>
      <p v-else>No remote documents found.</p>
    </div>

    <!-- Message d'erreur -->
    <div class="error" v-if="errorMessage">
      <h2>Error</h2>
      <p>{{ errorMessage }}</p>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      btnValue: 0,
      localDb: null,
      remoteDb: null,
      localDocs: [], // Liste des documents locaux
      remoteDocs: [], // Liste des documents distants
      newDoc: { name: '', age: null, occupation: '', targetDb: 'local' }, // Nouveau document
      errorMessage: null, // Message d'erreur
    };
  },
  methods: {
    initDatabases() {
      try {
        this.localDb = new PouchDB('local_worker');
        this.remoteDb = new PouchDB('http://admin:admin123@127.0.0.1:5984/worker');
      } catch (error) {
        this.errorMessage = 'Failed to initialize databases: ' + error.message;
      }
    },
    getAllDocs() {
      // Récupérer les documents locaux
      this.localDb
        .allDocs({ include_docs: true, descending: true })
        .then((result) => {
          this.localDocs = result.rows;
        })
        .catch((err) => {
          this.errorMessage = 'Error fetching local documents: ' + err.message;
        });

      // Récupérer les documents distants
      this.remoteDb
        .allDocs({ include_docs: true, descending: true })
        .then((result) => {
          this.remoteDocs = result.rows;
        })
        .catch((err) => {
          this.errorMessage = 'Error fetching remote documents: ' + err.message;
        });
    },
    addNewDoc() {
      if (!this.newDoc.name || !this.newDoc.age || !this.newDoc.occupation) {
        this.errorMessage = 'Please fill out all fields.';
        return;
      }

      const newDocument = {
        _id: new Date().toISOString(),
        name: this.newDoc.name,
        age: this.newDoc.age,
        occupation: this.newDoc.occupation,
      };

      const targetDb = this.newDoc.targetDb === 'local' ? this.localDb : this.remoteDb;

      targetDb
        .put(newDocument)
        .then(() => {
          this.newDoc = { name: '', age: null, occupation: '', targetDb: 'local' }; // Réinitialiser le formulaire
          this.getAllDocs(); // Rafraîchir les documents
        })
        .catch((err) => {
          this.errorMessage = 'Error adding document: ' + err.message;
        });
    },
    deleteDoc(id, targetDb) {
      const db = targetDb === 'local' ? this.localDb : this.remoteDb;

      db.get(id)
        .then((doc) => db.remove(doc))
        .then(() => {
          this.getAllDocs(); // Rafraîchir les documents
        })
        .catch((err) => {
          this.errorMessage = `Error deleting document from ${targetDb}: ` + err.message;
        });
    },
    syncLocalToRemote() {
      this.localDb
        .replicate.to(this.remoteDb)
        .on('complete', () => {
          alert('Sync from local to remote complete');
          this.getAllDocs();
        })
        .on('error', (err) => {
          this.errorMessage = 'Error syncing local to remote: ' + err.message;
        });
    },
    syncRemoteToLocal() {
      this.remoteDb
        .replicate.to(this.localDb)
        .on('complete', () => {
          alert('Sync from remote to local complete');
          this.getAllDocs();
        })
        .on('error', (err) => {
          this.errorMessage = 'Error syncing remote to local: ' + err.message;
        });
    },
  },
  mounted() {
    this.initDatabases();
    this.getAllDocs();
  },
};
</script>

<style>
.ex {
  padding: 20px;
}
.buttons {
  margin-bottom: 20px;
}
.documents,
.last-doc,
.error {
  margin-top: 20px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
}
.documents ul {
  list-style: none;
  padding: 0;
}
.documents li {
  margin-bottom: 10px;
  padding: 5px;
  border-bottom: 1px solid #ccc;
}
.error {
  color: red;
  background: #ffe6e6;
}
.documents h2 {
  margin-bottom: 10px;
}
.buttons button {
  margin-right: 10px;
  margin-bottom: 10px;
}
.add-document {
  margin-bottom: 20px;
}
.add-document form {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.add-document label {
  display: flex;
  flex-direction: column;
}
.add-document button {
  align-self: flex-start;
}
.documents li button {
  margin-top: 10px;
  background-color: #ff4d4d;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
}
.documents li button:hover {
  background-color: #ff1a1a;
}
@media (min-width: 1024px) {
  .ex {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>