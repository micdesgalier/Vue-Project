<template>
  <div class="ex">
    <!-- Boutons -->
    <div class="buttons">
      <button @click="getAllDocs">Get All Documents</button>
      <button @click="syncLocalToRemote">Sync Local to Remote</button>
      <button @click="syncRemoteToLocal">Sync Remote to Local</button>
      <button @click="generateMockData">Generate Mock Data</button>
    </div>

    <!-- Formulaire d'ajout -->
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

    <!-- Documents locaux -->
    <div class="documents">
      <h2>Local Documents ({{ localDocs.length }})</h2>
      <ul>
        <li v-for="doc in localDocs" :key="doc.id">
          <div>
            <strong>ID:</strong> {{ doc.id }}<br />
            <strong>Name:</strong> {{ doc.doc.name }}<br />
            <strong>Age:</strong> {{ doc.doc.age }}<br />
            <strong>Occupation:</strong> {{ doc.doc.occupation }}
          </div>
          <div>
            <button @click="startEditing(doc, 'local')">Edit</button>
            <button @click="deleteDoc(doc.id, 'local')">Delete</button>
          </div>
        </li>
      </ul>
    </div>

    <!-- Documents distants -->
    <div class="documents">
      <h2>Remote Documents ({{ remoteDocs.length }})</h2>
      <ul>
        <li v-for="doc in remoteDocs" :key="doc.id">
          <div>
            <strong>ID:</strong> {{ doc.id }}<br />
            <strong>Name:</strong> {{ doc.doc.name }}<br />
            <strong>Age:</strong> {{ doc.doc.age }}<br />
            <strong>Occupation:</strong> {{ doc.doc.occupation }}
          </div>
          <div>
            <button @click="startEditing(doc, 'remote')">Edit</button>
            <button @click="deleteDoc(doc.id, 'remote')">Delete</button>
          </div>
        </li>
      </ul>
    </div>

    <!-- Message d'erreur -->
    <div class="error" v-if="errorMessage">
      <h2>Error</h2>
      <p>{{ errorMessage }}</p>
    </div>
  </div>
</template>

<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import findPlugin from 'pouchdb-find'
PouchDB.plugin(findPlugin)

export default {
  data() {
    return {
      btnValue: 0,
      localDb: null,
      remoteDb: null,
      localDocs: [],
      remoteDocs: [],
      newDoc: { name: '', age: null, occupation: '', targetDb: 'local' },
      editingDoc: null, // État de l'édition
      errorMessage: null,
      searchQuery: '',
    };
  },
  methods: {
    initDatabases() {
      try {
        this.localDb = new PouchDB('local_worker');
        this.remoteDb = new PouchDB('http://admin:admin123@127.0.0.1:5984/worker');

        // Créer un index sur le champ "name"
        this.localDb.createIndex({ index: { fields: ['name'] } })
          .then(() => console.log('Index on "name" created'))
          .catch(err => console.error('Error creating index:', err));
      } catch (error) {
        this.errorMessage = 'Failed to initialize databases: ' + error.message;
      }
    },
    getAllDocs() {
      // Charger les documents locaux
      this.localDb
        .allDocs({ include_docs: true, descending: true })
        .then((result) => {
          this.localDocs = result.rows;
        })
        .catch((err) => {
          this.errorMessage = 'Error fetching local documents: ' + err.message;
        });

      // Charger les documents distants
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
          this.newDoc = { name: '', age: null, occupation: '', targetDb: 'local' };
          this.getAllDocs();
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
          this.getAllDocs();
        })
        .catch((err) => {
          this.errorMessage = `Error deleting document from ${targetDb}: ` + err.message;
        });
    },
    startEditing(doc, source) {
      this.editingDoc = {
        id: doc.id,
        source,
        data: { ...doc.doc },
      };
    },
    cancelEditing() {
      this.editingDoc = null;
    },
    saveEdit() {
      const targetDb = this.editingDoc.source === 'local' ? this.localDb : this.remoteDb;

      targetDb
        .get(this.editingDoc.id)
        .then((originalDoc) => {
          const updatedDoc = {
            ...originalDoc,
            ...this.editingDoc.data,
          };
          return targetDb.put(updatedDoc);
        })
        .then(() => {
          this.editingDoc = null;
          this.getAllDocs();
        })
        .catch((err) => {
          this.errorMessage = `Error saving document: ` + err.message;
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
    generateMockData() {
      console.log("IN");
      const mockData = [];
      for (let i = 0; i < 20; i++) {
        const doc = {
          _id: new Date().toISOString() + '-' + i,
          name: `Mock Name ${i + 1}`,
          age: Math.floor(Math.random() * 60) + 18, // Âge entre 18 et 77
          occupation: `Occupation ${i + 1}`,
        };
        mockData.push(doc);
      }

      this.localDb.bulkDocs(mockData)
        .then(() => {
          alert('20 mock documents added successfully!');
          this.getAllDocs();
        })
        .catch(err => {
          this.errorMessage = 'Error generating mock data: ' + err.message;
        });
    },
    searchDocuments() {
      if (this.searchQuery.trim() === '') {
        this.getAllDocs();
        return;
      }

      this.localDb.find({
        selector: { name: { $regex: new RegExp(this.searchQuery, 'i') } }
      })
      .then(result => {
        this.localDocs = result.docs.map(doc => ({ id: doc._id, doc }));
      })
      .catch(err => {
        this.errorMessage = 'Error searching documents: ' + err.message;
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
body {
  font-family: 'Arial', sans-serif;
  background-color: #f5f5f5;
  color: #333;
  margin: 0;
  padding: 0;
}

.ex {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  background: #fff;
  box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
}

.buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 20px;
}

.buttons button {
  background-color: #007bff;
  color: white;
  border: none;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.buttons button:hover {
  background-color: #0056b3;
}

.add-document,
.documents {
  margin-top: 20px;
  padding: 20px;
  background-color: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.add-document h2,
.documents h2 {
  margin-bottom: 15px;
  color: #444;
  border-bottom: 2px solid #ddd;
  padding-bottom: 5px;
}

.add-document form {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 15px;
}

.add-document label {
  display: flex;
  flex-direction: column;
  font-weight: bold;
}

.add-document button {
  grid-column: span 2;
  background-color: #28a745;
  color: white;
  border: none;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.add-document button:hover {
  background-color: #218838;
}

.documents ul {
  list-style: none;
  padding: 0;
}

.documents li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  padding: 10px;
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 5px;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.05);
}

.documents li button {
  background-color: #dc3545;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.documents li button:hover {
  background-color: #c82333;
}

.error {
  color: #721c24;
  background: #f8d7da;
  border: 1px solid #f5c6cb;
  padding: 10px;
  border-radius: 5px;
  margin-top: 20px;
}
</style>