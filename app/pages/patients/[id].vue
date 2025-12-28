<template>
  <div class="dashboard-container">
    <aside class="sidebar">
      <div class="logo">MediPass</div>
      <nav>
        <ul>
          <li><NuxtLink to="/" class="nav-link">Tableau de bord</NuxtLink></li>
          <li><NuxtLink to="/doctors" class="nav-link">Docteurs</NuxtLink></li>
          <li class="active"><NuxtLink to="/patients" class="nav-link">Patients</NuxtLink></li>
          <li><NuxtLink to="/laboratories" class="nav-link">Laboratoires</NuxtLink></li>
          <li><NuxtLink to="/settings" class="nav-link">Paramètres</NuxtLink></li>
        </ul>
      </nav>
      <div class="logout"><button @click="logout">Déconnexion</button></div>
    </aside>

    <main class="main-content">
      <div v-if="patient" class="patient-details">
        <header class="top-bar">
          <div class="header-left">
            <button @click="navigateTo('/patients')" class="back-btn">← Retour</button>
            <h2>Dossier Patient : {{ patient.name }}</h2>
          </div>
          <div class="user-info"><span class="badge">{{ user?.specialty || 'Généraliste' }}</span></div>
        </header>

        <div class="profile-card">
          <img :src="patient.photo_url || `https://ui-avatars.com/api/?name=${patient.name}&background=random`" class="avatar-large" />
          <div class="info">
            <h3>{{ patient.name }}</h3>
            <p>Email: {{ patient.email }}</p>
            <p>Téléphone: {{ patient.phone }}</p>
          </div>
        </div>

        <div class="actions-grid">
          <!-- Prescription Form -->
          <div class="action-card">
            <h3>💊 Nouvelle Prescription</h3>
            <form @submit.prevent="sendPrescription">
              <textarea v-model="prescriptionText" placeholder="Détails de la prescription (médicaments, posologie...)" required></textarea>
              <button type="submit" class="btn-submit" :disabled="loadingPrescription">
                {{ loadingPrescription ? 'Envoi...' : 'Envoyer Prescription' }}
              </button>
            </form>
            <p v-if="msgPrescription" class="success-msg">{{ msgPrescription }}</p>
          </div>

          <!-- Analysis Request Form -->
          <div class="action-card">
            <h3>🔬 Demande d'Analyse</h3>
            <form @submit.prevent="sendAnalysis">
              <select v-model="selectedLab" required>
                <option value="" disabled>Choisir un laboratoire</option>
                <option v-for="lab in laboratories" :key="lab.id" :value="lab.id">
                  {{ lab.name }} ({{ lab.city || 'Autre' }})
                </option>
              </select>
              <textarea v-model="analysisText" placeholder="Type d'analyse demandée..." required></textarea>
              <button type="submit" class="btn-submit" :disabled="loadingAnalysis">
                {{ loadingAnalysis ? 'Envoi...' : 'Envoyer Demande' }}
              </button>
            </form>
            <p v-if="msgAnalysis" class="success-msg">{{ msgAnalysis }}</p>
          </div>

          <!-- Chat Section -->
          <div class="action-card chat-card">
            <h3>💬 Messagerie</h3>
            <div class="chat-box" ref="chatBox">
              <div v-for="(msg, index) in messages" :key="index" :class="['message-bubble', msg.isMe ? 'me' : 'other']">
                <p>{{ msg.content }}</p>
                <span class="time">{{ msg.time }}</span>
              </div>
            </div>
            <form @submit.prevent="sendMessage" class="chat-input-form">
              <input v-model="messageInput" type="text" placeholder="Écrire un message..." />
              <button type="submit" class="btn-send">➤</button>
            </form>
          </div>
        </div>
      </div>
      <div v-else class="loading">Chargement du dossier...</div>
    </main>
  </div>
</template>

<script setup>
import { io } from 'socket.io-client'
const route = useRoute()
const user = useCookie('user')
if (!user.value) navigateTo('/login')

const patientId = route.params.id

// Fetch Data
const { data: patient } = await useFetch(`/api/patients/${patientId}`)
const { data: laboratories } = await useFetch('/api/laboratories')

