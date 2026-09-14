<script>
export default {
  data() {
    return {
      API_BASE: "http://localhost:8080/api",
      token: localStorage.getItem("token") || "",
      isRegisterMode: false,
      form: { name: "", email: "", password: "" },
      profile: { id: null, name: "", balance: 0 },
      topupAmount: null,
      transferForm: { toUserId: null, amount: null },
      billForm: { title: "", totalAmount: null, participantIds: "" },
      isCustomSplit: false,
      customShares: "", // format: "userId:amount,userId:amount"
      billIdToCheck: null,
      currentBill: null,
      history: [],
      errorMsg: "",
      successMsg: "",
    };
  },

  mounted() {
    if (this.token) {
      this.getProfile();
    }
  },

  methods: {
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
        // Gagal ambil riwayat tidak perlu bikin seluruh halaman error,
        // cukup diamkan saja (riwayat kosong).
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

    async createBill() {
      this.clearMessages();
      try {
        let data;

        if (this.isCustomSplit) {
          // Parse format "2:150000,3:100000" jadi array of {user_id, amount}
          const participants = this.customShares
            .split(",")
            .map((pair) => {
              const [userId, amount] = pair.split(":").map((s) => parseInt(s.trim()));
              return { user_id: userId, amount: amount };
            })
            .filter((p) => !isNaN(p.user_id) && !isNaN(p.amount));

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
        this.customShares = "";
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
    <h1>💰 Dana Clone</h1>
    <p class="subtitle">Aplikasi wallet & split bill sederhana</p>

    <div v-if="!token">
      <div class="card">
        <h2>{{ isRegisterMode ? "Daftar Akun Baru" : "Login" }}</h2>

        <input v-if="isRegisterMode" v-model="form.name" placeholder="Nama" />
        <input v-model="form.email" placeholder="Email" />
        <input v-model="form.password" type="password" placeholder="Password" />

        <button @click="isRegisterMode ? register() : login()">
          {{ isRegisterMode ? "Daftar" : "Login" }}
        </button>

        <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
        <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>

        <p class="toggle-link" @click="isRegisterMode = !isRegisterMode">
          {{ isRegisterMode ? "Sudah punya akun? Login" : "Belum punya akun? Daftar" }}
        </p>
      </div>
    </div>

    <div v-else>
      <div class="profile-box">
        <div>Halo, {{ profile.name }}</div>
        <div class="balance">Rp {{ formatMoney(profile.balance) }}</div>
      </div>

      <button class="secondary" @click="logout()">Logout</button>

      <div class="card">
        <h2>Top Up Saldo</h2>
        <input v-model.number="topupAmount" type="number" placeholder="Jumlah (misal: 50000)" />
        <button @click="topUp()">Top Up</button>
        <p class="error-msg" v-if="errorMsg">{{ errorMsg }}</p>
        <p class="success-msg" v-if="successMsg">{{ successMsg }}</p>
      </div>

      <div class="card">
        <h2>Transfer</h2>
        <input v-model.number="transferForm.toUserId" type="number" placeholder="Kirim ke User ID" />
        <input v-model.number="transferForm.amount" type="number" placeholder="Jumlah" />
        <button @click="transfer()">Kirim</button>
      </div>

      <div class="card">
        <h2>Buat Split Bill</h2>
        <input v-model="billForm.title" placeholder="Judul (misal: Makan malam)" />
        <input v-model.number="billForm.totalAmount" type="number" placeholder="Total tagihan" />

        <p class="toggle-link" @click="isCustomSplit = !isCustomSplit" style="text-align: left; margin: 8px 0">
          {{ isCustomSplit ? "↩ Pakai bagi rata saja" : "⚙ Atur porsi custom per orang" }}
        </p>

        <input
          v-if="!isCustomSplit"
          v-model="billForm.participantIds"
          placeholder="Participant User ID (pisah koma: 2,3)"
        />
        <div v-else>
          <input v-model="customShares" placeholder="Format: userId:jumlah (misal: 2:150000,3:100000)" />
          <p style="font-size: 11px; color: #999; margin: 2px 0">
            Sisa dari total tagihan otomatis jadi porsi kamu sendiri.
          </p>
        </div>

        <button @click="createBill()">Buat Bill</button>
      </div>

      <div class="card">
        <h2>Cek Bill</h2>
        <input v-model.number="billIdToCheck" type="number" placeholder="Bill ID" />
        <button @click="checkBill()">Lihat Detail</button>

        <div v-if="currentBill">
          <h3 style="margin-top: 16px">{{ currentBill.bill.title }}</h3>
          <p style="color: #666; font-size: 13px">
            Total: Rp {{ formatMoney(currentBill.bill.total_amount) }} — dibuat oleh
            {{ currentBill.bill.creator.name }}
          </p>
          <div v-for="p in currentBill.participants" :key="p.id" class="participant-row">
            <div>
              <div>{{ p.user.name }}</div>
              <div style="font-size: 12px; color: #888">Rp {{ formatMoney(p.amount) }}</div>
            </div>
            <div style="display: flex; align-items: center; gap: 8px">
              <span class="badge" :class="p.paid ? 'paid' : 'unpaid'">
                {{ p.paid ? "Lunas" : "Belum bayar" }}
              </span>
              <button
                v-if="!p.paid && p.user_id === profile.id"
                style="width: auto; padding: 6px 12px; font-size: 12px; margin: 0"
                @click="settle(p.id)"
              >
                Bayar
              </button>
            </div>
          </div>
        </div>
      </div>

      <div class="card">
        <h2>Riwayat Transaksi</h2>
        <div v-if="history.length === 0" style="color: #999; font-size: 13px">
          Belum ada riwayat transaksi.
        </div>
        <div v-for="h in history" :key="h.id" class="participant-row">
          <div>
            <div style="font-size: 13px">{{ h.description }}</div>
            <div style="font-size: 11px; color: #999">{{ formatDate(h.created_at) }}</div>
          </div>
          <div
            :style="{
              fontWeight: 700,
              fontSize: '14px',
              color: h.type === 'topup' || h.type === 'transfer_in' ? '#28a745' : '#d9534f',
            }"
          >
            {{ h.type === "topup" || h.type === "transfer_in" ? "+" : "-" }}
            Rp {{ formatMoney(h.amount) }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
* {
  box-sizing: border-box;
}
body {
  font-family: -apple-system, Segoe UI, Roboto, sans-serif;
  background: #f0f2f5;
  color: #1a1a1a;
  margin: 0;
}
#app {
  max-width: 480px;
  margin: 0 auto;
  padding: 20px;
  display: block;
  text-align: left;
}
h1 {
  font-size: 22px;
  margin-bottom: 4px;
}
.subtitle {
  color: #666;
  font-size: 14px;
  margin-bottom: 20px;
}
.card {
  background: white;
  border-radius: 12px;
  padding: 18px;
  margin-bottom: 16px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
}
.card h2 {
  font-size: 15px;
  margin: 0 0 12px;
  color: #333;
}
input {
  width: 100%;
  padding: 10px 12px;
  margin: 6px 0;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
}
button {
  width: 100%;
  background: #0064d2;
  color: white;
  border: none;
  padding: 11px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  margin-top: 8px;
}
button:hover {
  background: #0050a8;
}
button.secondary {
  background: #6c757d;
}
.profile-box {
  background: linear-gradient(135deg, #0064d2, #0050a8);
  color: white;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
}
.profile-box .balance {
  font-size: 28px;
  font-weight: 700;
  margin-top: 4px;
}
.participant-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 0;
  border-bottom: 1px solid #eee;
}
.badge {
  font-size: 12px;
  padding: 3px 10px;
  border-radius: 12px;
  font-weight: 600;
}
.badge.paid {
  background: #d4edda;
  color: #155724;
}
.badge.unpaid {
  background: #fff3cd;
  color: #856404;
}
.error-msg {
  color: #d9534f;
  font-size: 13px;
  margin-top: 6px;
}
.success-msg {
  color: #28a745;
  font-size: 13px;
  margin-top: 6px;
}
.toggle-link {
  text-align: center;
  color: #0064d2;
  font-size: 13px;
  cursor: pointer;
  margin-top: 10px;
}
</style>