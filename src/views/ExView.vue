<template>
  <div class="ex">
    <h1>This is my page {{ btnValue }}</h1>
    <button @click="getAllDocs">Get All Docs</button>
    <button v-if="lastCreatedId" @click="getOneDoc(lastCreatedId)">Get Last Created Doc</button>
    <button @click="addDemoDoc">Add Demo Doc</button>
    <button v-if="lastCreatedId" @click="updateDoc(lastCreatedId)">Update Last Created Doc</button>
    <button v-if="lastCreatedId" @click="deleteDoc(lastCreatedId)">Delete Last Created Doc</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      btnValue: 0,
      localDb: null,
      remoteDb: null,
      lastCreatedId: null, // Variable pour stocker l'id du dernier document créé
      syncHandler: null // Pour gérer la synchronisation en temps réel
    }
  },
  methods: {
    addToBtn: function() {
      this.btnValue += 1;
    },
    initDatabases: function() {
      try {
        // Base de données locale
        this.localDb = new PouchDB('local_worker');
        console.log('Base de données locale initialisée:', this.localDb);

        // Base de données distante
        this.remoteDb = new PouchDB('http://admin:admin123@127.0.0.1:5984/worker');
        console.log('Base de données distante initialisée:', this.remoteDb);

        // Synchronisation en temps réel entre la base locale et distante
        this.startSync();
      } catch (error) {
        console.error('Erreur lors de l\'initialisation des bases de données :', error);
      }
    },
    startSync: function() {
      // Démarrer la synchronisation entre les bases de données locale et distante
      this.syncHandler = this.localDb.sync(this.remoteDb, {
        live: true, // Synchronisation en temps réel
        retry: true // Réessayer automatiquement en cas d’échec
      }).on('change', (info) => {
        console.log('Changements détectés pendant la synchronisation:', info);
      }).on('paused', (err) => {
        console.log('Synchronisation en pause', err);
      }).on('active', () => {
        console.log('Synchronisation reprise');
      }).on('error', (err) => {
        console.error('Erreur de synchronisation:', err);
      });
    },
    getAllDocs: function() {
      this.localDb.allDocs({ include_docs: true, descending: true }, (err, doc) => {
        if (err) {
          console.error(err);
        } else {
          console.log(doc.rows);
        }
      });
    },
    getOneDoc: function(id) {
      this.localDb.get(id).then((doc) => {
        console.log(doc);
      }).catch((err) => {
        console.error(err);
      });
    },
    addDemoDoc: function() {
      const demoDoc = {
        _id: new Date().toISOString(),
        name: "Demo User",
        age: Math.floor(Math.random() * 50) + 20,
        occupation: "Developer"
      };
      this.localDb.put(demoDoc).then((response) => {
        console.log("Document ajouté:", response);
        this.lastCreatedId = response.id; // Stocker l'id du document créé
      }).catch((err) => {
        console.error("Erreur lors de l'ajout du document:", err);
      });
    },
    updateDoc: function(id) {
      this.localDb.get(id).then((doc) => {
        doc.age += 1; // Par exemple, augmenter l'âge de 1
        return this.localDb.put(doc);
      }).then((response) => {
        console.log("Document mis à jour:", response);
      }).catch((err) => {
        console.error("Erreur lors de la mise à jour du document:", err);
      });
    },
    deleteDoc: function(id) {
      this.localDb.get(id).then((doc) => {
        return this.localDb.remove(doc);
      }).then((response) => {
        console.log("Document supprimé:", response);
        this.lastCreatedId = null; // Réinitialiser l'id une fois le document supprimé
      }).catch((err) => {
        console.error("Erreur lors de la suppression du document:", err);
      });
    }
  },
  mounted() {
    this.initDatabases();
  },
  beforeDestroy() {
    // Arrêter la synchronisation lorsque le composant est détruit
    if (this.syncHandler) {
      this.syncHandler.cancel();
    }
  }
}
</script>

<style>
@media (min-width: 1024px) {
  .ex {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>