// Prescription Logic
const prescriptionText = ref('')
const loadingPrescription = ref(false)
const msgPrescription = ref('')

const sendPrescription = async () => {
  loadingPrescription.value = true
  try {
    await $fetch('/api/prescriptions', {
      method: 'POST',
      body: { patientId, content: prescriptionText.value }
    })
    msgPrescription.value = 'Prescription enregistrée avec succès.'
    prescriptionText.value = ''
    setTimeout(() => msgPrescription.value = '', 3000)
  } catch (e) {
    alert('Erreur lors de l\'envoi')
  } finally {
    loadingPrescription.value = false
  }
}

// Analysis Logic
const selectedLab = ref('')
const analysisText = ref('')
const loadingAnalysis = ref(false)
const msgAnalysis = ref('')

const sendAnalysis = async () => {
  loadingAnalysis.value = true
  try {
    await $fetch('/api/analysis', {
      method: 'POST',
      body: { patientId, laboratoryId: selectedLab.value, details: analysisText.value }
    })
    msgAnalysis.value = 'Demande envoyée au laboratoire.'
    analysisText.value = ''
    selectedLab.value = ''
    setTimeout(() => msgAnalysis.value = '', 3000)
  } catch (e) {
    alert('Erreur lors de l\'envoi')
  } finally {
    loadingAnalysis.value = false
  }
}

// Chat Logic
const messages = ref([])
const messageInput = ref('')
const chatBox = ref(null)
let socket = null

