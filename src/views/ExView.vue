<template>
  <div class="ex">
    <h1>This is my page {{ btnValue }}</h1>

    <div class="buttons">
      <button @click="getAllDocs">Get All Docs</button>
      <button v-if="lastCreatedId" @click="getOneDoc(lastCreatedId)">Get Last Created Doc</button>
      <button @click="addDemoDoc">Add Demo Doc</button>
      <button v-if="lastCreatedId" @click="updateDoc(lastCreatedId)">Update Last Created Doc</button>
      <button v-if="lastCreatedId" @click="deleteDoc(lastCreatedId)">Delete Last Created Doc</button>
      <button @click="syncLocalToRemote">Sync Local to Remote</button>
      <button @click="syncRemoteToLocal">Sync Remote to Local</button>
    </div>

    <!-- Affichage des documents -->
    <div class="documents">
      <h2>All Documents ({{ allDocs.length }})</h2>
      <ul v-if="allDocs.length > 0">
        <li v-for="doc in allDocs" :key="doc.id">
          <strong>ID:</strong> {{ doc.id }} <br />
          <strong>Name:</strong> {{ doc.doc.name }} <br />
          <strong>Age:</strong> {{ doc.doc.age }} <br />
          <strong>Occupation:</strong> {{ doc.doc.occupation }}
        </li>
      </ul>
      <p v-else>No documents found.</p>
    </div>

    <!-- Dernier document récupéré -->
    <div class="last-doc" v-if="lastDoc">
      <h2>Last Retrieved Document</h2>
      <p><strong>ID:</strong> {{ lastDoc._id }}</p>
      <p><strong>Name:</strong> {{ lastDoc.name }}</p>
      <p><strong>Age:</strong> {{ lastDoc.age }}</p>
      <p><strong>Occupation:</strong> {{ lastDoc.occupation }}</p>
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
      lastCreatedId: null,
      allDocs: [], // Liste des documents
      lastDoc: null, // Dernier document récupéré
      errorMessage: null, // Message d'erreur
      syncHandler: null,
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
      this.localDb
        .allDocs({ include_docs: true, descending: true })
        .then((result) => {
          this.allDocs = result.rows;
        })
        .catch((err) => {
          this.errorMessage = 'Error fetching documents: ' + err.message;
        });
    },
    getOneDoc(id) {
      this.localDb
        .get(id)
        .then((doc) => {
          this.lastDoc = doc;
        })
        .catch((err) => {
          this.errorMessage = 'Error fetching document: ' + err.message;
        });
    },
    addDemoDoc() {
      const demoDoc = {
        _id: new Date().toISOString(),
        name: 'Demo User',
        age: Math.floor(Math.random() * 50) + 20,
        occupation: 'Developer',
      };
      this.localDb
        .put(demoDoc)
        .then((response) => {
          this.lastCreatedId = response.id;
          this.getAllDocs(); // Rafraîchir les documents
        })
        .catch((err) => {
          this.errorMessage = 'Error adding document: ' + err.message;
        });
    },
    updateDoc(id) {
      this.localDb
        .get(id)
        .then((doc) => {
          doc.age += 1;
          return this.localDb.put(doc);
        })
        .then(() => {
          this.getAllDocs(); // Rafraîchir les documents
        })
        .catch((err) => {
          this.errorMessage = 'Error updating document: ' + err.message;
        });
    },
    deleteDoc(id) {
      this.localDb
        .get(id)
        .then((doc) => this.localDb.remove(doc))
        .then(() => {
          this.lastCreatedId = null;
          this.getAllDocs(); // Rafraîchir les documents
        })
        .catch((err) => {
          this.errorMessage = 'Error deleting document: ' + err.message;
        });
    },
    syncLocalToRemote() {
      this.localDb
        .replicate.to(this.remoteDb)
        .on('complete', () => {
          alert('Sync from local to remote complete');
        })
        .on('error', (err) => {
          this.errorMessage = 'Error syncing local to remote: ' + err.message;
        });
    },
    syncRemoteToLocal() {
      this.localDb
        .replicate.from(this.remoteDb)
        .on('complete', () => {
          alert('Sync from remote to local complete');
          this.getAllDocs(); // Rafraîchir les documents après synchronisation
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
@media (min-width: 1024px) {
  .ex {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>