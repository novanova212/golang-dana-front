<script>
export default {
  data() {
    return {
      API_BASE: "http://localhost:8080/api",
      token: localStorage.getItem("token") || "",
      isRegisterMode: false,
      currentPage: "dashboard", // dashboard | topup | transfer | bills | history | settings
      form: { name: "", email: "", password: "" },
      profile: { id: null, name: "", balance: 0 },
      topupAmount: null,
      transferForm: { toUserId: null, amount: null },
      billForm: { title: "", totalAmount: null, participantIds: "" },
      isCustomSplit: false,
      customShares: [{ userId: null, amount: null }],
      billIdToCheck: null,
      currentBill: null,
      history: [],
      errorMsg: "",
      successMsg: "",
    };
  },

  mounted() {
    if (this.token) this.getProfile();
  },

  methods: {
    goTo(page) {
      this.clearMessages();
      this.currentPage = page;
    },

    formatMoney(amount) {
      return new Intl.NumberFormat("id-ID").format(amount || 0);
    },

    formatDate(dateStr) {
      return new Date(dateStr).toLocaleString("id-ID", {
        day: "numeric",
        month: "short",
        hour: "2-digit",
        minute: "2-digit",
      });
    },

    clearMessages() {
      this.errorMsg = "";
      this.successMsg = "";
    },

    async apiCall(path, method = "GET", body = null, useAuth = false) {
      const headers = { "Content-Type": "application/json" };
      if (useAuth && this.token) headers["Authorization"] = "Bearer " + this.token;
      const res = await fetch(this.API_BASE + path, {
        method,
        headers,
        body: body ? JSON.stringify(body) : null,
      });
      const data = await res.json();
      if (!res.ok) throw new Error(data.error || "Terjadi kesalahan");
      return data;
    },

    async register() {
      this.clearMessages();
      try {
        await this.apiCall("/register", "POST", this.form);
        this.successMsg = "Registrasi berhasil! Silakan login.";
        this.isRegisterMode = false;
      } catch (err) {
        this.errorMsg = err.message;
      }
    },

    async login() {
      this.clearMessages();
      try {
        const data = await this.apiCall("/login", "POST", {
          email: this.form.email,
          password: this.form.password,
        });
        this.token = data.token;
        localStorage.setItem("token", this.token);
        await this.getProfile();
      } catch (err) {
        this.errorMsg = err.message;
      }
    },

    logout() {
      this.token = "";
      localStorage.removeItem("token");
      this.profile = { id: null, name: "", balance: 0 };
      this.history = [];
      this.currentPage = "dashboard";
    },

    async getProfile() {
      try {
        const data = await this.apiCall("/me", "GET", null, true);
        this.profile = data.user;
        await this.getHistory();
      } catch (err) {
        this.errorMsg = err.message;
        this.logout();
      }
    },

    async getHistory() {
      try {
        const data = await this.apiCall("/wallet/history/" + this.profile.id, "GET");
        this.history = data.history || [];
      } catch (err) {
        this.history = [];
      }
    },

    async topUp() {
      this.clearMessages();
      try {
        await this.apiCall("/wallet/topup", "POST", {
          user_id: this.profile.id,
          amount: this.topupAmount,
        });
        this.successMsg = "Top up berhasil!";
        this.topupAmount = null;
        await this.getProfile();
      } catch (err) {
        this.errorMsg = err.message;
      }
    },

    async transfer() {
      this.clearMessages();
      try {
        await this.apiCall("/wallet/transfer", "POST", {
          from_user_id: this.profile.id,
          to_user_id: this.transferForm.toUserId,
          amount: this.transferForm.amount,
        });
        this.successMsg = "Transfer berhasil!";
        this.transferForm = { toUserId: null, amount: null };
        await this.getProfile();
      } catch (err) {
        this.errorMsg = err.message;
      }
    },

    addParticipantRow() {
      this.customShares.push({ userId: null, amount: null });
    },

    removeParticipantRow(index) {
      this.customShares.splice(index, 1);
      if (this.customShares.length === 0) this.customShares.push({ userId: null, amount: null });
    },

    async createBill() {
      this.clearMessages();
      try {
        let data;
        if (this.isCustomSplit) {
          const participants = this.customShares
            .filter((row) => row.userId !== null && row.amount !== null)
            .map((row) => ({ user_id: row.userId, amount: row.amount }));
          data = await this.apiCall("/bills/custom", "POST", {
            creator_id: this.profile.id,
            title: this.billForm.title,
            total_amount: this.billForm.totalAmount,
            participants: participants,
          });
        } else {
          const ids = this.billForm.participantIds
            .split(",")
            .map((s) => parseInt(s.trim()))
            .filter((n) => !isNaN(n));
          data = await this.apiCall("/bills", "POST", {
            creator_id: this.profile.id,
            title: this.billForm.title,
            total_amount: this.billForm.totalAmount,
            participant_ids: ids,
          });
        }
        this.successMsg = `Bill "${data.bill.title}" berhasil dibuat (ID: ${data.bill.id})`;
        this.billForm = { title: "", totalAmount: null, participantIds: "" };
        this.customShares = [{ userId: null, amount: null }];
      } catch (err) {
        this.errorMsg = err.message;
      }
    },

    async checkBill() {
      this.clearMessages();
      try {
        const data = await this.apiCall("/bills/" + this.billIdToCheck, "GET");
        this.currentBill = data;
      } catch (err) {
        this.errorMsg = err.message;
        this.currentBill = null;
      }
    },

    async settle(participantId) {
      this.clearMessages();
      try {
        await this.apiCall(`/bills/participants/${participantId}/settle`, "POST", {
          user_id: this.profile.id,
        });
        this.successMsg = "Pembayaran berhasil!";
        await this.getProfile();
        await this.checkBill();
      } catch (err) {
        this.errorMsg = err.message;
      }
    },
  },
};
</script>