onMounted(() => {
  // Connexion au serveur Socket.IO (Backend externe)
  // ATTENTION : Assurez-vous que le port correspond à votre serveur medipass-backend (ex: 3000 ou 3001)
  socket = io('http://localhost:3001') // Port du socket-server.js

  socket.on('connect', () => {
    console.log('✅ Connecté au serveur de messagerie')
    console.log('ID Socket:', socket.id)
  })

  socket.on('connect_error', (err) => {
    console.error('❌ Erreur de connexion Socket.IO:', err)
  })

  const roomName = `chat_patient${patientId}_doctor${user.value.id}`
  socket.emit('join_room', roomName)

  // Charger l'historique des messages
  fetch(`http://localhost:3000/api/messages/${roomName}`) // Port du server.js (API REST)
    .then(res => res.json())
    .then(data => {
      console.log('📜 Historique chargé:', data)
      messages.value = data.map(msg => ({
        content: msg.content,
        isMe: msg.is_doctor === 1, // C'est moi si c'est le docteur (car je suis sur le dashboard docteur)
        time: new Date(msg.created_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
      }))
      scrollToBottom()
    })
    .catch(err => console.error("Erreur chargement historique:", err))

  socket.on('receive_message', (data) => {
    console.log('📩 Message reçu:', data)
    messages.value.push({
      content: data.content,
      isMe: false,
      time: new Date(data.timestamp).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    })
    scrollToBottom()
  })
})

const sendMessage = () => {
  if (!messageInput.value.trim()) return

  const roomName = `chat_patient${patientId}_doctor${user.value.id}`
  console.log('📤 Envoi vers', roomName, ':', messageInput.value)
  
  // Envoi avec les métadonnées pour la DB
  socket.emit('send_message', { 
    room: roomName, 
    content: messageInput.value,
    senderId: user.value.id,
    isDoctor: true 
  })
  
  // Ajout local du message (car socket.to() n'envoie pas à l'émetteur)
  messages.value.push({
    content: messageInput.value,
    isMe: true,
    time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
  })
  messageInput.value = ''
  scrollToBottom()
}

const scrollToBottom = () => {
  setTimeout(() => {
    if (chatBox.value) chatBox.value.scrollTop = chatBox.value.scrollHeight
  }, 50)
}

onUnmounted(() => {
  if (socket) socket.disconnect()
})

const logout = () => { user.value = null; navigateTo('/login') }
</script>

<style scoped>
.dashboard-container { display: flex; min-height: 100vh; background-color: #020420; color: white; }
.sidebar { width: 250px; background: rgba(255, 255, 255, 0.03); border-right: 1px solid rgba(255, 255, 255, 0.1); padding: 2rem; display: flex; flex-direction: column; }
.logo { font-size: 1.5rem; font-weight: bold; color: #00DC82; margin-bottom: 3rem; }
.sidebar nav ul { list-style: none; padding: 0; }
.sidebar nav li { margin-bottom: 10px; border-radius: 8px; transition: background 0.3s; }
.sidebar nav li:hover, .sidebar nav li.active { background: rgba(0, 220, 130, 0.1); }
.nav-link { text-decoration: none; color: #aaa; display: block; padding: 10px 15px; width: 100%; height: 100%; }
.sidebar nav li.active .nav-link { color: #00DC82; }
.logout { margin-top: auto; }
.logout button { width: 100%; padding: 10px; background: none; border: 1px solid rgba(255,255,255,0.2); color: white; border-radius: 6px; cursor: pointer; }
.logout button:hover { background: rgba(255, 0, 0, 0.1); border-color: #ff6b6b; color: #ff6b6b; }
.main-content { flex: 1; padding: 2rem; overflow-y: auto; }
.top-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; }
.header-left { display: flex; align-items: center; gap: 1rem; }
.back-btn { background: none; border: none; color: #aaa; cursor: pointer; font-size: 1rem; }
.back-btn:hover { color: white; }
.badge { background: #00DC82; color: #020420; padding: 5px 10px; border-radius: 20px; font-size: 0.8rem; font-weight: bold; }

.profile-card { display: flex; align-items: center; gap: 2rem; background: rgba(255,255,255,0.05); padding: 2rem; border-radius: 12px; margin-bottom: 2rem; border: 1px solid rgba(255,255,255,0.1); }
.avatar-large { width: 80px; height: 80px; border-radius: 50%; object-fit: cover; border: 3px solid #00DC82; }
.info h3 { margin: 0 0 0.5rem 0; font-size: 1.5rem; }
.info p { margin: 0.2rem 0; color: #ccc; }

.actions-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 2rem; }
.action-card { background: rgba(255,255,255,0.05); padding: 1.5rem; border-radius: 12px; border: 1px solid rgba(255,255,255,0.1); }
.action-card h3 { color: #00DC82; margin-top: 0; }
textarea { width: 100%; height: 100px; background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.2); color: white; padding: 10px; border-radius: 8px; margin-bottom: 1rem; resize: vertical; }
select { width: 100%; background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.2); color: white; padding: 10px; border-radius: 8px; margin-bottom: 1rem; }
.btn-submit { width: 100%; padding: 10px; background: #00DC82; color: #020420; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }
.btn-submit:disabled { opacity: 0.5; cursor: not-allowed; }
.success-msg { color: #00DC82; margin-top: 1rem; text-align: center; font-size: 0.9rem; }

.chat-card { display: flex; flex-direction: column; height: 400px; }
.chat-box { flex: 1; overflow-y: auto; background: rgba(0,0,0,0.2); border-radius: 8px; padding: 1rem; margin-bottom: 1rem; display: flex; flex-direction: column; gap: 0.5rem; }
.message-bubble { max-width: 80%; padding: 8px 12px; border-radius: 12px; font-size: 0.9rem; }
.message-bubble p { margin: 0; }
.message-bubble .time { font-size: 0.7rem; opacity: 0.7; display: block; text-align: right; margin-top: 4px; }
.message-bubble.me { align-self: flex-end; background: #00DC82; color: #020420; }
.message-bubble.other { align-self: flex-start; background: rgba(255,255,255,0.1); color: white; }

.chat-input-form { display: flex; gap: 0.5rem; }
.chat-input-form input { flex: 1; padding: 10px; border-radius: 8px; border: 1px solid rgba(255,255,255,0.2); background: rgba(0,0,0,0.2); color: white; outline: none; }
.btn-send { padding: 0 15px; background: #00DC82; border: none; border-radius: 8px; cursor: pointer; font-size: 1.2rem; color: #020420; }

@media (max-width: 768px) { .actions-grid { grid-template-columns: 1fr; } }
</style>