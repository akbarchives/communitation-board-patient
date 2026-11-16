<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from './lib/supabase'

const categories = ref([])
const activeCategory = ref(null)
const isLoading = ref(true)
const isEditMode = ref(false)
const activeBanner = ref(null)
const lastSpeech = ref('')

// Pilihan palet warna kontras tinggi untuk kategori
const colorOptions = [
  '#ea580c', // Oranye
  '#0284c7', // Biru Langit
  '#dc2626', // Merah
  '#0d9488', // Tosca / Teal
  '#7c3aed', // Ungu
  '#2563eb', // Biru Royal
  '#4f46e5', // Indigo
  '#16a34a', // Hijau
  '#db2777', // Pink Tua
  '#475569'  // Abu Gelap
]

// State untuk CRUD Sub-Item
const newLabel = ref('')
const newIcon = ref('🍽️')
const newSpeech = ref('')

// State untuk Modal Tambah & Edit Kategori
const showCategoryModal = ref(false)
const isEditingCategory = ref(false)
const catForm = ref({
  id: '',
  title: '',
  icon: '📁',
  color: '#0284c7'
})

// State untuk Modal Konfirmasi Hapus Kategori (Ketik Nama)
const categoryToDelete = ref(null)
const deleteConfirmInput = ref('')

// 1. READ: Ambil semua kategori beserta sub-itemnya
async function fetchBoardData() {
  isLoading.value = true
  const { data, error } = await supabase
    .from('categories')
    .select('*, items(*)')
    .order('sort_order', { ascending: true })

  if (error) {
    console.error('Gagal mengambil data:', error.message)
  } else {
    data.forEach(cat => {
      if (cat.items) {
        cat.items.sort((a, b) => a.sort_order - b.sort_order)
      }
    })
    categories.value = data
    if (activeCategory.value) {
      activeCategory.value = data.find(c => c.id === activeCategory.value.id) || null
    }
  }
  isLoading.value = false
}

// ==========================================
// CRUD KATEGORI (ADD, EDIT, DELETE + CONFIRM)
// ==========================================

function openAddCategoryModal() {
  isEditingCategory.value = false
  catForm.value = {
    id: '',
    title: '',
    icon: '📌',
    color: '#0284c7'
  }
  showCategoryModal.value = true
}

function openEditCategoryModal(cat) {
  isEditingCategory.value = true
  catForm.value = {
    id: cat.id,
    title: cat.title,
    icon: cat.icon,
    color: cat.color || '#0284c7'
  }
  showCategoryModal.value = true
}

async function saveCategory() {
  const cleanTitle = catForm.value.title.trim()
  if (!cleanTitle) return

  if (isEditingCategory.value) {
    // UPDATE kategori yang sudah ada
    const { error } = await supabase
      .from('categories')
      .update({
        title: cleanTitle,
        icon: catForm.value.icon || '📌',
        color: catForm.value.color
      })
      .eq('id', catForm.value.id)

    if (error) {
      alert('Gagal mengubah kategori: ' + error.message)
      return
    }
  } else {
    // CREATE kategori baru (buat slug ID unik)
    const slugId =
      cleanTitle
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, '-')
        .replace(/(^-|-$)/g, '') +
      '-' +
      Date.now().toString().slice(-4)

    const { error } = await supabase.from('categories').insert([
      {
        id: slugId,
        title: cleanTitle,
        icon: catForm.value.icon || '📌',
        color: catForm.value.color,
        sort_order: categories.value.length + 1
      }
    ])

    if (error) {
      alert('Gagal menambah kategori: ' + error.message)
      return
    }
  }

  showCategoryModal.value = false
  await fetchBoardData()
}

function openDeleteCategoryModal(cat) {
  categoryToDelete.value = cat
  deleteConfirmInput.value = ''
}

async function confirmDeleteCategory() {
  if (!categoryToDelete.value) return
  // Wajib sama persis (case-insensitive) dengan nama kategori
  if (
    deleteConfirmInput.value.trim().toLowerCase() !==
    categoryToDelete.value.title.trim().toLowerCase()
  ) {
    return
  }

  const { error } = await supabase
    .from('categories')
    .delete()
    .eq('id', categoryToDelete.value.id)

  if (error) {
    alert('Gagal menghapus kategori: ' + error.message)
    return
  }

  if (activeCategory.value?.id === categoryToDelete.value.id) {
    activeCategory.value = null
  }
  categoryToDelete.value = null
  deleteConfirmInput.value = ''
  await fetchBoardData()
}

// ==========================================
// CRUD SUB-ITEM
// ==========================================