<template>
  <div id="app">
    <!-- ============ AUTH SCREEN ============ -->
    <div v-if="!token" class="auth-wrap">
      <div class="auth-top">
        <div class="auth-icon-circle">
          <span style="font-size: 44px">{{ isRegisterMode ? "👤" : "🔒" }}</span>
          <span class="dot dot-teal"></span>
          <span class="dot dot-orange"></span>
          <span class="dot dot-red"></span>
        </div>
      </div>
      <div class="auth-sheet">
        <h2 class="auth-title">{{ isRegisterMode ? "Sign up" : "Sign in" }}</h2>
        <input v-if="isRegisterMode" v-model="form.name" placeholder="Nama" class="pill-input" />
        <input v-model="form.email" placeholder="Email" class="pill-input" />
        <input v-model="form.password" type="password" placeholder="Password" class="pill-input" />
        <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
        <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>
        <button class="pill-btn-primary" @click="isRegisterMode ? register() : login()">
          {{ isRegisterMode ? "Sign up" : "Sign in" }}
        </button>
        <p class="auth-switch">
          {{ isRegisterMode ? "Have an account?" : "Don't have an account?" }}
          <span @click="isRegisterMode = !isRegisterMode">{{ isRegisterMode ? "Sign in" : "Sign up" }}</span>
        </p>
      </div>
    </div>

    <!-- ============ MAIN APP (sudah login) ============ -->
    <div v-else class="app-shell">
      <div class="page-content">

        <!-- ===== PAGE: DASHBOARD ===== -->
        <div v-if="currentPage === 'dashboard'">
          <div class="topbar">
            <div>
              <div class="topbar-greeting">Halo, {{ profile.name }} 👋</div>
            </div>
            <div class="topbar-icon" @click="goTo('settings')">⚙️</div>
          </div>

          <div class="profile-box">
            <div style="font-size: 13px; opacity: 0.85">Saldo Kamu</div>
            <div class="balance">Rp {{ formatMoney(profile.balance) }}</div>
          </div>

          <div class="menu-grid">
            <div class="menu-item" @click="goTo('topup')">
              <div class="menu-icon" style="background: #dcfce7">⬆️</div>
              <div class="menu-label">Top Up</div>
            </div>
            <div class="menu-item" @click="goTo('transfer')">
              <div class="menu-icon" style="background: #dbeafe">💸</div>
              <div class="menu-label">Transfer</div>
            </div>
            <div class="menu-item" @click="goTo('bills')">
              <div class="menu-icon" style="background: #fef3c7">🧾</div>
              <div class="menu-label">Split Bill</div>
            </div>
            <div class="menu-item" @click="goTo('history')">
              <div class="menu-icon" style="background: #ede9fe">📜</div>
              <div class="menu-label">Riwayat</div>
            </div>
          </div>

          <div class="card">
            <h2>Transaksi Terbaru</h2>
            <div v-if="history.length === 0" style="color: #999; font-size: 13px">Belum ada riwayat.</div>
            <div v-for="h in history.slice(0, 3)" :key="h.id" class="history-row">
              <div class="history-icon" :class="h.type === 'topup' || h.type === 'transfer_in' ? 'icon-in' : 'icon-out'">
                {{ h.type === "transfer_out" ? "↑" : "↓" }}
              </div>
              <div style="flex: 1">
                <div style="font-size: 13px; font-weight: 600">{{ h.description }}</div>
                <div style="font-size: 11px; color: #999">{{ formatDate(h.created_at) }}</div>
              </div>
              <div
                :style="{ fontWeight: 700, fontSize: '14px', color: h.type === 'topup' || h.type === 'transfer_in' ? '#16A34A' : '#DC2626' }"
              >
                {{ h.type === "topup" || h.type === "transfer_in" ? "+" : "-" }}Rp {{ formatMoney(h.amount) }}
              </div>
            </div>
            <p v-if="history.length > 3" class="toggle-link" @click="goTo('history')">Lihat semua riwayat →</p>
          </div>
        </div>

        <!-- ===== PAGE: TOP UP ===== -->
        <div v-if="currentPage === 'topup'">
          <div class="page-header">
            <span class="back-arrow" @click="goTo('dashboard')">←</span>
            <span class="page-title">Top Up</span>
          </div>
          <div class="card">
            <input v-model.number="topupAmount" type="number" placeholder="Jumlah (misal: 50000)" />
            <button @click="topUp()">Top Up</button>
            <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
            <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>
          </div>
        </div>

        <!-- ===== PAGE: TRANSFER ===== -->
        <div v-if="currentPage === 'transfer'">
          <div class="page-header">
            <span class="back-arrow" @click="goTo('dashboard')">←</span>
            <span class="page-title">Transfer</span>
          </div>
          <div class="card">
            <input v-model.number="transferForm.toUserId" type="number" placeholder="Kirim ke User ID" />
            <input v-model.number="transferForm.amount" type="number" placeholder="Jumlah" />
            <button @click="transfer()">Kirim</button>
            <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
            <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>
          </div>
        </div>

        <!-- ===== PAGE: SPLIT BILL ===== -->
        <div v-if="currentPage === 'bills'">
          <div class="page-header">
            <span class="back-arrow" @click="goTo('dashboard')">←</span>
            <span class="page-title">Split Bill</span>
          </div>

          <div class="card">
            <h2>Buat Bill Baru</h2>
            <input v-model="billForm.title" placeholder="Judul (misal: Makan malam)" />
            <input v-model.number="billForm.totalAmount" type="number" placeholder="Total tagihan" />

            <div style="display: flex; justify-content: space-between; align-items: center; margin: 8px 0">
              <span class="toggle-link" @click="isCustomSplit = !isCustomSplit" style="margin: 0">
                {{ isCustomSplit ? "↩ Bagi rata" : "⚙ Custom per orang" }}
              </span>
              <button v-if="isCustomSplit" @click="addParticipantRow()" style="width: auto; padding: 6px 12px; font-size: 12px; margin: 0">
                + Tambah
              </button>
            </div>

            <input v-if="!isCustomSplit" v-model="billForm.participantIds" placeholder="Participant ID (pisah koma: 2,3)" />
            <div v-else>
              <div v-for="(row, index) in customShares" :key="index" style="display: flex; gap: 6px; align-items: center">
                <input v-model.number="row.userId" type="number" placeholder="ID" style="flex: 0 0 60px" />
                <input v-model.number="row.amount" type="number" placeholder="Jumlah tagihan" style="flex: 1" />
                <button @click="removeParticipantRow(index)" style="width: auto; padding: 8px 12px; margin: 6px 0; background: #d9534f">✕</button>
              </div>
            </div>

            <button @click="createBill()">Buat Bill</button>
            <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
            <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>
          </div>

          <div class="card">
            <h2>Cek Bill</h2>
            <input v-model.number="billIdToCheck" type="number" placeholder="Bill ID" />
            <button @click="checkBill()">Lihat Detail</button>

            <div v-if="currentBill">
              <h3 style="margin-top: 16px">{{ currentBill.bill.title }}</h3>
              <p style="color: #666; font-size: 13px">
                Total: Rp {{ formatMoney(currentBill.bill.total_amount) }} — dibuat oleh {{ currentBill.bill.creator.name }}
              </p>
              <div v-for="p in currentBill.participants" :key="p.id" class="participant-row">
                <div>
                  <div>{{ p.user.name }}</div>
                  <div style="font-size: 12px; color: #888">Rp {{ formatMoney(p.amount) }}</div>
                </div>
                <div style="display: flex; align-items: center; gap: 8px">
                  <span class="badge" :class="p.paid ? 'paid' : 'unpaid'">{{ p.paid ? "Lunas" : "Belum bayar" }}</span>
                  <button v-if="!p.paid && p.user_id === profile.id" style="width: auto; padding: 6px 12px; font-size: 12px; margin: 0" @click="settle(p.id)">
                    Bayar
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- ===== PAGE: RIWAYAT ===== -->
        <div v-if="currentPage === 'history'">
          <div class="page-header">
            <span class="back-arrow" @click="goTo('dashboard')">←</span>
            <span class="page-title">Riwayat Transaksi</span>
          </div>
          <div class="card">
            <div v-if="history.length === 0" style="color: #999; font-size: 13px">Belum ada riwayat transaksi.</div>
            <div v-for="h in history" :key="h.id" class="history-row">
              <div class="history-icon" :class="h.type === 'topup' || h.type === 'transfer_in' ? 'icon-in' : 'icon-out'">
                {{ h.type === "transfer_out" ? "↑" : "↓" }}
              </div>
              <div style="flex: 1">
                <div style="font-size: 13px; font-weight: 600">{{ h.description }}</div>
                <div style="font-size: 11px; color: #999">{{ formatDate(h.created_at) }}</div>
              </div>
              <div
                :style="{ fontWeight: 700, fontSize: '14px', color: h.type === 'topup' || h.type === 'transfer_in' ? '#16A34A' : '#DC2626' }"
              >
                {{ h.type === "topup" || h.type === "transfer_in" ? "+" : "-" }}Rp {{ formatMoney(h.amount) }}
              </div>
            </div>
          </div>
        </div>

        <!-- ===== PAGE: SETTINGS ===== -->
        <div v-if="currentPage === 'settings'">
          <div class="page-header">
            <span class="back-arrow" @click="goTo('dashboard')">←</span>
            <span class="page-title">Settings</span>
          </div>
          <div class="card" style="text-align: center">
            <div style="font-size: 40px">👤</div>
            <div style="font-weight: 700; font-size: 16px; margin-top: 6px">{{ profile.name }}</div>
            <div style="color: #999; font-size: 13px">User ID: {{ profile.id }}</div>
          </div>
          <button class="secondary" @click="logout()">Logout</button>
        </div>

      </div>

      <!-- ===== BOTTOM TAB BAR ===== -->
      <div class="tab-bar">
        <div class="tab-item" :class="{ active: currentPage === 'dashboard' }" @click="goTo('dashboard')">
          <div>🏠</div><div class="tab-label">Home</div>
        </div>
        <div class="tab-item" :class="{ active: currentPage === 'bills' }" @click="goTo('bills')">
          <div>🧾</div><div class="tab-label">Bills</div>
        </div>
        <div class="tab-item" :class="{ active: currentPage === 'history' }" @click="goTo('history')">
          <div>📜</div><div class="tab-label">Riwayat</div>
        </div>
        <div class="tab-item" :class="{ active: currentPage === 'settings' }" @click="goTo('settings')">
          <div>⚙️</div><div class="tab-label">Settings</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --primary: #4f46e5;
  --primary-dark: #3730a3;
  --bg: #f4f4fb;
}
* { box-sizing: border-box; }
body { font-family: -apple-system, "Segoe UI", Roboto, sans-serif; background: var(--bg); color: #1a1a2e; margin: 0; }
#app { max-width: 420px; margin: 0 auto; }

/* ===== Auth screen ===== */
.auth-wrap { min-height: 100vh; background: linear-gradient(160deg, #4f46e5 0%, #2c1f9e 100%); position: relative; overflow: hidden; padding-top: 48px; display: flex; flex-direction: column; }
.auth-top { display: flex; justify-content: center; padding-bottom: 60px; flex-shrink: 0; }
.auth-icon-circle { width: 130px; height: 130px; background: white; border-radius: 50%; display: flex; align-items: center; justify-content: center; position: relative; box-shadow: 0 12px 30px rgba(0,0,0,0.2); }
.dot { position: absolute; width: 16px; height: 16px; border-radius: 50%; }
.dot-teal { background: #2dd4bf; top: 10px; left: -6px; }
.dot-orange { background: #fb923c; bottom: 4px; left: 8px; }
.dot-red { background: #f87171; top: 6px; right: -8px; }
.auth-sheet { background: white; border-radius: 32px 32px 0 0; padding: 32px 28px 40px; position: relative; z-index: 1; flex: 1; display: flex; flex-direction: column; }
.auth-title { font-size: 22px; font-weight: 800; color: #1a1a2e; margin: 0 0 20px; }
.pill-input { border-radius: 999px !important; padding: 14px 20px !important; border: 1.5px solid #e5e5f0 !important; }
.pill-btn-primary { border-radius: 999px; background: var(--primary); color: white; border: none; padding: 15px; width: 100%; font-weight: 700; font-size: 15px; cursor: pointer; margin-top: 12px; }
.pill-btn-primary:hover { background: var(--primary-dark); }
.auth-switch { text-align: center; font-size: 13px; color: #8a8a9e; margin-top: 20px; }
.auth-switch span { color: var(--primary); font-weight: 700; cursor: pointer; }

/* ===== App shell (setelah login) ===== */
.app-shell { min-height: 100vh; display: flex; flex-direction: column; }
.page-content { flex: 1; padding: 20px 20px 90px; }

.topbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
.topbar-greeting { font-size: 16px; font-weight: 700; }
.topbar-icon { width: 40px; height: 40px; background: white; border-radius: 12px; display: flex; align-items: center; justify-content: center; font-size: 18px; cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }

.page-header { display: flex; align-items: center; gap: 12px; margin-bottom: 18px; }
.back-arrow { font-size: 20px; cursor: pointer; width: 36px; height: 36px; background: white; border-radius: 50%; display: flex; align-items: center; justify-content: center; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }
.page-title { font-size: 18px; font-weight: 800; }

.menu-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-bottom: 16px; }
.menu-item { text-align: center; cursor: pointer; }
.menu-icon { width: 52px; height: 52px; border-radius: 16px; display: flex; align-items: center; justify-content: center; font-size: 22px; margin: 0 auto 6px; }
.menu-label { font-size: 11px; font-weight: 600; color: #4a4a5e; }

.card { background: white; border-radius: 20px; padding: 20px; margin-bottom: 16px; box-shadow: 0 4px 16px rgba(79,70,229,0.06); border: 1px solid #f0f0f7; }
.card h2 { font-size: 15px; margin: 0 0 12px; color: #1a1a2e; font-weight: 700; }

input { width: 100%; padding: 12px 14px; margin: 6px 0; border: 1.5px solid #e5e5f0; border-radius: 12px; font-size: 14px; background: #fbfbfe; }
input:focus { outline: none; border-color: var(--primary); background: white; }
button { width: 100%; background: var(--primary); color: white; border: none; padding: 13px; border-radius: 999px; font-size: 14px; font-weight: 700; cursor: pointer; margin-top: 8px; }
button:hover { background: var(--primary-dark); }
button.secondary { background: #eeeef7; color: #4a4a5e; }

.profile-box { position: relative; background: linear-gradient(135deg, #4f46e5 0%, #1e1b6e 100%); color: white; border-radius: 24px; padding: 24px 20px; margin-bottom: 16px; overflow: hidden; box-shadow: 0 8px 24px rgba(79,70,229,0.25); }
.profile-box::before { content: ""; position: absolute; width: 160px; height: 160px; background: rgba(255,255,255,0.08); border-radius: 50%; top: -60px; right: -60px; }
.profile-box::after { content: ""; position: absolute; width: 90px; height: 90px; background: rgba(255,255,255,0.06); border-radius: 50%; bottom: -30px; right: 30px; }
.profile-box > div, .profile-box .balance { position: relative; z-index: 1; }
.profile-box .balance { font-size: 30px; font-weight: 800; margin-top: 6px; }

.participant-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid #f2f2f8; }
.history-row { display: flex; align-items: center; gap: 12px; padding: 12px 0; border-bottom: 1px solid #f2f2f8; }
.history-icon { width: 38px; height: 38px; border-radius: 12px; display: flex; align-items: center; justify-content: center; font-weight: 800; font-size: 16px; flex-shrink: 0; }
.icon-in { background: #dcfce7; color: #16a34a; }
.icon-out { background: #fee2e2; color: #dc2626; }
.badge { font-size: 11px; padding: 4px 12px; border-radius: 999px; font-weight: 700; }
.badge.paid { background: #dcfce7; color: #16a34a; }
.badge.unpaid { background: #fef3c7; color: #b45309; }
.error-msg { color: #dc2626; font-size: 13px; margin-top: 6px; }
.success-msg { color: #16a34a; font-size: 13px; margin-top: 6px; }
.toggle-link { text-align: center; color: var(--primary); font-size: 13px; font-weight: 600; cursor: pointer; margin-top: 10px; }

/* ===== Bottom Tab Bar ===== */
.tab-bar { position: fixed; bottom: 0; left: 50%; transform: translateX(-50%); width: 100%; max-width: 420px; background: white; display: flex; justify-content: space-around; padding: 10px 0 14px; box-shadow: 0 -4px 16px rgba(0,0,0,0.06); border-radius: 20px 20px 0 0; }
.tab-item { text-align: center; cursor: pointer; color: #b0b0c0; font-size: 20px; }
.tab-item.active { color: var(--primary); }
.tab-label { font-size: 10px; font-weight: 700; margin-top: 2px; }
</style>