async function addItem() {
  if (!newLabel.value || !activeCategory.value) return
  const speechText = newSpeech.value.trim() || `Saya ingin ${newLabel.value}`

  const { error } = await supabase.from('items').insert([
    {
      category_id: activeCategory.value.id,
      label: newLabel.value,
      icon: newIcon.value || '🔹',
      speech: speechText,
      sort_order: (activeCategory.value.items?.length || 0) + 1
    }
  ])

  if (!error) {
    newLabel.value = ''
    newSpeech.value = ''
    await fetchBoardData()
  }
}

async function deleteItem(itemId) {
  const { error } = await supabase.from('items').delete().eq('id', itemId)
  if (!error) {
    await fetchBoardData()
  }
}

// ==========================================
// TEXT TO SPEECH & BANNER KONFIRMASI
// ==========================================

function triggerAction(label, icon, speechText, color = '#0f172a') {
  lastSpeech.value = speechText
  activeBanner.value = { label, icon, speechText, color }

  window.speechSynthesis.cancel()
  const utterance = new SpeechSynthesisUtterance(speechText)
  utterance.lang = 'id-ID'
  utterance.rate = 0.92
  window.speechSynthesis.speak(utterance)
}

function repeatLastSpeech() {
  if (!lastSpeech.value) return
  window.speechSynthesis.cancel()
  const utterance = new SpeechSynthesisUtterance(lastSpeech.value)
  utterance.lang = 'id-ID'
  utterance.rate = 0.92
  window.speechSynthesis.speak(utterance)
}

onMounted(() => {
  fetchBoardData()
})
</script>

<template>
  <div class="app-shell">
    <!-- BARIS ATAS: TOMBOL CEPAT PERMANEN -->
    <header class="quick-bar">
      <button class="q-btn q-yes" @click="triggerAction('YA', '👍', 'Ya', '#16a34a')">
        <span class="q-icon">👍</span>
        <span>YA</span>
      </button>
      <button class="q-btn q-no" @click="triggerAction('TIDAK', '👎', 'Tidak', '#dc2626')">
        <span class="q-icon">👎</span>
        <span>TIDAK</span>
      </button>
      <button class="q-btn q-help" @click="triggerAction('TOLONG', '🆘', 'Tolong bantu saya sekarang', '#d97706')">
        <span class="q-icon">🆘</span>
        <span>TOLONG</span>
      </button>
      <button class="q-btn q-repeat" @click="repeatLastSpeech" :disabled="!lastSpeech">
        <span class="q-icon">🔊</span>
        <span>ULANGI</span>
      </button>
      <button
        class="q-btn q-lock"
        :class="{ active: isEditMode }"
        @click="isEditMode = !isEditMode"
        title="Buka/Kunci Mode Edit Menu"
      >
        <span class="q-icon">{{ isEditMode ? '🔓' : '🔒' }}</span>
        <span>{{ isEditMode ? 'SELESAI' : 'EDIT' }}</span>
      </button>
    </header>

    <!-- AREA UTAMA -->
    <main class="board-area">
      <div v-if="isLoading" class="loading-box">Memuat Papan Komunikasi...</div>

      <!-- TAMPILAN 1: GRID KATEGORI UTAMA -->
      <div v-else-if="!activeCategory" class="category-grid">
        <div
          v-for="cat in categories"
          :key="cat.id"
          class="cat-card"
          @click="activeCategory = cat"
          :style="{ backgroundColor: cat.color }"
        >
          <!-- Tombol Edit & Hapus Kategori (Muncul hanya saat Mode Edit aktif) -->
          <div v-if="isEditMode" class="cat-edit-actions" @click.stop>
            <button class="cat-action-btn edit" @click="openEditCategoryModal(cat)" title="Edit Kategori">
              ✏️ Edit
            </button>
            <button class="cat-action-btn del" @click="openDeleteCategoryModal(cat)" title="Hapus Kategori">
              🗑️
            </button>
          </div>

          <span class="cat-icon">{{ cat.icon }}</span>
          <span class="cat-title">{{ cat.title }}</span>
        </div>

        <!-- KARTU TAMBAH KATEGORI BARU (Muncul saat Mode Edit aktif) -->
        <button v-if="isEditMode" class="cat-card add-cat-card" @click="openAddCategoryModal">
          <span class="cat-icon">➕</span>
          <span class="cat-title">Tambah Kategori</span>
        </button>
      </div>

      <!-- TAMPILAN 2: SUB-ITEM KATEGORI YANG DIPILIH -->
      <div v-else class="sub-view">
        <div class="sub-header">
          <button class="back-btn" @click="activeCategory = null">
            ⬅️ KEMBALI
          </button>
          <div class="sub-heading" :style="{ color: activeCategory.color }">
            <span>{{ activeCategory.icon }}</span>
            <span>{{ activeCategory.title }}</span>
          </div>

          <!-- Shortcut Edit Kategori dari dalam sub-halaman -->
          <button
            v-if="isEditMode"
            class="header-edit-cat-btn"
            @click="openEditCategoryModal(activeCategory)"
          >
            ✏️ Edit Tema
          </button>
        </div>

        <div class="items-grid">
          <div
            v-for="item in activeCategory.items"
            :key="item.id"
            class="item-card"
            :style="{ borderColor: activeCategory.color }"
            @click="triggerAction(item.label, item.icon, item.speech, activeCategory.color)"
          >
            <span class="item-icon">{{ item.icon }}</span>
            <span class="item-label">{{ item.label }}</span>

            <button
              v-if="isEditMode"
              class="delete-badge"
              @click.stop="deleteItem(item.id)"
            >
              ✕ Hapus
            </button>
          </div>
        </div>

        <!-- FORM TAMBAH SUB-ITEM -->
        <form v-if="isEditMode" @submit.prevent="addItem" class="crud-bar">
          <input v-model="newIcon" placeholder="Emoji" class="input-emoji" />
          <input v-model="newLabel" placeholder="Nama item baru (misal: Bubur Ayam)" required class="input-text" />
          <input v-model="newSpeech" placeholder="Kalimat yang diucapkan (opsional)" class="input-text" />
          <button type="submit" class="add-btn">+ Tambah Item</button>
        </form>
      </div>
    </main>

    <!-- MODAL 1: TAMBAH / EDIT KATEGORI -->
    <div v-if="showCategoryModal" class="modal-overlay" @click="showCategoryModal = false">
      <div class="modal-box" @click.stop>
        <h3>{{ isEditingCategory ? '✏️ Edit Kategori' : '➕ Tambah Kategori Baru' }}</h3>

        <form @submit.prevent="saveCategory" class="modal-form">
          <label>
            <span>Ikon / Emoji</span>
            <input v-model="catForm.icon" placeholder="Contoh: 🍽️" class="modal-input emoji-field" required />
          </label>

          <label>
            <span>Nama Kategori</span>
            <input v-model="catForm.title" placeholder="Contoh: Terapi / Fisioterapi" class="modal-input" required />
          </label>

          <div>
            <span class="field-label">Pilih Warna Kartu</span>
            <div class="color-picker">
              <button
                v-for="color in colorOptions"
                :key="color"
                type="button"
                class="color-swatch"
                :class="{ selected: catForm.color === color }"
                :style="{ backgroundColor: color }"
                @click="catForm.color = color"
              ></button>
            </div>
          </div>

          <div class="modal-actions">
            <button type="button" class="btn-cancel" @click="showCategoryModal = false">Batal</button>
            <button type="submit" class="btn-save">Simpan Kategori</button>
          </div>
        </form>
      </div>
    </div>

    <!-- MODAL 2: KONFIRMASI HAPUS KATEGORI (WAJIB KETIK NAMA KATEGORI) -->
    <div v-if="categoryToDelete" class="modal-overlay" @click="categoryToDelete = null">
      <div class="modal-box delete-modal" @click.stop>
        <h3>⚠️ Hapus Kategori "{{ categoryToDelete.title }}"?</h3>
        <p class="delete-warning">
          Tindakan ini akan menghapus kategori <strong>{{ categoryToDelete.title }}</strong> beserta
          <strong>seluruh sub-item ({{ categoryToDelete.items?.length || 0 }} item)</strong> di dalamnya secara permanen.
        </p>

        <p class="delete-instruction">
          Ketik <strong>{{ categoryToDelete.title }}</strong> di bawah ini untuk mengonfirmasi:
        </p>

        <input
          v-model="deleteConfirmInput"
          :placeholder="categoryToDelete.title"
          class="modal-input confirm-input"
          autocomplete="off"
        />

        <div class="modal-actions">
          <button type="button" class="btn-cancel" @click="categoryToDelete = null">Batal</button>
          <button
            type="button"
            class="btn-danger"
            :disabled="deleteConfirmInput.trim().toLowerCase() !== categoryToDelete.title.trim().toLowerCase()"
            @click="confirmDeleteCategory"
          >
            Hapus Permanen
          </button>
        </div>
      </div>
    </div>

    <!-- OVERLAY KONFIRMASI LAYAR PENUH SAAT ITEM DIKLIK -->
    <div v-if="activeBanner" class="confirm-overlay" @click="activeBanner = null">
      <div class="confirm-box" :style="{ borderColor: activeBanner.color }" @click.stop>
        <div class="confirm-icon">{{ activeBanner.icon }}</div>
        <div class="confirm-label" :style="{ color: activeBanner.color }">{{ activeBanner.label }}</div>
        <div class="confirm-speech">"{{ activeBanner.speechText }}"</div>
        <div class="confirm-actions">
          <button class="btn-speak-again" @click="repeatLastSpeech">🔊 Ucapkan Lagi</button>
          <button class="btn-close-confirm" @click="activeBanner = null">✖ Tutup</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  -webkit-tap-highlight-color: transparent;
}

body, html {
  width: 100%;
  height: 100%;
  background-color: #f8fafc;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  overflow-x: hidden;
}

.app-shell {
  display: flex;
  flex-direction: column;
  width: 100vw;
  min-height: 100dvh;
  padding: 10px;
  gap: 10px;
}

/* BARIS TOMBOL CEPAT ATAS */
.quick-bar {
  display: grid;
  grid-template-columns: 1.2fr 1.2fr 1.2fr 1fr 0.7fr;
  gap: 8px;
  height: 76px;
  flex-shrink: 0;
}

.q-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: none;
  border-radius: 14px;
  color: white;
  font-weight: 800;
  font-size: clamp(13px, 1.8vw, 18px);
  cursor: pointer;
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.12);
  transition: transform 0.1s;
}

.q-btn:active {
  transform: scale(0.96);
}

.q-icon {
  font-size: clamp(20px, 2.5vw, 26px);
  line-height: 1.1;
}

.q-yes { background-color: #16a34a; }
.q-no { background-color: #dc2626; }
.q-help { background-color: #ea580c; }
.q-repeat { background-color: #0284c7; }
.q-repeat:disabled { background-color: #94a3b8; opacity: 0.6; }
.q-lock { background-color: #334155; }
.q-lock.active { background-color: #eab308; color: #0f172a; }

/* AREA BOARD UTAMA */
.board-area {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.loading-box {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  font-weight: bold;
  color: #475569;
}

/* GRID KATEGORI UTAMA */
.category-grid {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-auto-rows: 1fr;
  gap: 10px;
}

@media (min-width: 768px) {
  .category-grid {
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
  }
}

.cat-card {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: none;
  border-radius: 20px;
  color: white;
  padding: 16px;
  cursor: pointer;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.12);
  transition: transform 0.1s;
  user-select: none;
}

.cat-card:active {
  transform: scale(0.97);
}

.add-cat-card {
  background-color: #e2e8f0;
  color: #334155;
  border: 4px dashed #94a3b8;
  box-shadow: none;
}

.cat-edit-actions {
  position: absolute;
  top: 10px;
  right: 10px;
  display: flex;
  gap: 6px;
  z-index: 5;
}

.cat-action-btn {
  border: none;
  border-radius: 8px;
  padding: 6px 10px;
  font-size: 13px;
  font-weight: 800;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.cat-action-btn.edit {
  background: white;
  color: #0f172a;
}

.cat-action-btn.del {
  background: #991b1b;
  color: white;
}

.cat-icon {
  font-size: clamp(46px, 7vw, 76px);
  margin-bottom: 8px;
  line-height: 1;
}

.cat-title {
  font-size: clamp(20px, 2.6vw, 30px);
  font-weight: 800;
  text-align: center;
  line-height: 1.2;
}

/* TAMPILAN SUB-ITEM */
.sub-view {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.sub-header {
  display: flex;
  align-items: center;
  gap: 14px;
  background: white;
  padding: 10px 14px;
  border-radius: 16px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.06);
}

.back-btn {
  background-color: #0f172a;
  color: white;
  border: none;
  border-radius: 12px;
  padding: 14px 22px;
  font-size: clamp(16px, 2vw, 22px);
  font-weight: 800;
  cursor: pointer;
}

.sub-heading {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: clamp(22px, 3vw, 32px);
  font-weight: 800;
  flex: 1;
}

.header-edit-cat-btn {
  background: #f1f5f9;
  color: #0f172a;
  border: 2px solid #cbd5e1;
  border-radius: 10px;
  padding: 10px 16px;
  font-weight: 800;
  cursor: pointer;
}

.items-grid {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-auto-rows: 1fr;
  gap: 10px;
}

@media (min-width: 640px) {
  .items-grid {
    grid-template-columns: repeat(3, 1fr);
    gap: 14px;
  }
}

@media (min-width: 1024px) {
  .items-grid {
    grid-template-columns: repeat(4, 1fr);
  }
}

.item-card {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: white;
  border: 4px solid #cbd5e1;
  border-radius: 20px;
  padding: 14px;
  cursor: pointer;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.06);
  transition: transform 0.1s, background-color 0.15s;
}

.item-card:active {
  transform: scale(0.96);
  background-color: #f1f5f9;
}

.item-icon {
  font-size: clamp(44px, 6.5vw, 72px);
  margin-bottom: 8px;
  line-height: 1;
}

.item-label {
  font-size: clamp(18px, 2.3vw, 26px);
  font-weight: 800;
  color: #0f172a;
  text-align: center;
  line-height: 1.2;
}

.delete-badge {
  position: absolute;
  top: 8px;
  right: 8px;
  background: #dc2626;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 6px 10px;
  font-weight: bold;
  cursor: pointer;
}

/* FORM CRUD SUB-ITEM */
.crud-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  background: #e2e8f0;
  padding: 12px;
  border-radius: 14px;
}

.input-emoji {
  width: 70px;
  padding: 12px;
  font-size: 20px;
  text-align: center;
  border-radius: 10px;
  border: 1px solid #94a3b8;
}

.input-text {
  flex: 1;
  min-width: 180px;
  padding: 12px;
  font-size: 16px;
  border-radius: 10px;
  border: 1px solid #94a3b8;
}

.add-btn {
  background: #16a34a;
  color: white;
  border: none;
  border-radius: 10px;
  padding: 12px 20px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
}

/* MODAL TAMBAH/EDIT & KONFIRMASI HAPUS */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.75);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  z-index: 60;
}

.modal-box {
  background: white;
  border-radius: 20px;
  padding: 24px;
  width: 100%;
  max-width: 440px;
  box-shadow: 0 20px 35px rgba(0, 0, 0, 0.25);
}

.modal-box h3 {
  font-size: 22px;
  margin-bottom: 16px;
  color: #0f172a;
}

.modal-form {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.modal-form label,
.field-label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-weight: 700;
  font-size: 14px;
  color: #334155;
}

.modal-input {
  width: 100%;
  padding: 12px;
  font-size: 16px;
  border: 2px solid #cbd5e1;
  border-radius: 10px;
}

.modal-input:focus {
  outline: none;
  border-color: #0284c7;
}

.emoji-field {
  font-size: 24px;
  width: 90px;
  text-align: center;
}

.color-picker {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 8px;
}

.color-swatch {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 3px solid transparent;
  cursor: pointer;
}

.color-swatch.selected {
  border-color: #0f172a;
  transform: scale(1.15);
}

.delete-warning {
  font-size: 15px;
  color: #475569;
  line-height: 1.5;
  margin-bottom: 14px;
}

.delete-instruction {
  font-size: 14px;
  color: #0f172a;
  margin-bottom: 8px;
}

.confirm-input {
  margin-bottom: 16px;
  border-color: #fca5a5;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 6px;
}

.btn-cancel, .btn-save, .btn-danger {
  padding: 12px 18px;
  border-radius: 10px;
  font-size: 15px;
  font-weight: 700;
  border: none;
  cursor: pointer;
}

.btn-cancel {
  background: #e2e8f0;
  color: #0f172a;
}

.btn-save {
  background: #0284c7;
  color: white;
}

.btn-danger {
  background: #dc2626;
  color: white;
}

.btn-danger:disabled {
  background: #fca5a5;
  cursor: not-allowed;
}

/* OVERLAY KONFIRMASI SUARA */
.confirm-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.78);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 50;
}

.confirm-box {
  background: white;
  border: 8px solid #0284c7;
  border-radius: 28px;
  padding: 32px;
  width: 100%;
  max-width: 560px;
  text-align: center;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
}

.confirm-icon {
  font-size: 100px;
  line-height: 1.1;
}

.confirm-label {
  font-size: 40px;
  font-weight: 900;
  margin-top: 8px;
}

.confirm-speech {
  font-size: 22px;
  color: #334155;
  margin-top: 8px;
  font-weight: 600;
}

.confirm-actions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 24px;
}

.btn-speak-again, .btn-close-confirm {
  padding: 16px;
  font-size: 20px;
  font-weight: 800;
  border: none;
  border-radius: 14px;
  cursor: pointer;
}

.btn-speak-again {
  background: #0284c7;
  color: white;
}

.btn-close-confirm {
  background: #e2e8f0;
  color: #0f172a;
}
</style>